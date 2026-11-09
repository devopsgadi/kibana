---
name: egress-triage
description: Diagnose outbound connectivity from the mesh to external or cross-cluster destinations using Kibana — Istio egress gateway, ServiceEntry/DestinationRule gaps (NR/NC/404 passthrough), BlackHoleCluster, DNS failures, TLS origination, Palo Alto Panorama / F5 / NetworkPolicy denies, proxy (HTTP_PROXY) issues, and third-party API outages. Use for "cannot reach <external host>", UnknownHostException, egress 403/404/503, or dependency failures to anything outside the cluster (Key Vault, OIDC/JWKS, payment APIs, Kafka/DB outside mesh).
---

# Egress Triage

Prereq: `kibana-query-strategy`. Resolve the destination host from the ticket or from `authority`/`requested_server_name` in sidecar outbound logs. Catalog tells you whether the host should route via egressgateway, direct sidecar passthrough, or an explicit proxy.

```
SIDECAR_OUT = {ns}:"NS" AND {app}:"APP" AND {container}:"istio-proxy" AND upstream_cluster:outbound*
EGRESS_GW   = {ns}:"istio-system" AND {app}:"istio-egressgateway"
APP_LOGS    = {ns}:"NS" AND {app}:"APP" AND NOT {container}:"istio-proxy"
```

## Step 1 — What external hosts does this app talk to, and which are failing?

```json
{"size":0,"query":{"bool":{"filter":[RANGE,NS,APP,{"term":{"{container}":"istio-proxy"}},{"prefix":{"upstream_cluster":"outbound"}},
   {"bool":{"must_not":{"wildcard":{"upstream_cluster":"*.svc.cluster.local*"}}}}]}},
 "aggs":{"host":{"terms":{"field":"authority.keyword","size":25,"missing":"(tcp/sni)"},
   "aggs":{"sni":{"terms":{"field":"requested_server_name.keyword","size":3}},
           "cluster":{"terms":{"field":"upstream_cluster.keyword","size":3}},
           "codes":{"terms":{"field":"response_code","size":6}},
           "flags":{"terms":{"field":"response_flags.keyword","size":6}},
           "p99":{"percentiles":{"field":"duration","percents":[99]}}}}}}
```
`upstream_cluster` tells you the path:
| `upstream_cluster` value | Path |
|---|---|
| `PassthroughCluster` | no ServiceEntry; sidecar passes raw to original IP (ALLOW_ANY). Works until someone sets REGISTRY_ONLY or a NetworkPolicy/firewall blocks it |
| `BlackHoleCluster` | **REGISTRY_ONLY mode and no ServiceEntry** → 502/503 `UH`/`NC`, or 404 for HTTP. Root cause = missing ServiceEntry |
| `outbound\|443\|\|api.vendor.com` | ServiceEntry exists; failures are real connectivity/TLS |
| `outbound\|443\|\|istio-egressgateway.istio-system.svc.cluster.local` | routed via egress gateway → check Step 3 |
| `InboundPassthroughCluster*` | wrong direction; ignore |

## Step 2 — Flag decoder (sidecar outbound, external host)

