---
name: incident-intake
description: Front door for every incident. Takes only an INC number (and whatever description exists), scores what is actually known (app, namespace, time, host/path, error text, ids), enriches from ServiceNow, and bootstraps the unknowns from the logs themselves — error-text search, host/path → upstream service, anomaly scan across all apps — until there is enough to start evidence-collection-plan Phase 0. Never stalls on a thin ticket: it picks the best-supported candidate, records every assumption, and asks the human at most one targeted question in parallel.
---

# Incident Intake — from "INC0012345" to a runnable triage

Goal: produce `intake{T, env, app, ns, host, path, error_text, ids, status, known_level, assumptions[], question}` and hand off to `evidence-collection-plan` Phase 0. Max 1 ServiceNow call, 1 catalog call, 6 ES calls. Never ask the human *before* trying the bootstrap queries; ask one question *while* continuing.

## Step 1 — Pull the whole ticket, not the short description

ServiceNow MCP: `GET incident INC…` → fields: `short_description, description, cmdb_ci, business_service, u_application/u_app_id, opened_at, caller_id, location, category, assignment_group, work_notes, comments, related alerts/events (em_alert / em_event), attachments names, parent incident`.

Extract with the regexes from `evidence-collection-plan` 0.1 plus:
- **app candidates**: `cmdb_ci`, `business_service`, `u_application`, any token matching catalog naming (`*-api`, `*-svc`, `*-service`, `*-web`, `*-bff`, `dataapi-*`), hostnames (`\b[\w.-]+\.(com|net|bank|internal)\b`), URL paths (`/\S+`), Apigee proxy names, Kafka topics (`\b[\w.-]+-(events|topic)\b`), DB names.
- **time candidates** (priority order): explicit timestamp in description → alert `event time` from related alerts → work-note "since <time>" → `opened_at − 30m` (mark `assumed`).
- **env**: words `PROD|prod|production|UAT|IT|pre-prod|EN|EMP`, hostnames (`uat.`, `-it.`), or `cmdb_ci` environment attribute. Default PROD only if ticket priority ≥ P2; else ask.
- **symptom class**: status codes, words (`timeout`, `slow`, `login`, `401`, `certificate`, `kafka`, `lag`, `database`, `cannot reach`, `blank page`, `Reference #`).
- **caller type**: `caller_id` in a customer-facing group / "customer reports" → external path (Akamai/Apigee); internal team → may be internal caller.

## Step 2 — Score what you know

```
known.app   = app candidate resolves in catalog (exact or alias)          → 1 | 0
known.time  = explicit or alert-derived (not opened_at)                   → 1 | 0
known.where = host or path or request id or Akamai/Apigee ref             → 1 | 0
known.what  = status code or error text or symptom class                  → 1 | 0
level       = A (app+time) | B (where/what + time, no app) | C (app, no time) | D (what only) | E (nothing usable)
```
Record `intake.known_level`. Every unknown you fill below goes into `assumptions[]` with the evidence that justified it.

## Step 3 — Bootstrap by level (ES calls; all W = last 2h unless time known → W1)

### Level A — app + time known
Go to `evidence-collection-plan` Phase 0. No bootstrap needed.

### Level B — know where/what, not which app
**B1 host/path → app.** Gateway logs, W1:
```json
{"size":0,"query":{"bool":{"filter":[RANGE,{"term":{"{ns}":"GW_NS"}},{"term":{"{app}":"GW_APP"}},
  {"bool":{"should":[{"wildcard":{"authority.keyword":"*HOST*"}},{"wildcard":{"path.keyword":"*PATH*"}}],"minimum_should_match":1}}]}},
 "aggs":{"svc":{"terms":{"field":"upstream_cluster.keyword","size":10},
   "aggs":{"err":{"filter":{"range":{"response_code":{"gte":500}}}}}}}}
```
`upstream_cluster` = `outbound|PORT||SVC.NS.svc.cluster.local` → `app=SVC`, `ns=NS` (catalog confirms). Prefer the cluster with highest `err`. If gateway blind, use Apigee `target.url` or F5 `upstream_addr` → Service.

**B2 error text → app.** All app logs, W1, no ns filter:
```json
{"size":0,"query":{"bool":{"filter":[RANGE,{"bool":{"must_not":{"term":{"{container}":"istio-proxy"}}}},
  {"match_phrase":{"{msg}":"<error_text, longest distinctive 4–8 word fragment>"}}]}},
 "aggs":{"by_app":{"multi_terms":{"terms":[{"field":"{ns}"},{"field":"{app}"}],"size":10},
   "aggs":{"first":{"min":{"field":"@timestamp"}},"tl":{"date_histogram":{"field":"@timestamp","fixed_interval":"5m"}}}}}}
```
Fallback if no phrase match: `query_string` with the 3 rarest tokens (drop numbers, ids, common words).

**B3 request id / trace id** → cross-skill request-id query (no label filter) → `{ns},{app}` of hops → app = the *last* hop that logged (not the gateway).

**B4 Akamai/Apigee ref** → `request-origin-trace` Step 1 anchor → `x-request-id` → B3.

Result: top candidate → `app, ns` with `assumption:"app derived from <B1|B2|B3|B4>: <evidence>"`. If two candidates within 20% of each other → run the plan for the top one, list the second in `assumptions` as "alternate — run if first is clean".

