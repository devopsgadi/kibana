---
name: evidence-collection-plan
description: Fixed, ordered evidence-collection procedure the agent executes BEFORE reasoning about root cause — pod level → inbound path → app internals → dependencies → edge (Akamai/Apigee/F5) → 30-day baseline → GitLab config. Every step has the exact query, the output key to record, and the gate that decides the next step. Produces an EVIDENCE ledger that the domain skills then decode. Run this first on every incident; do not improvise queries outside it.
---

# Evidence Collection Plan

You are a stateless model in a tool-call loop. You do not plan queries; you execute this list. After each phase, append to the **EVIDENCE ledger** (format at the end) in your reply so it survives history truncation. Then continue. Only after Phase 8 do you load domain skills and reason.

Prereq: `kibana-query-strategy` (field map, async rules). Placeholders `{ns} {app} {pod} {container} {image} {level} {msg}` resolved in 0.3.
Windows: `W0=[T-15m,T+15m]  W1=[T-2h,T+2h]  W30=[T-30d,T]`. `RANGE(W)` = `{"range":{"@timestamp":{"gte":"<from>","lte":"<to>"}}}`.
Base filters: `NSF={"term":{"{ns}":"NS"}}  APPF={"term":{"{app}":"APP"}}  NOSC={"bool":{"must_not":{"term":{"{container}":"istio-proxy"}}}}  SC={"term":{"{container}":"istio-proxy"}}  GWF=[{"term":{"{ns}":"GW_NS"}},{"term":{"{app}":"GW_APP"}}]`.

Hard caps: 40 ES calls, 6 GitLab calls, 1 catalog call. W30 queries → `_async_search` with the dated index list from 0.5. Samples: max 5, `msg[:300]`.

---

## Phase 0 — Resolve (1 catalog call, 3 ES calls)

**0.0 Intake** — run `incident-intake` first. It delivers `intake{T, env, app, ns, host, path, ids, known_level, assumptions[]}`. If `known_level` is E with no anomaly, stop there (intake report). Otherwise copy `intake` into the ledger and continue; derived `T` or `app` ⇒ start with W1 instead of W0 and `confidence_cap=medium`.

**0.1 Parse ticket** (no tool; skip if intake already filled it). Extract and record `ticket{T_utc, tz_note, env, app, ns, ids{akamai, apigee, request_id, trace_id}, url, status, error_text}`.
Regexes: Akamai ref `\b\d{2}\.[0-9a-f]{6,}\b` · Apigee `rrt-[\w-]+` · request-id/uuid `[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}` · traceId `\b[0-9a-f]{16,32}\b` · status `\b(4\d\d|5\d\d)\b`.

**0.2 Catalog** → `catalog{ns, svc, deployment, deps[], topics[], db, egress_hosts[], owner, repo, gw_ns, gw_app, edge_hosts[], source:"catalog"}`. Miss → `source:"derived"`, defaults `gw_ns=istio-system gw_app=istio-ingressgateway`, fill `deps` at 3.4.

**0.3 Field map**
```
GET APP_INDEX/_field_caps?fields=kubernetes.*,log.level,level,message,log,response_code,response_flags,upstream_cluster,upstream_host,x_request_id,authority,path,duration,upstream_service_time,container.image.name,logger_name,error.type,traceId,trace.id,url.path,http.response.status_code,kubernetes.event.*
```
Record `fieldmap{…, envoy_parsed:bool}`. `envoy_parsed=false` → every Envoy step below uses `{"wildcard":{"{msg}":"*…*"}}` on the text access log and you add gap `envoy_access_logs_unparsed`.

**0.4 Hop map**
```
GET _cat/indices/*akamai*,*datastream*,*apigee*,*api-proxy*,*f5*,*nginx*,*rum*,*bff*,*k8s-event*,*event*?h=index,docs.count&s=index
```
Record `hopmap{akamai, apigee, f5, rum, k8s_events: <index|null>}`.

**0.5 Tiers**
```
GET _cat/indices/APP_INDEX_PATTERN?h=index,creation.date.string,pri.store.size&s=creation.date
```
Record `tiers{indices_w30[], indices_w1[]}`. Use `indices_w30` explicitly on every W30 call.

