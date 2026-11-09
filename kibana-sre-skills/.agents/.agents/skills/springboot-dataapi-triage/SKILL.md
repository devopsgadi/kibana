---
name: springboot-dataapi-triage
description: Diagnose Spring Boot micro-apps and the DataAPI layer from Kibana — startup failures (APPLICATION FAILED TO START, BeanCreationException, missing secret/placeholder), crash loops, OOM, thread/connection pool exhaustion, exception fingerprinting, 5xx returned by the app, slow endpoints, Feign/RestTemplate/WebClient dependency failures, Actuator health. Use when the failing hop is the app container, or for "app down", "500 from service", "not starting", "config missing".
---

# Spring Boot / DataAPI Triage

Prereq: `kibana-query-strategy`. `APP_LOGS = {ns}:"NS" AND {app}:"APP" AND NOT {container}:"istio-proxy"`.

Logback/Log4j JSON fields commonly present: `logger_name`, `thread_name`, `level`, `stack_trace`, `exception` / `error.type`, `error.message`, `error.stack_trace`, `traceId`, `spanId`, `http.request.method`, `http.response.status_code`, `url.path`, `application` / `spring.application.name`. Confirm in Step 0 of query-strategy.

## Step 0 — Are app logs shipped at all?

If `APP_LOGS` returns 0 docs while istio-proxy has docs, run the "shipper off vs app dead" check in `kibana-query-strategy` first. `ENABLE_FILE_BEAT=false` → skip Steps 1–5 here, run Step 6 (actuator via sidecar) and Step 7 (pod lifecycle) only, and report the gap.

## Step 1 — Did it start? Is it alive?

Startup markers, tight window + T-24h:
```
APP_LOGS AND {msg}:("Starting " OR "Started " OR "APPLICATION FAILED TO START" OR "Tomcat started on port" OR "Netty started on port" OR "Shutdown initiated" OR "Closing JPA" OR "Stopping service")
→ terms on {pod}; per pod: min/max @timestamp, count of "Started", count of "FAILED TO START"
```
- Many pods with `Starting` but never `Started` → **crash loop**; read lines between them (Step 2).
- Pods start fine then stop logging → OOMKill / liveness kill; check k8s events index (`OOMKilling`, `Unhealthy`, `BackOff`) if shipped.
- `Started` present, no errors, but Envoy `UF` → **port mismatch** (`server.port` vs Service targetPort) — compare `started on port N` with catalog.

## Step 2 — Startup failure decoder (read ~80 lines before `APPLICATION FAILED TO START`)

| Signature | Cause |
|---|---|
| `Could not resolve placeholder 'X' in value "${X}"` | ConfigMap/Secret key `X` missing in this env (common after config-centralization moves) |
| `BeanCreationException ... Injection of autowired dependencies failed` → look at the *last* `Caused by:` | Downstream of a missing property/bean; the last `Caused by` is the truth |
| `UnsatisfiedDependencyException` | Bean wiring; profile mismatch (`spring.profiles.active`) |
| `Failed to configure a DataSource: 'url' attribute is not specified` | DB config missing → `database-triage` |
| `Connection refused` / `UnknownHostException` during start (Config Server, Vault, Key Vault, Eureka) | Dependency unreachable at boot → `egress-triage` / `certs-tls-triage` |
| `AccessDenied` / `403 Forbidden` from `KeyVault`/`Vault`/`Config Server` | Identity/secret rotation → check first-seen vs deploy |
| `Web server failed to start. Port 8080 was already in use` | Sidecar/init ordering or duplicate process |
| `Liquibase`/`Flyway ... Migration checksum mismatch` / `lock` | Schema migration blocked → `database-triage` |
| `java.lang.OutOfMemoryError` / `Metaspace` | Heap too small for new build; correlate image tag |
| `ClassNotFoundException` / `NoSuchMethodError` | Bad build / dependency conflict — image tag correlation decisive |
| `Keystore was tampered with, or password was incorrect` / `PKIX path building failed` | Keystore/truststore secret → `certs-tls-triage` |
| `Profile "X" ... No active profile set` | Wrong/missing `SPRING_PROFILES_ACTIVE` in deployment |

Secret/config sweep (also catches runtime secret rotation):
```
APP_LOGS AND {msg}:(*secret* OR *KeyVault* OR *keyvault* OR *Vault* OR *credential* OR *"Could not resolve placeholder"* OR *BeanCreationException* OR *"APPLICATION FAILED TO START"* OR *Unauthorized* OR *Forbidden* OR *"environment variable"*)
```

## Step 3 — Runtime error fingerprinting (when app runs but fails requests)

```json
{"size":0,"query":{"bool":{"filter":[RANGE,NS,APP,NOT_SIDECAR,
   {"terms":{"{level}":["ERROR","FATAL","WARN"]}}]}},
 "aggs":{
  "type":{"terms":{"field":"error.type.keyword","size":15,"missing":"(none)"}},
  "logger":{"terms":{"field":"logger_name.keyword","size":15}},
  "endpoint":{"terms":{"field":"url.path.keyword","size":10}},
  "status":{"terms":{"field":"http.response.status_code","size":6}},
  "per_pod":{"terms":{"field":"{pod}","size":30}},
  "timeline":{"date_histogram":{"field":"@timestamp","fixed_interval":"1m"}},
  "samples":{"top_hits":{"size":3,"_source":["@timestamp","{pod}","traceId","error.type","{msg}"]}}}}
```
No structured `error.type`? Use the 120-char message prefix scripted agg from query-strategy.