| Code/flag | Meaning | Next |
|---|---|---|
| 502/503 `UH` + `BlackHoleCluster` | REGISTRY_ONLY + missing ServiceEntry | fix: ServiceEntry (report, don't apply) |
| 404 `NR` with external `authority` | HTTP host not in registry (REGISTRY_ONLY) or VirtualService for egress missing | ServiceEntry/VS |
| 503 `NC` | cluster vanished — ServiceEntry deleted or DNS resolution for `resolution: DNS` failed | istiod logs; DNS Step 4 |
| 503 `UF` + `upstream_transport_failure_reason` TLS | origination TLS failure | `certs-tls-triage` |
| 503 `UF` without TLS reason, `duration` ≈ connect timeout (10s) | **TCP blocked** — firewall (Panorama), NSG/NetworkPolicy, proxy required | Step 5 |
| 503 `UF` fast (<50ms) `connection refused` | host up, port closed / wrong port in ServiceEntry | ServiceEntry port |
| 504 `UT` | remote slow / hanging — third-party degradation | Step 6 |
| 0 `DC` on long-lived TCP (Kafka, DB, MQ) | connection reset by firewall idle timeout | idle keepalive vs firewall policy; `kafka-triage`/`database-triage` |
| 403 from remote, body `Forbidden`/`IP not allowed` | **Source IP not whitelisted** — egress NAT IP changed (new node pool, new egress gateway IP) | Step 5; compare first-seen with infra change |
| 401 from remote | credential / token — not egress | springboot |
| `DF` | DNS failure in Envoy (`resolution: DNS`) | Step 4 |
| 200 but slow p99 | WAN/proxy | Step 6 |

HTTPS with `PassthroughCluster`: you only see `requested_server_name` + bytes, no code. Use `response_flags` and `duration` and the **app container's** exception (Step 4) for detail.

## Step 3 — Egress gateway path

If the app's `upstream_cluster` points at `istio-egressgateway`, the app sidecar may show 200/503 from the gateway, and the real failure is on the gateway's outbound leg:
```
EGRESS_GW AND authority:"HOST" (or requested_server_name:"HOST")
→ code × flags × upstream_transport_failure_reason; timeline; samples with x_request_id
```
Trace the same `x_request_id` through app sidecar → egressgateway. Gateway shows `UF`/TLS → certs; gateway shows `NR` → egress Gateway/VS host mismatch; gateway has **no** entry → the app's VS to the egress gateway didn't match (app sidecar flag will be `NR`).

## Step 4 — DNS

App container:
```
APP_LOGS AND {msg}:(*UnknownHostException* OR *"Name or service not known"* OR *"Temporary failure in name resolution"* OR *"NXDOMAIN"* OR *"getaddrinfo"* OR *"DNS resolution failed"* OR *EAI_AGAIN*)
→ extract host; timeline; per pod (one node? all?)
```
Sidecar/istiod: `{msg}:(*"dns"* AND (*fail* OR *timeout* OR *SERVFAIL*))`; CoreDNS logs if shipped: `{ns}:kube-system AND {app}:(coredns OR kube-dns) AND {msg}:(*SERVFAIL* OR *"i/o timeout"* OR *NXDOMAIN* AND "HOST")`.

Patterns: one node's pods only → node DNS/conntrack; everything → CoreDNS/upstream DNS; only one host → external DNS record changed/removed (also check `NC` in Step 2 for `resolution: DNS` ServiceEntries).

## Step 5 — Firewall / Panorama / F5 / NetworkPolicy / Proxy (correlating, since you can't see the rule)

What Kibana can show:
- **Shape**: `UF` with `duration` pinned at connect timeout (~10,000ms) and no TLS reason → SYN dropped silently → firewall/NSG/NetworkPolicy. `connection refused` fast → something answers → not a firewall drop.
- **Scope**: all namespaces to this host failing (cross-ns query on `authority:"HOST" AND response_flags:UF`) → perimeter (Panorama/NSG). One namespace → NetworkPolicy/AuthorizationPolicy. One node → node-level.
- **Start time**: exact minute, no deploy → policy push. Compare with Panorama/F5 logs if they are shipped to Elastic (`{ns}:security OR index pattern paloalto-*`: `action:deny AND dst:"IP"`; F5 `irule`/`asm` denies).
- **Proxy**: app logs `407 Proxy Authentication Required`, `Unable to tunnel through proxy`, `Proxy CONNECT aborted` → HTTP_PROXY/NO_PROXY misconfig (often NO_PROXY missing the cluster CIDR after config centralization).
- **NAT IP change**: remote 403/`IP not whitelisted` starting after a node pool/egress gateway change.

Panorama/firewall logs shipped? Query them:
```
index: paloalto-* OR panorama-* : dst_ip:"RESOLVED_IP" AND action:(deny OR drop OR reset-both) AND RANGE
→ terms on rule_name, src_ip (map src to node IPs / egress gateway)
```
Not shipped → state "firewall decision not visible in Kibana; shape consistent with silent drop" and lower confidence to medium.

## Step 6 — Third-party outage vs us

Same host failing from **all** apps in **all** clusters with 5xx/`UT` from the remote (not `UF`) → vendor side. Only our cluster → us. Quote remote `response_code` and any body/error text the app logged. Check if the app has fallback/circuit-breaker lines (`CircuitBreaker ... OPEN`).

## Step 7 — Correlate & baseline

- Pod lifecycle: deploy in ±1h? (new host added in code, NO_PROXY change)
- istiod: `{app}:istiod AND {msg}:(*ServiceEntry* OR *"REGISTRY_ONLY"* OR *outboundTrafficPolicy*)` → mesh config change
- 30-day async baseline for `authority:"HOST" AND response_flags:(UF OR UH OR NR OR NC)` — new vs chronic; intermittent daily = WAN/vendor maintenance

## Output

```
Egress verdict: <missing ServiceEntry (BlackHole/NR) | TCP blocked (firewall/NSG/NetPol) | DNS (<host>) | TLS origination → certs | proxy misconfig | NAT IP not whitelisted | egress-gateway leg failed | vendor outage | healthy>
Host: <authority/SNI>  Path: <passthrough | ServiceEntry | egressgateway | proxy>
Scope: <this app | namespace | cluster-wide>   Start: <T>  Deploy/mesh-config change: <none | ...>
Evidence: <query> → <ts | pod | req-id | flags | msg[:200]> (≤3)
Gaps: <firewall logs not shipped | HTTPS passthrough hides codes | no CoreDNS logs>
Read-only next checks: istioctl proxy-config cluster <pod> --fqdn HOST ; kubectl get serviceentry,networkpolicy -n NS ; nslookup HOST from a debug pod ; Panorama traffic log filter dst=IP
```
