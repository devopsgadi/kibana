---
name: envoyfilter-lua-auth-triage
description: Go from "401/403 produced by an Envoy Lua / ext_authz / JWT EnvoyFilter" (found in Kibana) to root cause AND a proposed fix. Decodes Lua 401 patterns from istio-proxy logs, identifies which EnvoyFilter and which code path fired, pulls the EnvoyFilter manifest + Lua script from GitLab, matches the failing request against the script logic, and outputs a concrete remediation (config/code/MR) for a human to apply. Use when the orchestrator or envoy skill lands on 401/403 with Lua, envoy.filters.http.lua, ext_authz, jwt_authn, or a 401 with no app-container log.
---

# EnvoyFilter Lua / Auth 401 — Diagnose → Explain → Propose Fix

Prereqs: `kibana-query-strategy`, `envoy-istio-gateway-triage` Steps 1–3 done (you have NS/APP, window, a failing `x_request_id`, and the 401 confirmed at gateway or sidecar with **no app-container log for that request**).

Boundary: Kibana tells you *where* and *what*; GitLab tells you *why*; you propose the fix. You never apply it.

## Step 1 — Prove the 401 came from Envoy, not the app

```
x_request_id:"REQ" → hops sorted asc
```
| Pattern | Verdict |
|---|---|
| gateway/sidecar logs 401, `upstream_host: "-"` or `upstream_cluster: "-"`, `upstream_service_time: "-"`, `response_flags: "-"` or `UAEX` | **Envoy generated it locally** (Lua `respond()`, jwt_authn, ext_authz deny). Continue. |
| sidecar logs 401 with real `upstream_host` and `upstream_service_time > 0`, app container also logs 401 | App returned 401 (Spring Security). → `springboot-dataapi-triage` JWT signatures. Stop here. |
| `UAEX` flag | ext_authz filter denied or the authz service was unreachable (fail-closed). Jump to Step 5. |

Also aggregate over the window to size it:
```json
"aggs":{"authority":{"terms":{"field":"authority.keyword","size":10}},
        "path":{"terms":{"field":"path.keyword","size":15}},
        "method":{"terms":{"field":"method.keyword","size":5}},
        "ua":{"terms":{"field":"user_agent.keyword","size":5}},
        "has_upstream":{"filter":{"bool":{"must_not":{"term":{"upstream_host.keyword":"-"}}}}},
        "timeline":{"date_histogram":{"field":"@timestamp","fixed_interval":"1m"}}}
```
All paths vs one path, all callers vs one client, exact-minute start vs gradual — record it; it drives Step 6.

## Step 2 — Find the Lua filter's own log lines

Lua `logInfo/logWarn/logErr` and runtime errors land in the **istio-proxy container log**, not the access log. Same pod(s), same minute:

```
{container}:"istio-proxy" AND {ns}:"NS" AND ({app}:"APP" OR {app}:"istio-ingressgateway") AND
{msg}:(*lua* OR *"script log"* OR *"wasm log"* OR *"[lua]"* OR *"attempt to"* OR *"nil value"* OR *"bad argument"* OR *"stack traceback"* OR *"ext_authz"* OR *"jwt"* OR *"Jwt"* OR *"JWKS"* OR *"jwks"*)
→ terms on {pod}; samples 10, sort asc
```

Decode:

| Log text | Meaning | Class |
|---|---|---|
| `script log: ...` lines with your own strings (e.g. `token missing`, `invalid signature`, `tenant not allowed`) | Lua ran as designed and denied. Root cause is **input or policy**, not code. | POLICY |
| `attempt to index a nil value` / `attempt to concatenate a nil value` / `bad argument #1 to 'match'` | **Lua runtime error** — header missing, body nil, `headers:get()` returned nil and code didn't guard. Envoy returns 500 by default, but many scripts `pcall` and respond 401. | CODE |
| `lua filter ... script failed` / `stack traceback` | Same, with trace naming the function | CODE |
| `httpCall` timeout / `httpCall ... connect failure` / `cluster not found` | Lua called out (JWKS, authz API, token introspection) and the call failed; script's error branch returned 401 | DEPENDENCY |
| `Jwt is missing` / `Jwt verification fails` / `Jwks remote fetch is failed` / `Jwt issuer is not configured` / `Audiences in Jwt are not allowed` | istio `jwt_authn` (RequestAuthentication), not Lua | JWT |
| `ext_authz ... connection failure` / `ext_authz filter rejected the request` / gRPC `UNAVAILABLE` | ext_authz (AuthorizationPolicy CUSTOM) | EXT_AUTHZ |
| nothing at all in container log, only access-log 401 | Lua `respond()` without logging, or `failure_mode_allow: false` path. Go to Step 3 to read the script. | SILENT |

