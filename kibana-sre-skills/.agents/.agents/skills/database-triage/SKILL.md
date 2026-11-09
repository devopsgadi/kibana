---
name: database-triage
description: Diagnose database-side failures seen from application logs in Kibana — JDBC (Oracle, SQL Server, PostgreSQL, MySQL), HikariCP pool exhaustion, Cassandra driver errors (NoHostAvailable, timeouts, consistency), connection refused/reset, deadlocks, slow queries, schema/migration (Flyway/Liquibase) problems, credential rotation. Use when the failing hop is a datastore, or for "pool exhausted", "connection refused to db", SQL/ORA/Cassandra errors.
---

# Database Triage (from app logs in Kibana)

Prereq: `kibana-query-strategy`. You see the **client side** only: driver exceptions, pool stats, timings. DB server logs are not in Elastic unless the catalog says so. State the boundary.

```
APP_LOGS = {ns}:"NS" AND {app}:"APP" AND NOT {container}:"istio-proxy"
DB_LOGS  = APP_LOGS AND (logger_name:(*Hikari* OR *jdbc* OR *hibernate* OR *datastax* OR *cassandra* OR *Flyway* OR *liquibase* OR *jpa*) 
                         OR {msg}:(*SQL* OR *ORA-* OR *Hikari* OR *Cassandra* OR *NoHostAvailable* OR *deadlock* OR *JDBC*))
```

## Step 1 — Harvest connection config from boot logs

```
APP_LOGS AND {msg}:("HikariPool-1 - Starting" OR "HikariConfig" OR "maximumPoolSize" OR "jdbcUrl" OR "connectionTimeout" OR "contact-points" OR "Cassandra driver" OR "Flyway" OR "Liquibase")
```
Capture: `jdbcUrl` host (never log creds), `maximumPoolSize`, `connectionTimeout`, `maxLifetime`, Cassandra `contact-points`/`local-datacenter`/`consistency`. Needed to interpret Step 3.

## Step 2 — Signature aggregation

```json
{"size":0,"query":{"bool":{"filter":[RANGE,NS,APP,NOT_SIDECAR,
  {"query_string":{"query":"SQL OR ORA OR Hikari OR JDBC OR Cassandra OR NoHostAvailable OR deadlock OR Flyway OR Liquibase OR \"connection refused\"","fields":["{msg}","logger_name","error.type"]}}]}},
 "aggs":{"type":{"terms":{"field":"error.type.keyword","size":15,"missing":"(none)"}},
         "sqlstate":{"terms":{"script":{"source":"def m=doc['{msg}.keyword'].value; def i=m.indexOf('SQLSTATE'); def j=m.indexOf('ORA-'); return i>=0? m.substring(i,Math.min(i+16,m.length())) : (j>=0? m.substring(j,Math.min(j+9,m.length())) : 'n/a');"},"size":10}},
         "per_pod":{"terms":{"field":"{pod}","size":30}},
         "timeline":{"date_histogram":{"field":"@timestamp","fixed_interval":"1m"}},
         "samples":{"top_hits":{"size":3,"_source":["@timestamp","{pod}","traceId","{msg}"]}}}}
```
Scripted aggs disabled → `terms` on `error.type` and wildcard filters per code.

## Step 3 — Signature decoder

### Pool / connectivity (any RDBMS)

| Signature | Meaning | Confirm |
|---|---|---|
| `HikariPool-1 - Connection is not available, request timed out after Nms` | Pool exhausted: all `maximumPoolSize` busy for `connectionTimeout` | pool stats lines (`HikariPool-1 - Pool stats (total=, active=, idle=, waiting=)`) if debug on; `active == max`, `waiting > 0`. Cause = slow queries (Step 4) or leak (active stays at max after traffic drops) |
| `HikariPool-1 - Failed to validate connection ... (The connection is closed)` / `Connection reset` | DB/firewall idle-killed connections; `maxLifetime` > firewall idle timeout, or DB failover | burst aligned across all pods at same second → server-side event |
| `Connection refused` / `IO Error: The Network Adapter could not establish the connection` (ORA-17002) | DB listener down, wrong host/port, network policy | all pods, all apps on same DB host (Step 5) |
| `UnknownHostException: db.host` | DNS | `egress-triage` |
| `ORA-12514 listener does not currently know of service` | Wrong service name / DB failover to node without service | first-seen; DB maintenance window |
| `ORA-01017 invalid username/password` / `Login failed for user` (SQL Server 18456) / `password authentication failed` (PG 28P01) | **Credential rotation** without redeploy | first-seen vs secret change; springboot secret sweep |
| `ORA-28001 password has expired` | Expired DB account | date-driven; 30d baseline shows first failure at expiry |
| `ORA-00020 maximum number of processes` / `too many connections` (PG 53300) / `FATAL: remaining connection slots are reserved` | DB-side connection cap hit — sum of `maximumPoolSize × replicas` across apps too high | Step 5 fan-out; recent scale-up or new app |
| `ORA-01555 snapshot too old` / `ORA-01652 unable to extend temp` / `ORA-01653` | Server capacity (undo/temp/tablespace) | DBA boundary |
| `ORA-00060 deadlock detected` / `Deadlock found` (MySQL 1213) / `deadlock detected` (PG 40P01) / SQL Server 1205 | Application lock ordering; often one endpoint | `url.path` × signature |
| `Lock wait timeout exceeded` / `ORA-00054 resource busy` / `could not obtain lock` | Long transaction / migration holding lock | Flyway/Liquibase lines in window |
| `SQLTimeoutException` / `QueryTimeoutException` / `ORA-01013` | Statement > `queryTimeout`; slow query or DB under load | Step 4 |
| `ORA-03113 end-of-file` / `ORA-03135 connection lost contact` / `ORA-12170 TNS:Connect timeout` | DB node died / failover / network | all pods same second; DB RAC failover |
| `PSQLException: FATAL: the database system is in recovery mode` / `starting up` | PG restart/failover | timeline |
| `read-only transaction` / `cannot execute INSERT in a read-only transaction` | Writes hitting a replica after failover | first-seen |
| `Flyway ... Migration checksum mismatch` / `Validate failed` / `Detected failed migration` / `Liquibase ... lock` | Deploy blocked at schema step — pods crash-loop before serving | springboot startup; image tag change |