## Phase 1 — Pod level (≤ 6 ES calls)

**1.1 Pod lifecycle W1**
```json
{"size":0,"query":{"bool":{"filter":[RANGE(W1),NSF,APPF]}},
 "aggs":{"pods":{"terms":{"field":"{pod}","size":50},"aggs":{
   "first":{"min":{"field":"@timestamp"}},"last":{"max":{"field":"@timestamp"}},
   "image":{"terms":{"field":"{image}","size":3}},
   "by_container":{"terms":{"field":"{container}","size":5}}}}}}
```
→ `pods[]{name, first, last, images[], app_docs, sidecar_docs}`.
Gate: any pod with `app_docs==0 && sidecar_docs>0` → run 1.5. Any pod `last` inside W0 → `flags.pod_died_in_window=[names]`.

**1.2 Image timeline W30 (async)**
```json
{"size":0,"query":{"bool":{"filter":[RANGE(W30),NSF,APPF]}},
 "aggs":{"img":{"terms":{"field":"{image}","size":10},"aggs":{"first":{"min":{"field":"@timestamp"}}}}}}
```
→ `deploys[]{image, first_seen}`. Gate: `first_seen ∈ [T-24h,T]` → `flags.deploy_before_incident={image, at}`.

**1.3 K8s events W1** (only if `hopmap.k8s_events`)
```json
{"size":10,"sort":[{"@timestamp":"desc"}],"query":{"bool":{"filter":[RANGE(W1),
  {"term":{"kubernetes.event.involved_object.namespace":"NS"}},
  {"prefix":{"kubernetes.event.involved_object.name":"APP"}},
  {"term":{"kubernetes.event.type":"Warning"}}]}},
 "aggs":{"reason":{"terms":{"field":"kubernetes.event.reason","size":10}}}}
```
→ `k8s_events[]{ts, reason, object, msg}`. Gate: reason ∈ {OOMKilling, BackOff, Unhealthy, FailedMount, CreateContainerConfigError, FailedScheduling, FailedCreatePodSandBox} → `flags.pod_unhealthy=[reasons]`.

**1.4 Startup markers W1**
```json
{"size":100,"sort":[{"@timestamp":"asc"}],"_source":["@timestamp","{pod}","{msg}"],
 "query":{"bool":{"filter":[RANGE(W1),NSF,APPF,NOSC,
  {"query_string":{"query":"\"Starting \" OR \"Started \" OR \"APPLICATION FAILED TO START\" OR \"started on port\" OR \"Shutdown initiated\" OR \"Closing JPA\"","fields":["{msg}"]}}]}}}
```
→ `startups[]{pod, ts, marker}`. Gate: `Starting` with no `Started` within 3m on same pod → `flags.crashloop=true`. `APPLICATION FAILED TO START` → **1.4b**: fetch 80 lines before it on that pod (`size:80, sort desc, lte: that ts`) → `startup_failure_lines[]` (keep lines containing `Caused by`, `Exception`, `placeholder`, `Bean`, `port`, `profile`).

**1.5 Shipper check W30 (async)** — only if gated
```json
{"size":0,"query":{"bool":{"filter":[RANGE(W30),NSF,APPF]}},
 "aggs":{"daily":{"date_histogram":{"field":"@timestamp","fixed_interval":"1d","min_doc_count":0,"extended_bounds":{"min":"<W30.from/d>","max":"<T/d>"}},
   "aggs":{"app_only":{"filter":NOSC}}}}}
```
→ `shipping{daily[], off_since}`. Gate: `off_since` → `flags.app_logs_not_shipped=true`, queue 7.2, **skip 3.1–3.3**.

**1.6 Readiness W1**
```json
{"size":0,"query":{"bool":{"filter":[RANGE(W1),NSF,APPF,SC,
  {"query_string":{"query":"path:(\\/actuator\\/health* OR \\/health* OR \\/ready*)"}},
  {"bool":{"must_not":{"term":{"response_code":200}}}}]}},
 "aggs":{"per_pod":{"terms":{"field":"{pod}","size":30},"aggs":{"first":{"min":{"field":"@timestamp"}}}}}}
```
→ `readiness_failures[]`. Gate: any → `flags.readiness_flapping=true`.

## Phase 2 — Inbound path (≤ 8 ES calls)