## Step 3 — Identify the filter and pull its source from GitLab

You know: gateway vs sidecar (from which pod logged the 401), namespace, app, and the authority/path. Now find the EnvoyFilter.

**3a. From istiod logs (which filters apply to this workload):**
```
{ns}:"istio-system" AND {app}:"istiod" AND {msg}:(*EnvoyFilter* AND ("NS" OR "APP" OR "istio-ingressgateway"))
```
Names appear in `Push` / `applied` / `rejected` lines. Also catch bad filters: `*EnvoyFilter* AND (*rejected* OR *invalid* OR *"failed to apply"*)`.

**3b. From GitLab (your GitLab MCP):**
Search order:
1. The app's Helm chart / manifests repo (from service catalog): files matching `envoyfilter*.yaml`, `*lua*.yaml`, `templates/istio/*`
2. Platform/mesh repo: `istio/envoyfilters/`, `gateway/`, `mesh-config/`
3. Repo-wide code search: `kind: EnvoyFilter` + `envoy.filters.http.lua` + (`APP` OR `istio-ingressgateway` OR the `workloadSelector` label)
4. Also fetch: `RequestAuthentication`, `AuthorizationPolicy` for NS/APP (for JWT / ext_authz classes)

Fetch the **file at the commit deployed at INCIDENT_TIME**: list commits on that path, pick the last one before the first-error minute (Step 1 timeline). Also fetch the previous commit. **A commit on the filter path within 24h before first-error is the most likely root cause.**

If GitLab MCP is unavailable: ask the human for the EnvoyFilter name/YAML and stop — do not guess the script contents.

## Step 4 — Read the script against the failing request

Walk the Lua `envoy_on_request` (or `envoy_on_response`) with the real request from Step 1 samples: method, path, authority, headers you can see (`user_agent`, `x_forwarded_for`, `requested_server_name`), and whether `Authorization` existed (`has_upstream` agg + any `script log` line).

Checklist — the common root causes, in order of frequency:

| # | Check in script | Typical failure |
|---|---|---|
| 1 | Header name case / exact string: `headers:get("authorization")` vs `"Authorization"` | Envoy lowercases; `"Authorization"` returns nil → 401 for everyone |
| 2 | `Bearer ` prefix parsing: `string.sub(h, 8)` / `match("Bearer%s+(.+)")` | Client sends `bearer` lowercase or extra space; token nil |
| 3 | Path/authority allowlist: `if path:match("^/actuator") then return end` | New endpoint or new host (canary/new env) not in allowlist → 401 on just that path/host — matches Step 1 "one path" shape |
| 4 | `httpCall` to JWKS/introspection: cluster name, timeout, `headers[":authority"]` | Cluster renamed, cert rotated, timeout too low under load → DEPENDENCY class; matches `httpCall` errors in Step 2 |
| 5 | JWKS caching / `kid` lookup: `jwks[kid]` nil after IdP key rotation | Exact-minute start, all callers, no deploy → **IdP rotated keys**, script caches forever or JWKS fetch failing |
| 6 | `exp`/`nbf` clock checks with `os.time()` | Clock skew, or token lifetime shortened by IdP |
| 7 | `aud`/`iss`/`tenant`/`scope` string compare | IdP changed issuer URL (http→https, trailing slash), new client-id not allowlisted |
| 8 | `respond({[":status"]="401"}, ...)` inside `pcall` error branch | CODE class: any nil error becomes 401 — look for the nil source in Step 2 trace |
| 9 | `failure_mode_allow` / fail-open vs fail-closed on `httpCall` failure | Design choice; outage of authz dependency becomes global 401 |
| 10 | Filter `priority` / ordering vs `jwt_authn` | Lua runs before JWT filter populated metadata → reads nil |
| 11 | `workloadSelector` / `match.context` (`GATEWAY` vs `SIDECAR_INBOUND`) | Filter accidentally applied to a new workload after label change |
| 12 | Diff old→new commit | Anything above introduced in the deploy before first-error |