### Cassandra (DataStax driver)

| Signature | Meaning |
|---|---|
| `NoHostAvailableException` / `AllNodesFailedException` | No contact point reachable: nodes down, wrong DC in `local-datacenter`, network/egress, auth |
| `ReadTimeoutException ... Cassandra timeout during read query at consistency LOCAL_QUORUM (2 responses were required but only 1 replica responded)` | Replica down or overloaded; consistency not achievable |
| `WriteTimeoutException` / `UnavailableException: Not enough replicas available for query at consistency X (Y required but only Z alive)` | Nodes down below RF needed |
| `OperationTimedOutException` / `DriverTimeoutException` | Client-side timeout; coordinator slow (compaction, GC, large partition) |
| `OverloadedException` | Node shedding load |
| `AuthenticationException` / `Provided username ... incorrect` | Credential rotation |
| `Tombstone` warnings / `Read N live rows and M tombstone cells` | Data model issue → read latency |
| `InvalidQueryException: unconfigured table` / `Keyspace X does not exist` | Schema not deployed in env |
| `BusyPoolException` / `BusyConnectionException` | Driver pool saturated; too many in-flight |
| `ConnectionInitException` + TLS text | → `certs-tls-triage` |

## Step 4 — Latency / slow queries (client side)

If the app logs statement timings (`p6spy`, `hibernate.generate_statistics`, `took=`, `executionTime`):
```json
"aggs":{"by_stmt":{"terms":{"field":"sql.statement_prefix.keyword","size":10},"aggs":{"p99":{"percentiles":{"field":"sql.duration_ms","percents":[50,99]}}}}}
```
Otherwise infer: Hikari pool exhaustion + endpoint-level p99 spike (envoy inbound `upstream_service_time`) + no DB errors = **slow queries / DB load**. Say "inferred — no statement timings in logs".

## Step 5 — Blast radius (same DB host across apps)

```
NOT {container}:"istio-proxy" AND {msg}:"DB_HOST_OR_SERVICE" AND {level}:(ERROR OR WARN)   # over all ns
→ terms on {ns}, {app}; timeline
```
Many apps at the same second → DB/server/network event. One app → that app's pool, query, or creds.

Also sum pool capacity if `ORA-00020`/`too many connections`: Σ(`maximumPoolSize` × running pods) per app from Step 1 + pod lifecycle counts.

## Step 6 — Correlate

- First-seen vs **image tag change** (new query/ORM mapping, Flyway migration) → pod lifecycle agg
- First-seen vs **secret rotation** → springboot secret sweep for `Forbidden`/`Unauthorized`/`KeyVault` in T-24h
- Burst alignment across pods/apps at the same second → server-side (failover/restart/network)
- Recurring daily at same hour over 30d → batch/maintenance job contention

## Step 7 — 30-day baseline

Async `3h` histogram for the chosen signature. Report first-seen, chronic rate, and periodicity.

## Output

```
DB verdict: <pool exhausted (slow queries|leak) | DB unreachable (<host>) | failover @T | credentials | deadlock on <endpoint> | schema/migration | Cassandra: <exception>@<consistency> | capacity (ORA-xxxx)>
Config seen: host=<...> maxPool=<n> connTimeout=<ms> | contact-points=<...> dc=<...>
Blast radius: <apps/pods affected; aligned @ T?>
Evidence: <query> → <ts | pod | traceId | msg[:200]> (≤3)
Gaps: <no DB server logs | no statement timings | pool stats debug off>
Read-only next checks (DBA/with access): v$session count by machine; pg_stat_activity; nodetool status; listener status
```