### Level C — app known, time not
**C1 find the spike.** For the app, last 24h (async if cold):
```json
{"size":0,"query":{"bool":{"filter":[{"range":{"@timestamp":{"gte":"now-24h"}}},NSF,APPF]}},
 "aggs":{"tl":{"date_histogram":{"field":"@timestamp","fixed_interval":"5m","min_doc_count":0},
   "aggs":{"err":{"filter":{"bool":{"should":[{"terms":{"{level}":["ERROR","FATAL"]}},{"range":{"response_code":{"gte":500}}}],"minimum_should_match":1}}}}}}}
```
`T` = start of the first 5m bucket where `err/total` > 3× the median of the preceding 2h and stays elevated ≥ 2 buckets. No spike → `T = opened_at − 30m`, `assumption:"no error spike in 24h; using opened_at−30m"`, and widen the plan to W1 from the start.
If `opened_at` is > 24h ago, scan `[opened_at−24h, opened_at]` instead of `now-24h`.

### Level D — only a symptom ("customers can't log in", "site slow", "payments failing")
**D1 catalog keyword map** (1 call): symptom nouns → candidate apps (`login|auth|sso` → auth/identity services; `payment|transfer` → payments; `balance|account` → accounts/dataapi-*; `slow|site` → BFF/frontend gateway hosts). Catalog should expose `tags`/`capabilities`; if not, keep a static map in `AGENTS.md`.

**D2 global anomaly scan** (2 calls, last 2h vs previous 22h):
Gateway 5xx by upstream service:
```json
{"size":0,"query":{"bool":{"filter":[{"range":{"@timestamp":{"gte":"now-24h"}}},{"term":{"{ns}":"GW_NS"}},{"term":{"{app}":"GW_APP"}}]}},
 "aggs":{"svc":{"terms":{"field":"upstream_cluster.keyword","size":40},
   "aggs":{"recent":{"filter":{"range":{"@timestamp":{"gte":"now-2h"}}},"aggs":{"e5":{"filter":{"range":{"response_code":{"gte":500}}}}}},
           "before":{"filter":{"range":{"@timestamp":{"lt":"now-2h"}}},"aggs":{"e5":{"filter":{"range":{"response_code":{"gte":500}}}}}}}}}}
```
Compute per service `rate_recent = recent.e5/recent.doc_count`, `rate_before = before.e5/before.doc_count`; rank by `rate_recent − rate_before` (min 50 docs recent).
App ERROR delta across all namespaces:
```json
{"size":0,"query":{"bool":{"filter":[{"range":{"@timestamp":{"gte":"now-24h"}}},{"bool":{"must_not":{"term":{"{container}":"istio-proxy"}}}},{"terms":{"{level}":["ERROR","FATAL"]}}]}},
 "aggs":{"by_app":{"multi_terms":{"terms":[{"field":"{ns}"},{"field":"{app}"}],"size":40},
   "aggs":{"recent":{"filter":{"range":{"@timestamp":{"gte":"now-2h"}}}},"before":{"filter":{"range":{"@timestamp":{"lt":"now-2h"}}}}}}}}
```
Rank by `recent / (before/11)` (per-2h normalized).

**D3 intersect**: candidates in both D1 and D2 top-5 → pick; else top of D2 that is customer-facing per catalog. `T` = C1 on the chosen app. `assumption:"app chosen by anomaly scan: <svc> 5xx 0.2%→14% in last 2h; symptom map agreed/disagreed"`.

### Level E — nothing usable (empty description, no CI, no alerts)
Run D2 only. If a clear anomaly exists (one service ≥ 10× its baseline error rate) → treat as Level D with that app, flagged `assumption:"ticket empty; triaging the only anomalous service"`. If nothing is anomalous → produce an **intake report**, not a triage: "no app/time/symptom in INC; no service anomalous in last 2h; need one of: app name, time, URL, screenshot reference #" and stop. This is the only case where you stop before the plan.

## Step 4 — One parallel question (optional, never blocking)

If `known_level ∈ {B,C,D}` post one comment to the INC / return to the user **while continuing**: the single most valuable missing fact, phrased so a non-engineer can answer:
- B: "Which application or URL were you using? (e.g. the address bar)"
- C: "Roughly when did it start (time + timezone)? Is it still happening?"
- D: "Do you have a Reference # from the error page, or the exact error text?"
Record `intake.question`. Do not wait for the answer; if it arrives in a later turn, re-score and, if the app changes, restart the plan for the new app (keep the old ledger under `superseded`).

## Step 5 — Hand off

```
INTAKE | INC <id> | level <A–E> | T=<utc> (<source>) | env=<..> (<source>)
app/ns = <..> (<source: ticket|catalog|B1|B2|B3|B4|D2>)   alternates: [..]
where  = host:<..> path:<..> ids:{..}     what = status:<..> class:<..> text:"<..>"
caller = <external customer | internal team | alert>
assumptions: [ "<fact> ← <evidence>", ... ]
question posted: "<..>" | none
→ evidence-collection-plan Phase 0 with these inputs
```
The final report must carry `assumptions[]` verbatim under a **BASIS** line and cap confidence at `medium` whenever `app` or `T` was derived rather than stated.

## Rules

- Never ask before bootstrapping. A thin ticket is a query problem first, a people problem second.
- Never pick an app because its name sounds right; pick it because logs (B/D) or catalog (A/D1) point at it, and say which.
- Derived `T` always widens the first plan window to W1.
- If the description contradicts the logs (ticket says app X, logs show X clean and Y failing on the same path) → triage Y, report the contradiction up front.
- Repeat/parent incidents: if ServiceNow links a parent or a recent similar INC, read its resolution notes (1 call) and add `assumption:"similar to INC… resolved by …"` — a hint, not a conclusion.
