---
name: kibana-query-strategy
description: How the SRE agent must query Elasticsearch/Kibana through the MCP — field discovery, time pinning, 30-day retention tiers, async search, PIT pagination, aggregate-before-fetch. Load this FIRST in any triage; every domain skill assumes it.
---

# Kibana / Elasticsearch Query Strategy (30-day window safe)

This skill is about *how* to query, not *what* to look for. Domain skills (envoy, kafka, springboot, database, certs, egress) tell you what to look for and reference the rules here.

## Hard rules

1. **Read-only.** Only `GET`/`POST _search`, `_async_search`, `_pit`, `_field_caps`, `_cat/indices`, `_ilm/explain`. Never write, delete, reindex, or change ILM.
2. **Discover before you query.** Run Step 0 once per incident and cache the field map. Never assume ECS names.
3. **Always pin an absolute UTC window.** Convert the ticket timezone explicitly and print the conversion.
4. **Aggregate first, fetch raw docs last.** Never `size > 500` on a raw query; never `from/size` past 10k. Use `terms`/`date_histogram` to locate the problem, then fetch ≤ 50 sample docs.
5. **Windows > 24h go through `_async_search`.** Windows > 7d also set `"terminate_after"` and target only the indices that cover the window.
6. **Cite evidence.** Every finding = query used + 1–3 sample docs (`@timestamp`, pod, request id, message).

## Step 0 — Field discovery (run once per incident)

```http
GET {log_index_pattern}/_field_caps?fields=kubernetes.*,log.level,message,response_code,response_flags,upstream_cluster,x_request_id,*request_id*,container.image.name,event.*
```

Fallback if `_field_caps` is blocked: fetch one doc for the app and read keys.

```json
POST {log_index_pattern}/_search
{ "size": 1, "sort": [{"@timestamp":"desc"}],
  "query": {"bool":{"filter":[
    {"term":{"kubernetes.namespace":"NS"}},
    {"term":{"kubernetes.labels.app":"APP"}}]}} }
```

Placeholder map used by every domain skill:

| Placeholder | Filebeat/ECS | Fluent Bit | Fluentd (kubernetes_metadata) |
|---|---|---|---|
| `{ns}` | `kubernetes.namespace` | `kubernetes.namespace_name` | `kubernetes.namespace_name` |
| `{app}` | `kubernetes.labels.app` | `kubernetes.labels.app` | `kubernetes.labels.app` |
| `{pod}` | `kubernetes.pod.name` | `kubernetes.pod_name` | `kubernetes.pod_name` |
| `{container}` | `kubernetes.container.name` | `kubernetes.container_name` | `kubernetes.container_name` |
| `{image}` | `container.image.name` | `kubernetes.container_image` | `kubernetes.container_image` |
| `{level}` | `log.level` | `level` | `level` |
| `{msg}` | `message` | `log` | `log` |
| `{ts}` | `@timestamp` | `@timestamp` | `@timestamp` |

Envoy access-log fields, when parsed: `response_code`, `response_flags`, `upstream_cluster`, `upstream_host`, `authority`, `path`, `method`, `x_request_id`, `duration`, `upstream_service_time`, `bytes_sent`, `downstream_remote_address`, `requested_server_name`, `route_name`.

If Envoy fields are **not** structured, access logs are raw text in `{msg}`. Use wildcard/regex on `{msg}` and report **"ingest pipeline needed — Envoy access logs unparsed"** as a gap.

## Time pinning

```
INCIDENT_TIME (ticket, local tz) → UTC
WINDOW_TIGHT  = T-15m .. T+15m        (first pass)
WINDOW_WIDE   = T-2h  .. T+2h          (if tight is empty)
WINDOW_TREND  = T-30d .. T              (baseline / "has this happened before")
```

Always emit: `Window (UTC): 2026-11-09T14:30Z – 2026-11-09T15:00Z (ticket said 06:30 PST)`.

## 30-day retention: tiers and how to query each

Logs older than a few days usually live on slower tiers. Find out before querying:

```http
GET _cat/indices/{log_index_pattern}?h=index,creation.date.string,store.size,pri,rep,health&s=creation.date
GET {index}/_ilm/explain
```

| Age | Typical tier | Rule |
|---|---|---|
| 0–2d | hot | Normal `_search`, tight window, `size ≤ 200` |
| 2–7d | warm | `_search` ok, but aggregations only; raw docs `size ≤ 50` |
| 7–30d | cold / frozen (searchable snapshot) | `_async_search` **only**; `date_histogram` + `terms`; `terminate_after: 100000`; restrict `index` list to the dated indices covering the window; expect 30–120s |
| > 30d | gone / snapshot | Report "outside retention" — do not retry |