**2.1 Gateway errors W0**
```json
{"size":0,"query":{"bool":{"filter":[RANGE(W0),...GWF,
  {"wildcard":{"upstream_cluster":"*SVC.NS.svc.cluster.local*"}},
  {"bool":{"should":[{"range":{"response_code":{"gte":500}}},{"term":{"response_code":0}}],"minimum_should_match":1}}]}},
 "aggs":{"code_flag":{"multi_terms":{"terms":[{"field":"response_code"},{"field":"response_flags"}],"size":15}},
  "paths":{"terms":{"field":"path.keyword","size":10}},"authority":{"terms":{"field":"authority.keyword","size":5}},
  "first":{"min":{"field":"@timestamp"}},"last":{"max":{"field":"@timestamp"}},
  "samples":{"top_hits":{"size":5,"_source":["@timestamp","x_request_id","response_code","response_flags","upstream_host","duration","path"]}}}}
```
→ `gw.errors`. Gate: 0 hits → rerun with W1; still 0 → `flags.gw_clean=true`.

**2.2 Gateway timeline W1** (same filter, no code restriction)
```json
"aggs":{"tl":{"date_histogram":{"field":"@timestamp","fixed_interval":"1m","min_doc_count":0},
  "aggs":{"e5":{"filter":{"range":{"response_code":{"gte":500}}}},"p99":{"percentiles":{"field":"duration","percents":[50,99]}}}}}
```
→ `gw.timeline[]`. Gate (compute): first minute where `e5/total > 0.05` and previous 5 minutes < 0.01 → `flags.gw_error_start=M`.

**2.3 Sidecar inbound errors W0**
```json
{"size":0,"query":{"bool":{"filter":[RANGE(W0),NSF,APPF,SC,{"prefix":{"upstream_cluster":"inbound"}},{"range":{"response_code":{"gte":400}}}]}},
 "aggs":{"code_flag":{"multi_terms":{"terms":[{"field":"response_code"},{"field":"response_flags"}],"size":15}},
  "paths":{"terms":{"field":"path.keyword","size":10}},
  "no_upstream":{"filter":{"term":{"upstream_host.keyword":"-"}}},
  "samples":{"top_hits":{"size":5,"_source":["@timestamp","x_request_id","response_code","response_flags","upstream_host","upstream_service_time","path","user_agent"]}}}}
```
→ `sidecar.inbound_errors`. Gate: `(401|403) && no_upstream>0` → `flags.envoy_generated_auth=true` → 3.5 + 7.1.

**2.4 Request trace** — up to 3 ids: ticket ids first, else `x_request_id` from 2.1/2.3 samples.
```json
{"size":100,"sort":[{"@timestamp":"asc"}],
 "_source":["@timestamp","{ns}","{app}","{container}","{pod}","response_code","response_flags","upstream_cluster","upstream_host","upstream_service_time","{msg}"],
 "query":{"bool":{"filter":[{"range":{"@timestamp":{"gte":"T-1h","lte":"T+1h"}}},
  {"multi_match":{"query":"ID","fields":["x_request_id","x-request-id","request_id","trace.id","traceId","{msg}"]}}]}}}
```
→ `traces[]{id, hops[]{ts, ns, app, container, code, flags, upstream_host, msg}, last_hop}`. Compute `last_hop ∈ {gateway, sidecar_in, app, sidecar_out:<authority>}` = last hop that logged.

**2.5 Proxy container log W0** (GW pods + app pods)
```json
{"size":15,"sort":[{"@timestamp":"asc"}],"_source":["@timestamp","{ns}","{app}","{pod}","{msg}"],
 "query":{"bool":{"filter":[RANGE(W0),SC,{"bool":{"should":[{"bool":{"filter":[NSF,APPF]}},{"bool":{"filter":GWF}}],"minimum_should_match":1}},
  {"query_string":{"query":"lua OR \"script log\" OR ext_authz OR jwt OR Jwt OR JWKS OR jwks OR \"TLS error\" OR SDS OR \"attempt to\" OR \"stack traceback\" OR \"nil value\" OR CERTIFICATE_VERIFY_FAILED OR HANDSHAKE_FAILURE","fields":["{msg}"]}}]}}}
```
→ `proxy_log[]`. Gate: hits → `flags.lua_or_auth_or_tls=true`.

