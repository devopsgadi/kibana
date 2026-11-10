---
name: service-to-service-auth-triage
description: Caller app A gets 401/403 (or a 503 that looks like auth) calling app B inside the mesh. Locates which of the three parties produced the status — A's own sidecar, B's sidecar (Istio AuthorizationPolicy / RequestAuthentication / PeerAuthentication mTLS) or B's Spring Security (scope, audience, issuer, expired client-credentials token) — scopes it (only A? all callers? which paths?), pins the onset to a policy push, deploy, ServiceAccount or secret change, pulls the policies and configs from GitLab, and proposes the exact fix. Use for "forbidden", "unauthorized", "access denied", "RBAC", "insufficient scope", "invalid token" between internal services.
---

# Service-to-service 401/403 triage (A → B)

Prereq: `kibana-query-strategy`, intake done (A, B, T). Catalog: `A.ns, A.sa, B.ns, B.svc, B.ports, auth model (mTLS only | JWT between services | OAuth client-credentials | API key)`. If B is reached through Apigee, run `request-origin-trace` funnel instead — Apigee owns the 401 in that case.

```
A_OUT  = {ns}:A_NS AND {app}:A AND {container}:istio-proxy AND upstream_cluster:outbound|*|B_SVC.B_NS.svc.cluster.local*
B_IN   = {ns}:B_NS AND {app}:B AND {container}:istio-proxy AND upstream_cluster:inbound*
B_APP  = {ns}:B_NS AND {app}:B AND NOT {container}:istio-proxy
A_APP  = {ns}:A_NS AND {app}:A AND NOT {container}:istio-proxy
```

## Step 1 — Which party answered? (3 calls, W0)

**1a A outbound**
```json
{"size":0,"query":{"bool":{"filter":[RANGE,A_OUT,{"terms":{"response_code":[401,403,503]}}]}},
 "aggs":{"code_flag":{"multi_terms":{"terms":[{"field":"response_code"},{"field":"response_flags"}],"size":10}},
  "no_upstream":{"filter":{"term":{"upstream_host.keyword":"-"}}},
  "tls":{"terms":{"field":"upstream_transport_failure_reason.keyword","size":3}},
  "path":{"terms":{"field":"path.keyword","size":10}},"method":{"terms":{"field":"method.keyword","size":5}},
  "first":{"min":{"field":"@timestamp"}},
  "samples":{"top_hits":{"size":5,"_source":["@timestamp","x_request_id","response_code","response_flags","upstream_host","upstream_service_time","path","method"]}}}}
```
**1b B inbound** — same aggs on `B_IN AND response_code:(401 OR 403)`, plus `source_principal` / `downstream_peer_principal` / `x_forwarded_client_cert` terms (whichever exists) and `downstream_remote_address`.
**1c Trace 3 request ids** from 1a samples across A sidecar → B sidecar → B app (cross-skill request-id query). Also B sidecar **container** log for those minutes:
```
B_IN-pods AND {msg}:(*"RBAC: access denied"* OR *"Jwt is missing"* OR *"Jwt verification fails"* OR *"Jwks"* OR *"Jwt issuer is not configured"* OR *"Audiences in Jwt are not allowed"* OR *"ext_authz"* OR *"TLS error"*)
```
And B app:
```
B_APP AND (http.response.status_code:(401 OR 403) OR {msg}:(*AccessDeniedException* OR *"insufficient_scope"* OR *JwtValidationException* OR *"invalid_token"* OR *"Jwt expired"* OR *"iss claim"* OR *"aud claim"* OR *"Access Denied"* OR *PreAuthorize* OR *"Full authentication is required"*))
```

## Step 2 — Decision table (the verdict)

