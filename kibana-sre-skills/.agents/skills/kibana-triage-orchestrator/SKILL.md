---
name: kibana-triage-orchestrator
description: Entry point for SRE incident triage using only Kibana/Elasticsearch logs via MCP. Classifies the symptom, picks domain skills (envoy/istio/gateway, kafka, springboot/dataapi, database, certs, egress), runs them in the right order, correlates by x-request-id and time, and produces a root-cause report with evidence. Use for any ServiceNow incident, alert, or "why is app X failing" question.
---

# Kibana Triage Orchestrator

You have **no cluster access**. Everything comes from Elasticsearch through the Kibana/ES MCP. Diagnosis only — never propose mutating actions.

## Load order

1. `kibana-query-strategy` — always. Gives you field discovery, time pinning, 30-day tiering, async search.
2. Service-catalog lookup (your MCP/catalog tool) — resolve `APP` → namespace, Service name, upstream dependencies, Kafka topics, DB, egress hosts, owning team. If the catalog is unreachable, derive from logs (`upstream_cluster` aggregation) and flag it.
3. Domain skills, chosen by the decision table below.

## Inputs

| Input | Source | Fallback |
|---|---|---|
| `INCIDENT_TIME` | ticket / alert | ask; otherwise use ticket `opened_at - 30m` |
| `APP`, `NS` | ticket CI field / catalog | grep ticket text for `*-svc`, `*-api`, hostnames |
| `SYMPTOM` | ticket short description | classify from first error-level log burst |
| `REQUEST_ID` / `traceId` | ticket, user screenshot | none — find one in Step 2 |
| `ENV` | ticket (IT/UAT/pre-prod/EN/EMP/PROD) | infer from index pattern or namespace suffix |

## Decision table — symptom → skills, in order

| Symptom text / signal | Run first | Then | Finish with |
|---|---|---|---|
| "gateway error", 502/503/504, "bad gateway", Akamai/F5 5xx | `envoy-istio-gateway-triage` | whichever hop failed: springboot / database / egress | pod lifecycle |
| 404 at edge, "route not found", new endpoint not reachable | `envoy-istio-gateway-triage` (NR flag path) | — | pod lifecycle |
| "slow", timeouts, 504 UT, p99 spike | `envoy-istio-gateway-triage` (duration aggs) | `database-triage`, `kafka-triage` | baseline 30d |
| "consumer lag", "messages not processed", rebalance, Kafka exception | `kafka-triage` | `springboot-dataapi-triage` | baseline 30d |
| "connection refused", "pool exhausted", SQL/Cassandra errors, `HikariPool` | `database-triage` | `springboot-dataapi-triage` | pod lifecycle |
| "certificate", TLS/SSL handshake, `x509`, `UAEX`, expired, SNI | `certs-tls-triage` | `egress-triage` if outbound | baseline 30d (cert expiry is date-driven) |
| "cannot reach <external host>", DNS, egress 403/404, ServiceEntry | `egress-triage` | `certs-tls-triage` | — |
| App crash-loop, "APPLICATION FAILED TO START", secret/config missing | `springboot-dataapi-triage` | — | pod lifecycle + image tag change |
| Unknown / vague ("app is down") | Step A below | route by what dominates | all |

## Step A — Triage opening (unknown symptom)

One async aggregation over `WINDOW_WIDE` for the app:

```json
{"size":0,"query":{"bool":{"filter":[
   {"range":{"@timestamp":{"gte":"T-2h","lte":"T+2h"}}},
   {"term":{"{ns}":"NS"}},{"term":{"{app}":"APP"}}]}},
 "aggs":{
  "by_container":{"terms":{"field":"{container}","size":5}},
  "levels":{"terms":{"field":"{level}","size":6}},
  "envoy_codes":{"terms":{"field":"response_code","size":10}},
  "envoy_flags":{"terms":{"field":"response_flags","size":10}},
  "timeline":{"date_histogram":{"field":"@timestamp","fixed_interval":"5m","min_doc_count":1},
     "aggs":{"err":{"filter":{"terms":{"{level}":["ERROR","FATAL"]}}},
             "five_xx":{"filter":{"range":{"response_code":{"gte":500}}}}}},
  "pods":{"terms":{"field":"{pod}","size":30},"aggs":{"first":{"min":{"field":"@timestamp"}},"last":{"max":{"field":"@timestamp"}},"image":{"terms":{"field":"{image}","size":2}}}}
 }}
```

Read it like this:
- `envoy_flags` dominated by `UF/UC/URX` → sidecar can't reach app → `springboot-dataapi-triage` (crash) 
- `UH` → no endpoints → pod lifecycle (all pods dead / not ready)
- `UAEX` / `response_code: 0` with TLS text → `certs-tls-triage`
- `NR` → routing → `envoy-istio-gateway-triage`
- App `ERROR` burst but Envoy 200s → business-logic / dependency → `springboot-dataapi-triage` then `database-triage` / `kafka-triage`
- Pods `last` timestamp inside window + new `image` tag → bad deploy, confirm with Step 6 of envoy skill
- Zero docs → sanity checks in `kibana-query-strategy` (shipper down? wrong label? retention?)

## Step B — Pick one request and walk the chain

Find a failing `x_request_id` from the gateway/sidecar query, then trace it with no label filter (see query-strategy "Request-ID correlation"). The **last hop that logged** owns the failure. Hand that hop's app to the relevant domain skill with the same request id.

Chain shape for your platform:
```
Akamai → F5/NGINX Plus → istio-ingressgateway → app sidecar (inbound) → Spring Boot app
                                                   └→ app sidecar (outbound) → dataapi / other ns svc / egressgateway → external
                                                   └→ Kafka client → brokers
                                                   └→ JDBC / Cassandra driver → DB
```

## Step C — Baseline (30 days)

For the top error signature, run the 30-day `date_histogram` from `kibana-query-strategy`. Determine: new vs chronic, and whether first-seen aligns with an image tag change, a cert `notAfter`, or a Kafka partition reassignment.

## Step D — Report

```
INCIDENT <id> | ENV <env> | NS/APP <ns>/<app> | Window (UTC) <from>–<to>
Catalog: svc=<svc> deps=[...] topics=[...] db=<...> egress=[...]  (source: catalog | derived-from-logs)

SYMPTOM      <codes/flags/messages, counts, first/last seen, ongoing?>
FAILING HOP  <edge | gateway | sidecar→app | app | dependency <name> | kafka | db | egress/TLS>
ROOT CAUSE   <one sentence>               CONFIDENCE high|medium|low
EVIDENCE
  1. <skill> query → <sample: ts | pod | req-id | msg>
  2. ...
TIMELINE     <T-x image tag change / cert expiry / lag start / first error / ticket opened>
BASELINE     <new since <date> | chronic, N/day> 
SKILLS RUN   <list>  INDICES <list>  TIER <hot/warm/frozen>
GAPS         <unparsed Envoy logs, no k8s events index, catalog miss, retention, async timeout>
READ-ONLY NEXT CHECKS  <kubectl get/describe, istioctl proxy-config, openssl s_client — for a human with access>
```

## Guardrails

- Stop after 12 MCP calls per domain skill unless evidence is converging; report partial findings and gaps instead of looping.
- Never state a cause without a sample log line.
- If two domain skills disagree, report both with their evidence and lower the confidence.
- Confidence `high` requires: failing hop identified by request-id trace **and** a matching error signature **and** timeline alignment.