**2.6 Istiod W1**
```json
{"size":10,"sort":[{"@timestamp":"desc"}],"_source":["@timestamp","{msg}"],
 "query":{"bool":{"filter":[RANGE(W1),{"term":{"{ns}":"istio-system"}},{"term":{"{app}":"istiod"}},
  {"query_string":{"query":"(rejected OR conflict OR \"Push failed\" OR \"no endpoints\" OR EnvoyFilter OR \"root cert\" OR expired) AND (\"NS\" OR \"APP\" OR \"GW_APP\")","fields":["{msg}"]}}]}}}
```
→ `istiod[]`.

## Phase 3 — App internals (≤ 6 ES calls; skip 3.1–3.3 if `flags.app_logs_not_shipped`)

**3.1 Error fingerprint W0 (W1 if empty)**
```json
{"size":0,"query":{"bool":{"filter":[RANGE(W0),NSF,APPF,NOSC,{"terms":{"{level}":["ERROR","FATAL","WARN","error","warn"]}}]}},
 "aggs":{"type":{"terms":{"field":"error.type.keyword","size":15,"missing":"(none)"}},
  "logger":{"terms":{"field":"logger_name.keyword","size":15}},
  "endpoint":{"terms":{"field":"url.path.keyword","size":10}},
  "status":{"terms":{"field":"http.response.status_code","size":6}},
  "per_pod":{"terms":{"field":"{pod}","size":30}},
  "tl":{"date_histogram":{"field":"@timestamp","fixed_interval":"1m"}},
  "samples":{"top_hits":{"size":5,"sort":[{"@timestamp":"desc"}],"_source":["@timestamp","{pod}","traceId","error.type","logger_name","{msg}"]}}}}
```
→ `app.errors`. If `error.type` absent, add scripted 120-char prefix agg from `kibana-query-strategy`.

**3.2 Classify (no tool)** — match `app.errors.samples` + `startup_failure_lines` against this table; record `app.classes[]{class, count, first_seen, sample, extracted}`:

| Class | Regex (case-insensitive) | Extract |
|---|---|---|
| CONFIG_MISSING | `Could not resolve placeholder '([^']+)'` · `APPLICATION FAILED TO START` · `BeanCreationException` · `UnsatisfiedDependency` · `No active profile` | property |
| SECRET_AUTH | `KeyVault.*(Forbidden\|NotFound\|Unauthorized)` · `Vault.*(403\|permission denied)` · `FailedMount` · `secret .* not found` | secret name |
| DB_POOL | `HikariPool-\d+ - Connection is not available` · `request timed out after (\d+)ms` · `Pool stats` | timeout, pool stats |
| DB_CONN | `CannotGetJdbcConnection` · `Connection refused` · `ORA-1200\d` · `ORA-03113` · `ORA-03135` · `ORA-12514` · `ORA-12170` · `Communications link failure` · `recovery mode` | host, ora_code |
| DB_AUTH | `ORA-01017` · `ORA-28001` · `Login failed for user` · `password authentication failed` | user |
| DB_CAPACITY | `ORA-00020` · `too many connections` · `remaining connection slots` · `ORA-0165[23]` | ora_code |
| DB_LOCK | `ORA-00060` · `deadlock` · `Lock wait timeout` · `ORA-00054` | endpoint |
| DB_SCHEMA | `Flyway` · `Liquibase` · `checksum mismatch` · `Validate failed` | migration |
| CASSANDRA | `NoHostAvailable` · `AllNodesFailed` · `(Read\|Write)TimeoutException` · `UnavailableException` · `OverloadedException` · `BusyPool` | consistency, required/alive |
| KAFKA_REBALANCE | `rebalanc` · `\(Re-\)joining group` · `heartbeat failed` · `CommitFailedException` · `UNKNOWN_MEMBER_ID` · `ILLEGAL_GENERATION` | group |
| KAFKA_BROKER | `Connection to node -?\d+ .* could not be established` · `Bootstrap broker .* disconnected` · `NOT_LEADER` · `LEADER_NOT_AVAILABLE` · `Expiring \d+ record` | bootstrap, node |
| KAFKA_AUTH | `TOPIC_AUTHORIZATION_FAILED` · `GROUP_AUTHORIZATION_FAILED` · `SaslAuthentication` | topic |
| KAFKA_POISON | `SerializationException` · `Error deserializing .* partition (\S+) at offset (\d+)` · `ListenerExecutionFailed` | partition, offset |
| KAFKA_TOPIC | `UNKNOWN_TOPIC_OR_PARTITION` · `RecordTooLarge` | topic |
| TLS | `SSLHandshakeException` · `PKIX path building failed` · `certificate_unknown` · `handshake_failure` · `No subject alternative` · `NotAfter` · `Keystore was tampered` · `CERTIFICATE_VERIFY_FAILED` | host |
| DNS | `UnknownHostException: (\S+)` · `EAI_AGAIN` · `NXDOMAIN` · `Name or service not known` | host |
| DEP_TIMEOUT | `RetryableException: Read timed out executing \w+ (\S+)` · `ReadTimeout` · `SocketTimeout` · `AsyncRequestTimeout` | url |
| DEP_5XX | `HttpServerErrorException: (5\d\d)` · `WebClientResponseException\$\w+` · `ServiceUnavailable` · `BadGateway` | status, url |
| DEP_RESET | `Connection reset` · `Broken pipe` · `Connection prematurely closed` | url |
| CB_OPEN | `CircuitBreaker '([^']+)' is OPEN` · `HystrixRuntimeException` | breaker |
| OOM | `OutOfMemoryError` · `GC overhead` · `Metaspace` | — |
| THREADPOOL | `RejectedExecutionException` · `Thread pool exhausted` · `threads.*busy` · `Too many open files` | — |
| JWT | `JwtException` · `invalid_token` · `Jwk .* not found` · `JWKS` · `expired` · `issuer` | issuer |
| CODE_BUG | `NullPointerException` · `ClassCastException` · `NoSuchMethodError` · `ClassNotFoundException` · `IllegalArgumentException` | class |

