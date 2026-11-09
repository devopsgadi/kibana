---
name: envoy-istio-gateway-triage
description: Diagnose Istio ingress gateway, Envoy sidecar, EnvoyFilter, VirtualService/DestinationRule and mesh routing failures from Kibana access logs — 502/503/504/404, response flags (UF/UH/UO/UT/NR/UAEX/DC), request tracing gateway→sidecar→app, latency, pod lifecycle and bad-deploy correlation. Use for "gateway error", 5xx at edge, route not found, timeouts, or any istio-proxy symptom.
---

# Envoy / Istio / Gateway Triage

Prereq: `kibana-query-strategy` loaded, field map done. Placeholders `{ns} {app} {pod} {container} {image} {msg}` resolved.

## Base filters

```
BASE      = {ns}:"NS" AND {app}:"APP"
APP_LOGS  = BASE AND NOT {container}:"istio-proxy"
SIDECAR   = BASE AND {container}:"istio-proxy"
GATEWAY   = {ns}:"istio-system" AND {app}:"istio-ingressgateway"      # override GW_NS/GW_APP per env
EGRESS_GW = {ns}:"istio-system" AND {app}:"istio-egressgateway"
SVC_MATCH = upstream_cluster:*SVC.NS.svc.cluster.local*
```

## Step 1 — Gateway-side errors

```json
{"size":0,"query":{"bool":{"filter":[
  {"range":{"@timestamp":{"gte":"FROM","lte":"TO"}}},
  {"term":{"{ns}":"istio-system"}},{"term":{"{app}":"istio-ingressgateway"}},
  {"wildcard":{"upstream_cluster":"*SVC.NS.svc.cluster.local*"}},
  {"bool":{"should":[{"range":{"response_code":{"gte":500}}},{"term":{"response_code":0}}]}}]}},
 "aggs":{
  "code_flag":{"multi_terms":{"terms":[{"field":"response_code"},{"field":"response_flags"}],"size":15}},
  "paths":{"terms":{"field":"path.keyword","size":10}},
  "authority":{"terms":{"field":"authority.keyword","size":5}},
  "first":{"min":{"field":"@timestamp"}},"last":{"max":{"field":"@timestamp"}},
  "samples":{"top_hits":{"size":3,"_source":["@timestamp","x_request_id","response_code","response_flags","upstream_host","duration","path"]}}}}
```

App unknown? Drop the `upstream_cluster` filter, aggregate `terms` on `upstream_cluster` → the failing service surfaces.

Gateway shows **nothing** but users see 5xx → failure is upstream of the mesh (F5 / NGINX Plus / Akamai). Query the edge LB index for same host+window and report "failed before Istio ingress".

## Step 2 — Trace one request

Take an `x_request_id` from Step 1 samples; run the cross-skill request-id query (no label filter, ±1h, sort asc).

| Logged by | Missing | Verdict |
|---|---|---|
| gateway only | app sidecar | never reached pod → read gateway flag (UH/UF/NR/UAEX) |
| gateway + sidecar inbound | app container | sidecar→app failed (app down, wrong port, crash) → `springboot-dataapi-triage` |
| app container logs it | — | app owned it; check its ERROR lines + outbound (Step 4) |
| sidecar outbound 5xx | — | dependency; name it from `upstream_cluster`/`authority`; recurse |

## Step 3 — Response flag decoder

| Code + flag | Meaning | Next skill |
|---|---|---|
| 503 `UH` | no healthy upstream — pods not Ready / endpoints empty | pod lifecycle (Step 6) |
| 503 `UF` | upstream connection failure (refused) | springboot (crash/port) |
| 503 `UO` | upstream overflow — DestinationRule connectionPool / circuit breaker | latency (Step 5) + database/kafka |
| 503 `URX` | retries exhausted | whatever the underlying flag is |
| 503 `UC` | upstream connection terminated | springboot (OOM / restart) |
| 503 `NR` (also 404) | no route — VirtualService host/path/port mismatch, Gateway `hosts` missing | routing (Step 7) |
| 503 `UAEX` | ext authz denied/unavailable (EnvoyFilter ext_authz) | EnvoyFilter (Step 8) |
| 503 `UPE` / `UMSDR` | upstream protocol / max stream duration | springboot |
| 502 `-` | app itself returned 502 (usually its own dependency) | springboot → database/egress |
| 504 `UT` | upstream request timeout (VS `timeout` too low or app slow) | latency (Step 5) |
| 0 `DC` | downstream (client) disconnected | client / F5 idle timeout; not app |
| 0 `DT` / `DPE` | downstream timeout / protocol error | edge LB |
| 404 `-` | app says no such path → version skew, wrong context path | springboot (compare image tags) |
| 403 `RL` | rate limited (EnvoyFilter ratelimit) | EnvoyFilter |
| 403 `-` + `RBAC: access denied` in msg | AuthorizationPolicy | routing/policy |
| `NC` | no cluster found — upstream service deleted / ServiceEntry missing | egress-triage if external |
| `LH` | local service failed health check | pod lifecycle |
| `UIF` | invalid upstream frame (h2) | springboot |

