---
name: request-origin-trace
description: Locate a failing request when its origin is unknown — React frontend, Akamai, Apigee, F5/NGINX Plus, Istio ingress, Spring Boot micro-app — using only logs. Discovers which hops are in Elastic, finds a correlation key (Akamai reference, Apigee messageid, x-request-id, traceId), walks the chain in both directions from whatever evidence the ticket gives (user, time, URL, screenshot error), and names the first hop that failed. Use for "user says it's broken", "don't know where this came from", "frontend shows error", "who called this", "is it Akamai, Apigee or us".
---

# Request Origin Trace (edge → mesh → app)

Prereq: `kibana-query-strategy`. Hand the result to the domain skill for the failing hop; finish with the orchestrator's PROPOSED FIX.

## The chain and what each hop can tell you

```
Browser/React ──▶ Akamai (edge, WAF, cache) ──▶ Apigee (API proxy, auth, routing key) ──▶ F5 / NGINX Plus (LB, TLS, WAF) 
            ──▶ istio-ingressgateway ──▶ sidecar(inbound) ──▶ Spring Boot app ──▶ sidecar(outbound) ──▶ DataAPI / Kafka / DB / egress
```

| Hop | Typical index pattern (discover!) | Correlation fields it logs | Error it produces when it is the culprit |
|---|---|---|---|
| Frontend (if shipped: RUM / browser errors / BFF) | `rum-*`, `frontend-*`, `bff-*` | `trace.id`, `session.id`, `url.full`, `http.response.status_code`, `user_agent`, `client.ip` | JS error, CORS, 0/`net::ERR_*`, timeout, 401 after token refresh |
| Akamai (DataStream 2 / LDS) | `akamai-*`, `datastream-*` | `reqId` (Akamai reference #, also shown to user as "Reference #18.xxxx"), `cliIP`, `reqHost`, `reqPath`, `statusCode`, `turnAroundTimeMSec`, `cacheStatus`, `errorCode`, `edgeIP`, `securityRules` | 403 (WAF rule), 503/504 origin timeout, cache serving stale, bot block, geo block |
| Apigee (Edge/X message logging → ELK, or Apigee analytics) | `apigee-*`, `api-proxy-*` | `messageid` (`x-apigee-message-id`), `request.header.x-request-id`, `apiproxy.name`, `proxy.pathsuffix`, `target.url`, `response.status.code`, `fault.name`, `error.state`, `client_id`/`developer.app.name`, routing key header | 401 `InvalidApiKey`/`InvalidAccessToken`, 403 quota/spike arrest, 404 no proxy matched, 500 policy fault (`ExecutionFailed`), 503 `TargetUnreachable`/`TargetTimeout`, wrong target env via routing key |
| F5 / NGINX Plus | `f5-*`, `nginx-*`, `nginxplus-*` | `$request_id` (NGINX), `X-Forwarded-For`, `upstream_addr`, `upstream_status`, `upstream_response_time`, `ssl_protocol`, ASM `support_id` | 502 upstream down, 504, 499 client closed, ASM/WAF block (`support_id`), TLS handshake, `no live upstreams` |
| istio-ingressgateway | app log pattern, `{ns}:istio-system` | `x_request_id`, `authority`, `path`, `response_flags`, `downstream_remote_address`, `x_forwarded_for` | see envoy skill flags |
| sidecar / app | app log pattern | `x_request_id`, `traceId`/`spanId`, `url.path`, `error.type` | see springboot / lua skills |

## Step 0 — Discover which hops exist in Elastic (once per env, cache it)

```http
GET _cat/indices/*akamai*,*datastream*,*apigee*,*api-proxy*,*f5*,*nginx*,*rum*,*frontend*,*bff*?h=index,docs.count,creation.date.string&s=index
```
Then `_field_caps` on each for the correlation fields above. Record a **hop map**: `{hop: {index, fields present}}`. Missing hop → you will infer around it (Step 4). Say which hops are blind.

## Step 1 — Pick the entry point from whatever the ticket gives

| You have | Start query |
|---|---|
| Akamai reference number (user screenshot "Reference #18.ab12cd…") | `akamai: reqId:"18.ab12cd…"` → gives `statusCode`, `reqHost`, `reqPath`, `cliIP`, time, origin response; this is **the** anchor |
| Apigee `messageid` / `x-apigee-message-id` (from API client/ partner) | `apigee: messageid:"…"` → `x-request-id` it forwarded, `target.url`, `fault.name` |
| `x-request-id` / `traceId` (from app error page or dev tools) | cross-skill request-id query, no label filter, ±1h, all index patterns |
| URL + approximate time + user (most common) | Step 2 funnel |
| Only "it's broken" + time | Step 2 funnel with `status >= 400` on every hop, aggregate by `host`/`path`, find the hop whose error rate jumped |
| Frontend error text (CORS, 401 loop, "Something went wrong") | Step 3 frontend-first |

## Step 2 — The funnel (URL + time, no IDs)

Run the **same filter on every hop that exists**, tight window, aggregate counts by status. The first hop in the chain whose status differs from the hop before it is where the request died.

```json
// per hop, substitute fields from hop map
{"size":0,"query":{"bool":{"filter":[RANGE,
   {"wildcard":{"<host_field>":"*HOST*"}},{"wildcard":{"<path_field>":"*PATH*"}}]}},
 "aggs":{"status":{"terms":{"field":"<status_field>","size":10}},
         "timeline":{"date_histogram":{"field":"@timestamp","fixed_interval":"1m"},
            "aggs":{"err":{"filter":{"range":{"<status_field>":{"gte":400}}}}}},
         "samples":{"top_hits":{"size":3,"_source":["@timestamp","<id_fields>","<status_field>","<client_ip>","<upstream/target>"]}}}}
```

Read the funnel top-down:

| Akamai | Apigee | F5/NGINX | Istio GW | App | Verdict |
|---|---|---|---|---|---|
| 403/`securityRules` set | no record | — | — | — | **Akamai WAF/bot/geo block**; quote rule id |
| 5xx, `errorCode: ERR_CONNECT_TIMEOUT`/`ERR_READ_TIMEOUT` | no record | — | — | — | Akamai can't reach origin → Apigee/F5 reachability (DNS, cert, firewall) → `egress`-style check on the edge path |
| 200 `cacheStatus: HIT` | no record | — | — | — | **Stale from cache**; app never saw it. Check TTL/purge |
| 200 | 401/403 `fault.name` | — | — | — | **Apigee auth/quota**: `InvalidApiKey`, `access_token_expired`, `QuotaViolation`, `SpikeArrestViolation` |
| 200 | 404 `no proxy matched` / wrong `target.url` | — | — | — | **Apigee routing**: basepath or **routing-key header → wrong target env** (canary/IT-UAT consolidation) |
| 200 | 503 `TargetUnreachable`/`TargetTimeout` | no record | — | — | Apigee → F5 path down (cert, DNS, firewall) |
| 200 | 200 | 502 `upstream_status`/`no live upstreams` | no record | — | **F5/NGINX → Istio GW down**: pool members, health monitor, TLS to gateway |
| 200 | 200 | 200 | 503 `UH`/`UF`/`NR` | no record | mesh: → `envoy-istio-gateway-triage` |
| 200 | 200 | 200 | 401, `upstream_host:-` | no record | → `envoyfilter-lua-auth-triage` |
| 200 | 200 | 200 | 5xx `-` | ERROR | app: → `springboot-dataapi-triage` |
| 200 | 200 | 200 | 200 | 200 | **Frontend problem** (JS, CORS preflight, token refresh loop, wrong env URL) → Step 3 |
| nothing anywhere | | | | | wrong host/path/time, or request never left the browser (DNS, Akamai edge not logged, ad-blocker) → Step 3 |

Statuses may **differ per hop legitimately**: Akamai 504 while F5 shows 200 slow = Akamai origin timeout shorter than app latency → report the two numbers (`turnAroundTimeMSec` vs `upstream_response_time`).

## Step 3 — Frontend-first

Browser can't log into Elastic unless RUM/BFF is shipped. Use:
- RUM/BFF index: `session.id` or `user.id` → list of API calls with status → pick the first ≥400 → its `trace.id`/`x-request-id` → Step 1.
- No RUM: ask the user for the **Akamai reference #** (on Akamai error pages) or the **dev-tools `x-request-id`/`traceparent`** response header. Give them the exact header names to copy.
- CORS: look at Apigee/Istio for `OPTIONS` with ≠ 200/204 on the path; `Origin` header value vs allowed origins (Apigee CORS policy / Istio `CorsPolicy` in VS — fetch from GitLab).
- 401 loop after login: Apigee `access_token_expired` + frontend retry storm (high `OPTIONS`/`GET` rate from one `cliIP`) → token refresh broken; see `envoyfilter-lua-auth-triage` Step 4 #5–7 for issuer/audience.

## Step 4 — Blind hops (not in Elastic)

Infer from neighbours and say so:
- **No Akamai logs**: F5/Apigee `X-Forwarded-For` first IP = Akamai edge; `True-Client-IP` = user. If Apigee shows nothing for the window but users report errors → Akamai (or DNS before it) ate it. Ask for reference #.
- **No Apigee logs**: Istio GW `x_forwarded_for` chain and `user_agent`; Apigee adds headers (`x-apigee-*`, routing key). Presence of routing-key header at the GW = came via Apigee; absence = direct/internal caller.
- **No F5/NGINX logs**: Istio GW `downstream_remote_address` = F5 SNAT IP → confirms path; 502 at Akamai/Apigee with no GW record → F5 or its pool.
- Always print the **hop map** and which verdicts are inferred.

## Step 5 — Who originated it (unknown caller)

At the Istio GW / sidecar inbound:
```json
"aggs":{"xff":{"terms":{"field":"x_forwarded_for.keyword","size":20}},
        "ua":{"terms":{"field":"user_agent.keyword","size":10}},
        "authority":{"terms":{"field":"authority.keyword","size":10}},
        "routing_key":{"terms":{"field":"<x-routing-key header field>.keyword","size":10,"missing":"(none)"}},
        "apigee_app":{"terms":{"field":"<x-apigee-developer-app field>.keyword","size":10,"missing":"(none)"}}}
```
- `x_forwarded_for` first hop in Akamai ranges + Apigee headers → **external via full edge chain**.
- `x_forwarded_for` = internal CIDR, no Apigee headers → **internal caller / another micro-app / batch** → sidecar *outbound* logs of candidate callers: `upstream_cluster:*APP.NS.svc*` aggregated by `{ns},{app}`.
- `user_agent` = `Java/…`, `Apache-HttpClient`, `okhttp` → service-to-service; `Mozilla` → browser; `curl`/`python-requests` → script/test.
- Apigee `developer.app.name` / `client_id` → partner/channel.

## Step 6 — Secrets: trace from symptom to source (when "we don't know which secret")

Start from the **symptom hop**, never from the vault:

| Symptom | Which secret | Where it's defined (fetch from GitLab) |
|---|---|---|
| Apigee 401 `InvalidApiKey`/`InvalidAccessToken` | API key / OAuth client of the *caller*; or Apigee→target credential (`AssignMessage` basic auth, KVM) | Apigee proxy bundle (`apiproxy/policies/*.xml`), KVM entry name; caller's config |
| Akamai origin 401/403 | Origin auth header / mTLS cert Akamai presents | Akamai property config (not in Kibana; ask) |
| F5/NGINX → GW TLS error | Edge client cert / CA bundle | F5 SSL profile / `nginx.conf` ssl_* in GitLab |
| Istio GW 401 `upstream_host:-` | JWT issuer/JWKS or Lua token secret | `RequestAuthentication`, EnvoyFilter in GitLab → `envoyfilter-lua-auth-triage` |
| Spring `Could not resolve placeholder 'X'` | property `X` | `values.yaml`/ConfigMap key `X` → if templated from Key Vault (`secretProviderClass`, `azure.keyvault.secret-name`), the Key Vault secret name → check app log for `KeyVault ... Forbidden/NotFound` and k8s events `FailedMount` (CSI) |
| Spring `401`/`403` calling dependency | that dependency's client credential | `values.yaml` env → Key Vault ref → springboot skill secret sweep for rotation time |
| JDBC `invalid username/password` | DB cred | same path → `database-triage` |
| Kafka `SaslAuthenticationException` | SASL/SCRAM or keystore | same → `kafka-triage` |

Procedure: (1) take the exact property/secret **name** from the log line; (2) GitLab search that name across the app chart, centralized config repo, and platform repo; (3) follow the reference to its source type (literal / ConfigMap / k8s Secret / Key Vault CSI / Vault agent); (4) find the last change to that source (commit on values, or `KeyVault` rotation lines in app logs / k8s `SecretProviderClass` events); (5) correlate with first-error minute. Report **name → source → last change → correlation**. You cannot read secret *values* and must not try; you report names and timestamps only.

## Output

```
HOP MAP       akamai:<idx|blind> apigee:<idx|blind> f5:<idx|blind> istio-gw:<idx> app:<idx|not shipped>
ANCHOR        <akamai reqId | apigee messageid | x-request-id | traceId | funnel(url,time)>
FUNNEL        akamai <st> → apigee <st,fault> → f5 <st,upstream_status> → gw <st,flags> → app <st,error.type>
FIRST FAILING HOP  <hop> — <one sentence>        inferred?: <yes (blind hop) | no>
ORIGINATOR    <external via Akamai/Apigee app <name> | internal <ns/app> | browser UA | script>
SECRET TRACE  <name → source → last changed T → matches first error? y/n>   (if applicable)
→ hand off to <domain skill> with <ids, hop, window>; orchestrator adds PROPOSED FIX
EVIDENCE      <≤3 lines, one per hop that mattered>
GAPS          <blind hops, missing correlation header, RUM absent>
ASK THE USER  <Akamai reference # | x-request-id response header | time + URL + account>  (only what's needed)
```