**3.3 Secret/config sweep W1**
```json
{"size":10,"sort":[{"@timestamp":"desc"}],"_source":["@timestamp","{pod}","{msg}"],
 "query":{"bool":{"filter":[RANGE(W1),NSF,APPF,NOSC,
  {"query_string":{"query":"secret OR KeyVault OR keyvault OR Vault OR credential OR \"Could not resolve placeholder\" OR Unauthorized OR Forbidden OR FailedMount OR \"environment variable\"","fields":["{msg}"]}}]}}}
```
→ `app.secret_lines[]`, extracted `secret_names[]` (quoted names after `placeholder`, `secret`, `KeyVault`). Non-empty → queue 7.3.

**3.4 Outbound deps W1**
```json
{"size":0,"query":{"bool":{"filter":[RANGE(W1),NSF,APPF,SC,{"prefix":{"upstream_cluster":"outbound"}}]}},
 "aggs":{"dep":{"terms":{"field":"authority.keyword","size":20,"missing":"(tcp/sni)"},"aggs":{
   "sni":{"terms":{"field":"requested_server_name.keyword","size":2}},
   "cluster":{"terms":{"field":"upstream_cluster.keyword","size":2}},
   "codes":{"terms":{"field":"response_code","size":6}},
   "flags":{"terms":{"field":"response_flags.keyword","size":6}},
   "p99":{"percentiles":{"field":"duration","percents":[50,99]}}}}}}
```
→ `deps[]{host, internal(bool: contains .svc.cluster.local), cluster, codes, flags, p99}`. Gate (compute): any dep with `(5xx + UF + UT + NR + UH)/total ≥ 0.05` → `flags.dep_failing=[hosts]` (max 3). `PassthroughCluster`/`BlackHoleCluster` → `flags.egress_registry=[hosts]`. If `catalog.source=derived`, set `catalog.deps=deps.hosts`.