Match the Step 1 shape:
- **all paths, all callers, exact minute, no commit** → #4/#5/#7 (dependency or IdP change)
- **all paths, all callers, starts at a commit/deploy** → #1/#2/#8/#10/#11
- **one path or one host** → #3
- **one client/user-agent** → #2/#7 (client-side token format or new client-id)
- **intermittent** → #4 timeout under load, or #9 with flaky dependency

## Step 5 — ext_authz / jwt_authn variants (not Lua)

- **ext_authz** (`UAEX`): query the authz service's own logs (from catalog: `{app}:opa` / `{app}:authz-service`) same window: 5xx, timeouts, policy-decision logs (`decision: deny`, `allowed: false`, reason). Pull `AuthorizationPolicy` (action CUSTOM, provider name) and meshConfig `extensionProviders` from GitLab. Common fixes: provider timeout, `failOpen`, policy bundle rollback, authz service scale.
- **jwt_authn**: pull `RequestAuthentication` from GitLab; compare `issuer`, `jwksUri`, `audiences` with the token claims the client says it sends. `Jwks remote fetch is failed` → JWKS host via `egress-triage` (ServiceEntry/TLS). `Jwt issuer is not configured` → issuer string mismatch (trailing slash, http vs https, new tenant).

## Step 6 — Produce the fix proposal

You must output **a specific change**, with the evidence line that justifies it. Never "check the EnvoyFilter". Format:

```
ROOT CAUSE
  <class: POLICY|CODE|DEPENDENCY|JWT|EXT_AUTHZ>  <one sentence naming the exact line/condition>
  Filter: <EnvoyFilter name> ns=<ns> context=<GATEWAY|SIDECAR_INBOUND> repo=<path@sha>
  Trigger: <commit <sha> by <author> at <T> | IdP key rotation at <T> | dependency <host> failing since <T> | new path /x not allowlisted>

EVIDENCE
  1. Kibana: <query> → <ts | pod | req-id | 401 upstream_host:-> 
  2. Kibana istio-proxy: <script log / lua error line>
  3. GitLab: <file:line> — <the offending snippet, ≤5 lines>
  4. Timeline: first 401 <T>; commit/rotation <T-x>

PROPOSED FIX  (for human review — not applied)
  Change: <file path> 
    - <old line>
    + <new line>
  Or config: <RequestAuthentication/AuthorizationPolicy field → value>
  Why this fixes it: <one sentence tying to the evidence>
  Risk: <low|medium|high> — <who else uses this filter (workloadSelector scope)>
  Rollback alternative: revert <sha> (restores behavior as of <T>)
  Validation after fix: Kibana query <same filter> expect 401 rate → 0 within 2 min of istiod push; confirm with x_request_id trace showing upstream_host populated.

IF UNRESOLVED
  Need from human: <EnvoyFilter YAML | sample failing token (redacted claims) | authz service logs not shipped>
```

### Fix templates by class

- **CODE / header case**: `local auth = headers:get("authorization")` (lowercase) + nil guard `if not auth then respond 401 "missing authorization" end` with `logWarn` so future triage isn't SILENT.
- **CODE / nil crash**: wrap parse in `pcall`, return 401 with explicit log, never let runtime error become a silent 500/401.
- **POLICY / allowlist**: add path/host to the allowlist table; recommend moving allowlist to a ConfigMap-fed `inlineCode` template so canary/new envs don't need code edits.
- **DEPENDENCY / httpCall**: raise `timeout_ms` only if p99 of the authz dependency justifies it; otherwise fix the dependency (its own triage). Add `cluster` existence check (`NC` in sidecar logs).
- **JWT / key rotation**: JWKS fetch with TTL (`jwks_duration`/refresh in `RequestAuthentication`), or Lua cache keyed by `kid` with miss → refetch. Immediate relief: istiod restart is NOT yours to run — propose it as a human action.
- **JWT / issuer mismatch**: align `issuer` string exactly with token `iss`; list both in `RequestAuthentication.jwtRules` during migration.
- **EXT_AUTHZ unavailable**: scale/heal the authz service (own triage), set `timeout`, decide fail-open explicitly, never by accident.

## Guardrails

- Quote the script line — don't paraphrase Lua from memory.
- If the deployed commit cannot be determined, say so and present fixes for both candidate versions.
- A proposed diff is a recommendation. The report must say "not applied" and name who owns the repo (from catalog/CODEOWNERS).