Per-pod skew (one pod has 95% of errors) → bad node / stale pod / sticky session → pod lifecycle.

## Step 4 — Runtime signature decoder

| Signature | Cause | Hand-off |
|---|---|---|
| `HikariPool-1 - Connection is not available, request timed out after 30000ms` | DB pool exhausted — slow queries or leak; check `maximumPoolSize` in boot logs | `database-triage` |
| `CannotGetJdbcConnectionException` / `Connection refused` / `Communications link failure` | DB unreachable | `database-triage` |
| `JpaSystemException` / `DataIntegrityViolationException` / `ORA-` / `SQLSTATE` | Query/schema; data | `database-triage` |
| `feign.RetryableException: Read timed out executing GET http://X` | Dependency X slow/down | envoy Step 4 on X; recurse |
| `WebClientResponseException$ServiceUnavailable` / `HttpServerErrorException: 503` | Dependency returned 5xx | recurse on dependency |
| `ResourceAccessException: I/O error ... Connection reset` | Dependency connection dropped (idle timeout, pod kill) | envoy outbound flags `UC`/`UR` |
| `CircuitBreaker 'X' is OPEN` (Resilience4j) / `HystrixRuntimeException` | Breaker tripped — dependency has been failing; find first failure | recurse |
| `RejectedExecutionException` / `Thread pool exhausted` / `tomcat.threads ... busy` | Tomcat/executor saturated — upstream slow or traffic spike | envoy latency Step 5; check RPS in gateway logs |
| `Too many open files` | FD leak / connection leak | pod lifecycle, first-seen vs deploy |
| `java.lang.OutOfMemoryError: Java heap space` / GC overhead | Memory; often appears right before pod disappears | pod lifecycle |
| `AsyncRequestTimeoutException` / `Request processing failed ... timeout` | App-side timeout shorter than downstream latency | envoy Step 5 |
| `JwtException` / `invalid_token` / `Jwk ... not found` / `401` burst | Token/OIDC issuer change or JWKS fetch failed | `egress-triage` (JWKS host) / `certs-tls-triage` |
| `Kafka` loggers | — | `kafka-triage` |
| `SSLHandshakeException` / `certificate_unknown` / `handshake_failure` | TLS to dependency | `certs-tls-triage` |
| `UnknownHostException` | DNS / egress / ServiceEntry | `egress-triage` |
| `NullPointerException` / `ClassCastException` burst after deploy | Code bug — bad deploy | image tag correlation; recommend rollback review |

## Step 5 — DataAPI specifics

DataAPI apps are a thin facade over DB/Mainframe (Hogan) and are called by many micro-apps. For a DataAPI incident:

1. **Who is calling it and failing?** From its *inbound* sidecar logs: `terms` on `downstream_remote_address`/`x_forwarded_for` and `path` with `response_code ≥ 500` → callers + endpoints.
2. **What is it calling?** Outbound sidecar `authority` + app logs for JDBC/Cassandra/MQ/Mainframe connector exceptions (`CICS`, `MQ`, `IMS`, `Hogan`, `CTG`, `ECI` keywords if present).
3. **Latency decomposition**: envoy `duration` vs `upstream_service_time` on inbound; app-side `took=` / `elapsed` log fields if present; DB latency from `database-triage`.
4. **Pagination/size blowups**: look for `maxResultSize`, `Page size`, `OutOfMemory` right after a specific `url.path`.

Mainframe/Hogan dependencies cannot be seen beyond the connector exception; state the boundary explicitly.

## Step 6 — Actuator/health

```
SIDECAR AND path:(/actuator/health* OR /health* OR /ready*) AND NOT response_code:200
→ timeline per pod; first non-200
```
Readiness flapping explains `UH` at the gateway even when the app process is up.

## Step 7 — Deploy correlation (always)

Pod lifecycle agg (query-strategy). New image tag within 30m before first error → **probable bad deploy**; emit `old → new @ T`. Also check for `spring.profiles.active` or config hash change lines in startup logs between the two tags.

## Step 8 — 30-day baseline

Async `3h` histogram of the top `error.type`. Report new vs chronic and the first-seen tag.

## Output

```
App verdict: <crash-loop: <cause> | runtime: <error.type> on <endpoint> | pool exhaustion | dependency <X> | bad deploy A→B @ T | healthy>
Startup: <Started OK on port N | FAILED: <last Caused by>>
Top errors: <type: count (pods n/N)>
Hand-offs: <database-triage | kafka-triage | certs-tls-triage | egress-triage | recurse on <dep>>
Evidence: <query> → <ts | pod | traceId | msg[:200]> (≤3)
Gaps: <no structured error.type | no k8s events index | mainframe boundary>
Read-only next checks: kubectl describe pod; kubectl get cm/secret -n NS (names only); actuator/env (with access)
```