**3.5 Auth-401 shape W0** (only if `flags.envoy_generated_auth`)
```json
{"size":0,"query":{"bool":{"filter":[RANGE(W0),NSF,APPF,SC,{"terms":{"response_code":[401,403]}},{"term":{"upstream_host.keyword":"-"}}]}},
 "aggs":{"path":{"terms":{"field":"path.keyword","size":15}},"authority":{"terms":{"field":"authority.keyword","size":5}},
  "ua":{"terms":{"field":"user_agent.keyword","size":5}},"xff":{"terms":{"field":"x_forwarded_for.keyword","size":5}},
  "tl":{"date_histogram":{"field":"@timestamp","fixed_interval":"1m"}}}}
```
→ `auth401{shape: all_paths|one_path|one_client, start}`.

## Phase 4 — Dependency drill (≤ 8 ES calls; triggered by Phase 3 classes/flags; max 3 hosts)

**4.1 Internal dep failing** (`flags.dep_failing` host ends `.svc.cluster.local`): resolve `dep.ns/dep.app` from host; run **1.1, 2.3, 3.1** for the dep with W0. Depth 1 only. → `dep_drill[host]`.

**4.2 External dep failing** (host not internal, or `flags.egress_registry`):
```json
{"size":0,"query":{"bool":{"filter":[RANGE(W0),SC,{"prefix":{"upstream_cluster":"outbound"}},
  {"bool":{"should":[{"term":{"authority.keyword":"HOST"}},{"term":{"requested_server_name.keyword":"HOST"}}],"minimum_should_match":1}}]}},
 "aggs":{"scope":{"terms":{"field":"{ns}","size":20}},"flags":{"terms":{"field":"response_flags.keyword","size":8}},
  "tls":{"terms":{"field":"upstream_transport_failure_reason.keyword","size":5}},
  "cluster":{"terms":{"field":"upstream_cluster.keyword","size":3}},"p99":{"percentiles":{"field":"duration","percents":[99]}}}}
```
Plus same host on egress gateway (`GWF` with `istio-egressgateway`). → `egress[host]{scope_ns[], flags, tls_reason, cluster, p99, via_egressgw}`. Compute: `UF && p99 ≈ 10000 && !tls_reason` → `shape:"tcp_blocked"`; `tls_reason` → `shape:"tls"`; `BlackHoleCluster` → `shape:"no_serviceentry"`.

**4.3 KAFKA_\*** (any kafka class):
a) config: `size:3` app logs W30(async) `{msg}:("ConsumerConfig values" OR "ProducerConfig values")` sort desc → extract `bootstrap.servers, group.id, max.poll.interval.ms, session.timeout.ms, security.protocol`.
b) consume rate W1: `date_histogram 5m, min_doc_count 0, extended_bounds` with filter `{msg}:(Received OR Consumed OR Processing OR offset)` on app logs → `kafka.consume_rate[]`; compute `flatline_since`.
c) blast radius W0 all ns: `NOSC AND {level}:(ERROR OR WARN) AND {msg}:"<bootstrap_host>"` → terms `{ns},{app}` → `kafka.blast_radius[]`.
→ `kafka{config, consume_rate, flatline_since, blast_radius}`.

**4.4 DB_\* / CASSANDRA**:
a) config: app logs W30(async) `size:3` `{msg}:("HikariConfig" OR "maximumPoolSize" OR "jdbcUrl" OR "contact-points")` → extract host (strip creds), `maximumPoolSize`, `connectionTimeout`.
b) pool stats W1: `{msg}:"Pool stats"` size 20 sort desc → `db.pool_stats[]{ts,total,active,idle,waiting}`.
c) alignment W0: the DB class filter → `date_histogram 1s` across `per_pod` → compute `aligned_at` (≥ 3 pods same second).
d) blast radius W0 all ns: `NOSC AND {level}:ERROR AND {msg}:"<db_host>"` → terms `{ns},{app}`.
→ `db{config, pool_stats, aligned_at, blast_radius}`.

**4.5 TLS** (class TLS or `flags.lua_or_auth_or_tls` with TLS text): W1, `{terms {ns}:[NS, istio-system]}`, query_string of TLS terms (certs skill Step 1) → `by_app/by_container`, `tl 1m`, `first_seen`, samples 5 → `tls{layer, hosts[], first_seen}`. If `hopmap` has cert-manager index: `{app}:cert-manager* AND Certificate AND (failed OR error OR renew)` size 5.

**4.6 DNS** (class DNS): app logs W1 `UnknownHostException OR EAI_AGAIN OR NXDOMAIN` → terms extracted host (scripted or `significant_terms`), `per_pod` → `dns{hosts[], per_pod}`.

