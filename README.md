# kibana-sre-skills

Skill pack for an SRE triage agent that diagnoses incidents from **Kibana/Elasticsearch logs only** (no cluster access), via a Kibana/ES MCP server. Covers 30-day log windows safely.

```
AGENTS.md                         # agent contract + load order
.agents/skills/
  kibana-query-strategy/          # HOW to query: fields, UTC, tiers, async, PIT, aggs   (load first)
  kibana-triage-orchestrator/     # symptom → skill routing, request-id chain walk, report template
  envoy-istio-gateway-triage/     # gateway/sidecar/EnvoyFilter/routing/flags/pod lifecycle
  springboot-dataapi-triage/      # app startup, secrets, exceptions, pools, DataAPI facade
  kafka-triage/
  database-triage/                # JDBC + HikariCP + Cassandra
  certs-tls-triage/
  egress-triage/                  # ServiceEntry, egress gw, DNS, firewall/Panorama, proxy
  request-origin-trace/           # Akamai→Apigee→F5→Istio→app funnel, correlation ids, originator, secret trace
  envoyfilter-lua-auth-triage/    # Envoy Lua/JWT/ext_authz 401 → GitLab source → proposed fix
mcp/
  mcp.agent-builder.json          # Elastic 9.3+ official Kibana MCP
  mcp.es8-community.json          # Elastic 7/8 via cr7258/elasticsearch-mcp-server
  readonly-api-key.json           # least-privilege key body
```

## Choose the MCP

| Elastic version | Use |
|---|---|
| 9.3+ / Serverless | Kibana Agent Builder MCP (`{KIBANA_URL}/api/agent_builder/mcp`) — define per-domain ES\|QL tools in Kibana if you want server-side guardrails |
| 7.x / 8.x | `cr7258/elasticsearch-mcp-server` (has `general_api_request` for `_async_search`, `_pit`, `_field_caps`) |

The skills only need these MCP capabilities: `search` (DSL), raw API passthrough (for `_async_search`, `_pit`, `_field_caps`, `_cat/indices`, `_ilm/explain`), `list_indices`, `get_mappings`.

## Before first run — adapt per environment

1. Index patterns per env (IT/UAT/pre-prod/EN/EMP/PROD) → put in `AGENTS.md` or let Step 0 discover.
2. Gateway namespace/app label if not `istio-system`/`istio-ingressgateway`.
3. Whether these are shipped to Elastic: Akamai DataStream, Apigee message logs, F5/NGINX access logs, RUM/BFF, k8s events, istiod, cert-manager, CoreDNS, Kafka brokers, Panorama/F5, edge NGINX. Each skill degrades gracefully and reports the gap, but knowing up front saves calls.
4. Service catalog MCP tool name → referenced by the orchestrator.
5. If scripted aggregations are disabled, the skills already name the fallback (`error.type` terms / `significant_terms`).

## Test prompts

- "INC0012345: users get 502 on /accounts/summary since 06:30 PST, app payments-api in ns payments-prod"
- "consumer lag alert on topic txn-events, app ledger-consumer ns ledger"
- "dataapi-customer returning 500 intermittently for last 3 days — is it new?"
- "everything calling api.vendor.com fails since 02:00 UTC"
- "customer screenshot shows Reference #18.4f2a1b… at 09:12 PST on /accounts — where did it fail?"
- "frontend gets 401 in a loop after login, don't know if Apigee or app"