| A outbound | B inbound sidecar | B app log | Verdict | Class |
|---|---|---|---|---|
| 403, `upstream_host: -`, flags `-` | **no record** | none | A's own sidecar denied: egress `AuthorizationPolicy` in A_NS, `Sidecar` resource egress hosts, or REGISTRY_ONLY + no entry (`BlackHoleCluster` → 502/503, `NR` → 404) | `A_EGRESS_POLICY` |
| 403 | 403, `upstream_host: -`, container `RBAC: access denied` | none | `AuthorizationPolicy` on B rejects: principal / namespace / path / method / header rule | `B_AUTHZ_POLICY` |
| 401 | 401, `upstream_host: -`, `Jwt is missing` / `verification fails` / `Jwks remote fetch failed` / `issuer is not configured` / `Audiences not allowed` | none | `RequestAuthentication` on B: no token forwarded, wrong issuer/aud, JWKS unreachable | `B_REQUEST_AUTHN` |
| 503 `UF`/`URX`, `tls` reason set, or 503 with `upstream_transport_failure_reason: ...TLS...` | 0/503 or no record | none | **mTLS mismatch**: B `PeerAuthentication STRICT` vs A plaintext (no sidecar / DestinationRule `tls.mode: DISABLE`), or cert expiry | `MTLS` → also `certs-tls-triage` |
| 403 | 403, real `upstream_host`, `upstream_service_time > 0` | 403 `AccessDeniedException` / `insufficient_scope` / `PreAuthorize` | B's Spring Security: token lacks scope/role, wrong audience, caller not in allowlist | `B_APP_AUTHZ` |
| 401 | 401, real upstream | 401 `JwtValidation` / `expired` / `iss`/`aud` claim / `Full authentication is required` | Token problem: A's token expired/rotated/wrong issuer, or A sent none (header stripped) | `TOKEN` |
| A_APP logs `401/400 invalid_client` / `unauthorized_client` from the token endpoint **before** calls to B | — | — | A cannot obtain a token: client secret rotated, client disabled, wrong token URL | `A_TOKEN_FETCH` |
| 403 on some `path`/`method` only | same subset | — | policy `to.operation` rule mismatch (new endpoint not added) | `B_AUTHZ_POLICY` (path) |
| 403 for A's principal only; other callers 200 | | | A's identity changed: new ServiceAccount, namespace move, or A missing from `source.principals` | `B_AUTHZ_POLICY` (principal) |
| 403 for **all** callers at exact minute | | | policy replaced/`action: DENY` added, or `ALLOW` policy deleted (deny-by-default kicks in) | `B_AUTHZ_POLICY` (global) |

Multiple rows true → the **earliest hop** wins (A sidecar before B sidecar before B app).

## Step 3 — Scope (1–2 calls)

On `B_IN AND response_code:(401 OR 403)` W1:
```json
"aggs":{"by_principal":{"terms":{"field":"source_principal.keyword","size":20,"missing":"(none)"}},
        "by_caller_ip":{"terms":{"field":"downstream_remote_address.keyword","size":20}},
        "by_path_method":{"multi_terms":{"terms":[{"field":"path.keyword"},{"field":"method.keyword"}],"size":15}},
        "tl":{"date_histogram":{"field":"@timestamp","fixed_interval":"1m"}},
        "ok_vs_denied":{"filters":{"filters":{"ok":{"range":{"response_code":{"lt":400}}},"denied":{"terms":{"response_code":[401,403]}}}},
            "aggs":{"principals":{"terms":{"field":"source_principal.keyword","size":20}}}}}
```
Record: `scope ∈ {only_A, some_callers, all_callers}`, `paths ∈ {all, subset}`, `onset_minute`. `source_principal` = `spiffe://cluster.local/ns/<ns>/sa/<sa>` — the exact string policies match on. If the field is missing (mTLS off or not logged) note "principal not visible — mTLS likely PERMISSIVE/DISABLE" and use `downstream_remote_address` → pod IP → pod (from `{pod}` lifecycle of A).

## Step 4 — Pin the onset (2–3 calls)

Against `onset_minute`:
- **istiod**: `{app}:istiod AND {msg}:(*AuthorizationPolicy* OR *RequestAuthentication* OR *PeerAuthentication* OR *"Sidecar"*) AND (B_NS OR A_NS)` W1 → push/accept/reject lines; `rejected`/`invalid` = broken policy.
- **A identity change**: 1.1 pod lifecycle for A — new image tag, or `source_principal` for A's pods changes SA name across the onset (compare `ok` principals before vs `denied` after).
- **Secret/token rotation at A**: `A_APP AND {msg}:(*token* OR *client_secret* OR *KeyVault* OR *Vault* OR *oauth* OR *"invalid_client"*)` W1.
- **JWKS/issuer reachability** (`B_REQUEST_AUTHN` with `Jwks remote fetch failed`): run `egress-triage` on the JWKS host from istiod/B sidecar logs.
- **mTLS**: `certs-tls-triage` Step 3 (SDS/istiod cert lines) if class `MTLS`.

## Step 5 — GitLab (≤ 5 calls)