## Phase 5 — Edge (≤ 5 ES calls; only hops present in `hopmap`)

**5.1 Anchor** — if `ticket.ids.akamai`: `akamai: {"term":{"reqId":"ID"}}` size 1. If `ticket.ids.apigee`: `apigee: {"term":{"messageid":"ID"}}` size 1. → `edge.anchor{hop, doc}`; adopt its `x-request-id` for 2.4 if 2.4 had none.

**5.2 Funnel W0** — per existing hop, same host+path filter (fields per hop from `request-origin-trace` table):
```json
{"size":0,"query":{"bool":{"filter":[RANGE(W0),{"wildcard":{"<host_field>":"*HOST*"}},{"wildcard":{"<path_field>":"*PATH*"}}]}},
 "aggs":{"status":{"terms":{"field":"<status_field>","size":10}},
  "tl":{"date_histogram":{"field":"@timestamp","fixed_interval":"1m"},"aggs":{"err":{"filter":{"range":{"<status_field>":{"gte":400}}}}}},
  "fault":{"terms":{"field":"<fault_or_errorcode_field>","size":5}},
  "samples":{"top_hits":{"size":3,"_source":["@timestamp","<id_fields>","<status_field>","<client_ip>","<target>"]}}}}
```
→ `edge.funnel{akamai, apigee, f5, gw}`. Compute `edge.first_failing_hop` = first hop in chain order with ≥400 while previous hop shows 200/absent-by-design.

**5.3 Originator W0** at GW inbound (or sidecar inbound if GW blind):
```json
"aggs":{"xff":{"terms":{"field":"x_forwarded_for.keyword","size":20}},"ua":{"terms":{"field":"user_agent.keyword","size":10}},
 "routing_key":{"terms":{"field":"<ROUTING_KEY_FIELD>","size":10,"missing":"(none)"}},"apigee_app":{"terms":{"field":"<APIGEE_APP_FIELD>","size":10,"missing":"(none)"}}}
```
→ `edge.originator{external_via_edge:bool, internal_callers[], ua_class}`. If internal CIDR and no Apigee headers → one extra call: sidecar outbound all-ns `upstream_cluster:*SVC.NS.svc*` → terms `{ns},{app}` → `internal_callers`.

## Phase 6 — Baseline (≤ 3 async calls on `tiers.indices_w30`)

**6.1** top class from 3.2 (or `gw.errors.code_flag[0]` if no app logs): same filter, `date_histogram 3h` W30 → `baseline.signature{first_seen, daily[], new:bool}` (`new` = zero buckets for ≥ 7 days before `first_seen`).
**6.2** gateway 5xx: 2.1 filter, `date_histogram 3h` W30 → `baseline.gw_5xx[]`.
**6.3** if class ∈ {TLS, JWT, KAFKA_AUTH, DB_AUTH, SECRET_AUTH} or `flags.envoy_generated_auth`: `date_histogram 1h` W30 then `1m` over the first non-zero hour → `baseline.onset{minute, gradual:bool}`.

## Phase 7 — GitLab (≤ 6 calls; only queued items)

**7.1** (`flags.envoy_generated_auth` or `flags.lua_or_auth_or_tls`): search `kind: EnvoyFilter` AND (`APP` OR `GW_APP`) in `catalog.repo` and the mesh/platform repo → for the best file: commits on path (last ≤ T → `sha`, previous → `prev_sha`), content at both; also `RequestAuthentication`/`AuthorizationPolicy` for NS → `git.envoyfilter{name, path, sha, content, prev_sha, prev_content, commits_24h[]}`.
**7.2** (always if repo known, 1 call): `values.yaml` for APP/ENV at last commit ≤ T; extract `ENABLE_FILE_BEAT`, `image.tag`, `env[]`, `secretProviderClass`, `resources`, `replicas`; commits on path [T-7d, T] → `git.values`.
**7.3** (`secret_names[]`): search each name in `catalog.repo` + centralized config repo → `git.secrets[]{name, source: literal|configmap|k8s_secret|keyvault_csi|vault, path, last_change{sha, ts, author}}`.
**7.4** (`flags.deploy_before_incident`): compare `prev_image_tag..image_tag` (tag→sha via pipeline or tag ref) → files changed summary, MR title → `git.deploy_diff`.
**7.5** (`flags.dep_failing` internal): dep's `values.yaml` `image.tag` + last commit → `git.dep_values`.