Build the index list from the dates instead of using the wildcard pattern alone:

```
logs-app-2026.10.12, logs-app-2026.10.13, ... logs-app-2026.11.09
```

Or use the alias with an `@timestamp` range and `"_ignore_throttled": false` only when the index pattern spans frozen tier (some clusters require `?ignore_throttled=false`).

## Async search pattern (any window > 24h)

```http
POST {indices}/_async_search?wait_for_completion_timeout=10s&keep_alive=10m
{
  "size": 0,
  "terminate_after": 100000,
  "query": {"bool":{"filter":[
    {"range":{"@timestamp":{"gte":"FROM","lte":"TO"}}},
    {"term":{"{ns}":"NS"}},
    {"term":{"{app}":"APP"}}]}},
  "aggs": { ...see domain skill... }
}
```

Then poll `GET _async_search/{id}` until `is_running: false`. Do not spin faster than every 5s. Delete the search when done: `DELETE _async_search/{id}` (this is the one DELETE that is allowed — it frees cluster memory, touches no data).

## Standard aggregations (copy into any domain query)

**Find the spike** — always first on wide windows:
```json
"aggs": {"over_time": {"date_histogram": {"field":"@timestamp","fixed_interval":"5m","min_doc_count":1},
  "aggs": {"errors": {"filter": {"terms": {"{level}": ["ERROR","FATAL","error","fatal"]}}}}}}
```
Choose `fixed_interval`: ≤2h → `1m`; ≤24h → `5m`; ≤7d → `30m`; ≤30d → `3h`.

**Top-N dimension** — pods, flags, loggers, upstreams:
```json
"aggs": {"by_x": {"terms": {"field":"{field}.keyword","size":20,"order":{"_count":"desc"}}}}
```

**Pod lifecycle** — who lived, who died, image tag changes:
```json
"aggs": {"pods": {"terms": {"field":"{pod}","size":50},
  "aggs": {"first": {"min":{"field":"@timestamp"}},
           "last":  {"max":{"field":"@timestamp"}},
           "image": {"terms":{"field":"{image}","size":3}}}}}
```

**Fingerprint errors** — cluster identical stack traces without pulling them all:
```json
"aggs": {"sig": {"terms": {"script": {"source": "def m = doc['{msg}.keyword'].size()>0 ? doc['{msg}.keyword'].value : ''; return m.length()>120 ? m.substring(0,120) : m;"}, "size": 15}}}
```
(If scripted aggs are disabled, use `significant_terms` on `{msg}` or `error.type.keyword`.)

## Pagination when you must read raw docs

Open a PIT, then `search_after`. Never `from`.

```http
POST {indices}/_pit?keep_alive=5m
POST _search
{ "size": 200, "pit": {"id":"PIT_ID","keep_alive":"5m"},
  "sort": [{"@timestamp":"asc"},{"_shard_doc":"asc"}],
  "query": {...}, "search_after": [LAST_SORT_VALUES] }
```
Stop after 5 pages (1,000 docs). If you need more, your aggregation was wrong — go back and narrow.

## Request-ID correlation (cross-skill)

`x-request-id` is the universal join key across gateway → sidecar → app → downstream sidecar. Trace it with **no label filter** and ±1h:

```json
{"query":{"bool":{"filter":[
  {"range":{"@timestamp":{"gte":"T-1h","lte":"T+1h"}}},
  {"multi_match":{"query":"REQUEST_ID","fields":["x_request_id","x-request-id","request_id","trace.id","traceId","{msg}"]}}]}},
 "sort":[{"@timestamp":"asc"}],"size":200}
```

Spring Boot apps often log `traceId`/`spanId` (Sleuth/Micrometer) rather than `x-request-id`; if the ticket gives one, search both fields.

## Baseline / "is this new?" (uses the 30d window)

For any error signature found in the tight window, run one async query over `WINDOW_TREND` with a `date_histogram` (`3h`) on the same filter. Report:
- first-seen date
- daily rate before vs during incident
- whether it correlates with an image tag change (pod lifecycle agg)

This is what turns "there are errors" into "these errors started with build 4.12.3 on Nov 6".

## Output block every domain skill appends

```
Field mapping: {ns}=kubernetes.namespace, {app}=kubernetes.labels.app, ...
Indices queried: <list>  Tier: <hot|warm|frozen>  Mode: <_search|_async_search>
Window (UTC): <from> – <to>  (converted from <ticket tz>)
Query: <compact DSL or KQL>
Sample evidence (≤3): <@timestamp | pod | request id | message[:200]>
Gaps: <unparsed fields, missing index, retention, throttled tier timeouts>
```