| Class | Fetch (at last commit ≤ T and the one before) |
|---|---|
| `B_AUTHZ_POLICY` | all `kind: AuthorizationPolicy` with `selector` matching B (or namespace-wide, or root `istio-system`); `PeerAuthentication` for B_NS |
| `B_REQUEST_AUTHN` | `RequestAuthentication` for B: `issuer`, `jwksUri`, `audiences`, `forwardOriginalToken`; paired `AuthorizationPolicy` with `requestPrincipals` |
| `A_EGRESS_POLICY` | `AuthorizationPolicy` in A_NS, `Sidecar` resource `egress.hosts`, mesh `outboundTrafficPolicy` |
| `MTLS` | `PeerAuthentication` (B_NS + root), `DestinationRule` for B host `trafficPolicy.tls.mode`, A's deployment sidecar inject annotation |
| `B_APP_AUTHZ` / `TOKEN` | B's `application*.yml` security section (`issuer-uri`, `audiences`, `jwk-set-uri`, scope/role mapping, `@PreAuthorize` constants); A's `values.yaml` OAuth client id / token URL / scope; secret **name** of the client secret → `incident-intake`/springboot secret trace |
| `A_TOKEN_FETCH` | A's OAuth client config; IdP client registration if in GitLab; secret name last change |

Commits on those paths in [T−24h, T] → leading suspect. Diff them.

## Step 6 — Match policy to the denied request

For `B_AUTHZ_POLICY`, evaluate like Envoy does, against one denied sample (principal, namespace, path, method, headers):
1. Any `DENY` policy whose `from`/`to`/`when` matches → denied (DENY wins).
2. Else, any `ALLOW` policy selecting B exists → request must match at least one `rule`; check `source.principals` (exact spiffe string, note `sa` rename), `source.namespaces`, `to.operation.paths` (glob: `/accounts/*` vs `/accounts/v2/*`), `methods`, `ports`, `when` (`request.auth.claims[...]`, headers).
3. No `ALLOW` selects B → allowed (unless a namespace-wide/root ALLOW exists, then deny-by-default applies).
Quote the rule line that fails. Common root causes: SA renamed at A; A moved namespace; new path not in `paths`; policy switched `selector` label; `requestPrincipals` added without `RequestAuthentication`; `DENY` with empty `from` (denies all).

For `B_REQUEST_AUTHN`: compare `issuer` string with A's token `iss` (A logs it at startup/debug or from catalog), `audiences` with token `aud`, and whether A actually forwards `Authorization` (A outbound sample shows `authorization` header presence only if logged; else infer from B `Jwt is missing`).

For `B_APP_AUTHZ`/`TOKEN`: B log line names the failing check (`scope`, `aud`, `exp`, `iss`); map to A's client config (scope requested) or IdP change (issuer URL, key rotation → `JwtValidation... kid`).

## Step 7 — Output

```
S2S AUTH | A=<ns/app sa=<sa>> → B=<ns/svc> | T=<utc> onset=<minute>
Answering party: <A sidecar | B sidecar (AuthorizationPolicy|RequestAuthentication|mTLS) | B app (Spring Security) | A token fetch>
Class: <...>   Scope: <only A | callers [..] | all>   Paths: <all | [..]>
Evidence:
  1. A_OUT sample: <ts req-id code flags upstream_host>
  2. B_IN / container: <RBAC: access denied | Jwt ... | none>
  3. B_APP: <AccessDenied... | none>
  4. Onset vs change: <istiod push AuthorizationPolicy accounts-api-allow at T-3m | A image tag | SA rename | secret rotation>
  5. GitLab: <path@sha> rule <quoted line> ← denied principal/path <value>
PROPOSED FIX (not applied)
  <path> - <old line> / + <new line>   | or: rotate/redeploy A with new client secret <name> | or: add RequestAuthentication issuer <x>
  Why: <ties to evidence 5>   Risk: <who else the policy covers>   Rollback: revert <sha>
  Validate: B_IN AND response_code:403 AND source_principal:"<A spiffe>" → 0 within 2 min of push; A_OUT 200 on <path>
Gaps: <source_principal not logged | policies not in GitLab | token claims not visible>
Read-only next checks: istioctl x authz check <B pod>; istioctl pc listener <B pod> --port <p> -o json | grep rbac; kubectl get authorizationpolicy,requestauthentication,peerauthentication -n B_NS
```

## Guardrails
- Never conclude "token invalid" from a 401 at B's sidecar without the `Jwt ...` container line; a bare 401 with `upstream_host: -` can also be an ext_authz filter → `envoyfilter-lua-auth-triage`.
- Never conclude AuthorizationPolicy without `RBAC: access denied` or `upstream_host: -` at B's sidecar — a 403 with a real upstream is the app's.
- 503 with TLS reason is never an auth verdict; route to `MTLS`.