## Phase 8 — Pre-verdict (no tool). Compute and record `hints`:

```
failing_hop        = majority(traces[].last_hop) ‖ edge.first_failing_hop ‖ (gw.errors.code_flag[0] → envoy skill table) ‖ "unknown"
timeline[]         = sort([deploys.first_seen, k8s_events.ts, flags.gw_error_start, baseline.onset.minute, git.*.commits.ts, tls.first_seen, shipping.off_since, kafka.flatline_since, db.aligned_at])
candidate_classes  = app.classes sorted by count desc, then first_seen asc  (top 3)
deploy_correlated  = |gw_error_start − deploy.first_seen| < 30m  ‖ |first error − git commit| < 30m
exact_minute_onset = baseline.onset.gradual == false
blast_radius       = kafka.blast_radius ∪ db.blast_radius ∪ egress[*].scope_ns
confidence_cap     = "medium" if any(intake.known_level != A, app_logs_not_shipped, blind hop on path, catalog.source=="derived", async_timeout, envoy_parsed==false) else "high"
skills_to_load     = by failing_hop/classes: gateway|sidecar→envoy-istio-gateway; app+CONFIG/SECRET/CODE/OOM/THREADPOOL→springboot-dataapi; DB_*/CASSANDRA→database; KAFKA_*→kafka; TLS/JWT→certs-tls; DNS/egress→egress; envoy_generated_auth→envoyfilter-lua-auth; edge hop→request-origin-trace (decoder table only)
```

Then: load `skills_to_load` (max 3) + orchestrator Step C2/D, and reason **only over the ledger**. Every claim cites a ledger key (`traces[0].hops[3]`, `git.envoyfilter.content:L42`). If the ledger is insufficient, name the exact step id to re-run with a wider window — max 2 extra steps — then conclude with gaps.

---

## EVIDENCE ledger format (append after every phase; this is your state)

```
EVIDENCE v<n> | INC <id> | T=<utc> | NS/APP=<..> | calls_used=<es>/<git>
flags: {...}
P0 catalog:{source,svc,deps[..3],gw} fieldmap:{envoy_parsed} hopmap:{akamai,apigee,f5,k8s_events} tiers:{w30_indices:n}
P1 pods:[{name,first,last,img,app_docs,sidecar_docs}] deploys:[..] k8s_events:[reason:count] startups:{crashloop,failed_to_start} shipping:{off_since} readiness:[..]
P2 gw.errors:{code_flag[..5],first,last,paths[..3]} gw_error_start:M sidecar.inbound:{code_flag,no_upstream} traces:[{id,last_hop,hops:n}] proxy_log:[..3] istiod:[..3]
P3 app.errors:{type[..5],endpoint[..3],per_pod_skew} classes:[{class,count,first_seen,extracted}] secret_names:[..] deps:[{host,codes,flags,p99}] auth401:{shape,start}
P4 dep_drill:{..} egress:{host:{shape,scope}} kafka:{config,flatline_since,blast} db:{config,pool_stats_last,aligned_at,blast} tls:{layer,hosts,first_seen} dns:{hosts}
P5 edge.anchor:{hop} funnel:{akamai:st,apigee:st/fault,f5:st,gw:st} first_failing_hop:X originator:{..}
P6 baseline:{signature:{first_seen,new},onset:{minute,gradual}}
P7 git:{envoyfilter:{name,sha,prev_sha,commits_24h:n}, values:{ENABLE_FILE_BEAT,image.tag,commits:n}, secrets:[{name,source,last_change}], deploy_diff:{files:n,mr}}
P8 hints:{failing_hop, timeline[..6], candidate_classes[..3], deploy_correlated, exact_minute_onset, confidence_cap, skills_to_load}
gaps: [...]
samples: {key: [≤3 lines, msg[:200]]}   ← only the samples the hints reference
```

Keep the ledger ≤ 4 KB per phase block; drop raw aggregation JSON once summarized. The ledger, not the tool outputs, is what you reason from in the final step.