Multiple flags appear together (e.g. `UF,URX`) — the first explains, the rest are consequences.

## Step 4 — Outbound dependencies of the app

```json
{"size":0,"query":{"bool":{"filter":[RANGE,{"term":{"{ns}":"NS"}},{"term":{"{app}":"APP"}},
   {"term":{"{container}":"istio-proxy"}},{"prefix":{"upstream_cluster":"outbound"}}]}},
 "aggs":{"dep":{"terms":{"field":"authority.keyword","size":20},
   "aggs":{"codes":{"terms":{"field":"response_code","size":6}},
           "p99":{"percentiles":{"field":"duration","percents":[50,95,99]}}}}}}
```

Cross-namespace dependency with 5xx in the same window = likely root cause → recurse the orchestrator on that service. Hosts outside `.svc.cluster.local` → `egress-triage`. HTTPS egress shows only SNI (`requested_server_name`), no path.

## Step 5 — Latency

```json
"aggs":{"lat_over_time":{"date_histogram":{"field":"@timestamp","fixed_interval":"1m"},
  "aggs":{"p99":{"percentiles":{"field":"duration","percents":[99]}},
          "ust_p99":{"percentiles":{"field":"upstream_service_time","percents":[99]}}}}}
```
`duration` ≫ `upstream_service_time` → time spent in Envoy/connect/retry (pool exhaustion, DNS, TLS). `duration ≈ upstream_service_time` → the app is slow → springboot/database.

## Step 6 — Pod lifecycle & bad deploy (always run)

Pod lifecycle aggregation from `kibana-query-strategy` on `BASE`. Interpret:
- pod `last` inside window → died during incident → fetch its final 50 `APP_LOGS` lines (sort desc)
- new `{image}` tag beginning just before first error → bad deploy; report `old → new @ T`
- pod stops logging with no ERROR → OOMKill / node loss; confirm via k8s events index if shipped (`kubernetes.event.reason: OOMKilling|BackOff|Unhealthy|FailedMount`)

## Step 7 — Routing / VirtualService checks (NR, 404)

From gateway logs: aggregate `authority` × `path` × `route_name` for the failing host. `route_name: "-"` or missing on NR = no VS matched. Compare `authority` against the catalog's expected host; check for trailing-dot, port suffix (`host:443`), or case mismatches. Istiod logs (`{ns}:istio-system AND {app}:istiod`) in the window: `"rejected"`, `"conflict"`, `"duplicate"`, `"no endpoints"` reveal config push failures:

```
{ns}:"istio-system" AND {app}:"istiod" AND {msg}:(*rejected* OR *conflict* OR *"Push failed"* OR *"no endpoints"* OR *validation*)
```

## Step 8 — EnvoyFilter / ext_authz / ratelimit

Signals: `UAEX`, `RL`, 401/403 with no app log, `upstream_host: "-"` on a 4xx, latency added before `upstream_service_time`.

**401/403 generated by Envoy itself (no app-container log for the request) → stop here and hand off to `envoyfilter-lua-auth-triage`**, which reads the Lua/JWT/ext_authz filter source from GitLab and proposes the fix. Logs alone cannot explain a Lua decision.

```
SIDECAR AND {msg}:(*ext_authz* OR *"authz"* OR *ratelimit* OR *lua* OR *wasm* OR *"filter chain"*)
GATEWAY AND {msg}:(*ext_authz* OR *ratelimit* OR *wasm*)
```
Also query the ext-authz service itself (from catalog) for 5xx/timeouts in the window. EnvoyFilter changes ship as istiod pushes — correlate with istiod `Push debounce`/`xds` lines right before first error. Report "EnvoyFilter <name> likely changed at T" with the istiod evidence; you cannot read the CR from Kibana, say so.

## Step 9 — Empty-result sanity

1. Other apps have istio-proxy docs in window? No → access logging disabled or shipper down.
2. Label wrong? Retry `{pod}:APP-*`.
3. Timezone? Re-state conversion.
4. Frozen tier timeout? Narrow window, use `_async_search`.

## Output

Append the standard evidence block from `kibana-query-strategy` plus:
```
Failing hop: <edge | gateway | sidecar→app | app | dependency X>
Flag verdict: <code+flag → meaning>
Deploy correlation: <none | image A→B at T>
```

## Reference: Envoy flag glossary
UH no healthy upstream · UF upstream connection failure · UO upstream overflow (CB) · NR no route · URX retry limit · NC no cluster · DT downstream timeout · UT upstream timeout · UC upstream connection termination · DC downstream connection termination · LH local health check fail · LR local reset · UR upstream reset · DI delay injected · FI fault injected · RL rate limited · UAEX unauthorized ext authz · RLSE rate limit service error · IH invalid header · SI stream idle · DPE downstream protocol error · UPE upstream protocol error · UMSDR upstream max stream duration · OM overload manager · DF DNS fail · UIF upstream invalid frame
