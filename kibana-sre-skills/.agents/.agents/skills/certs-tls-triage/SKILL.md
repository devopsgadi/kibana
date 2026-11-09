---
name: certs-tls-triage
description: Diagnose certificate and TLS failures from Kibana — expired/rotated certs, Istio mTLS and SDS/istiod cert issuance, ingress gateway server certs, egress/upstream TLS (PKIX, hostname mismatch, SNI), Java keystore/truststore errors, cert-manager renewals, F5/NGINX Plus edge TLS. Use for "certificate", x509, SSLHandshakeException, handshake_failure, UAEX/response_code 0 with TLS text, or any failure that started at a round date/hour.
---

# Certificates / TLS Triage

Prereq: `kibana-query-strategy`. TLS failures are **date-driven** — always run the 30-day baseline; a signature that starts at an exact minute with no deploy is a cert expiry or rotation.

## Where TLS breaks on your platform

```
client → Akamai → F5/NGINX Plus (edge cert) → istio-ingressgateway (gateway server cert, SDS) 
       → sidecar mTLS (istiod-issued workload certs) → Spring Boot (JVM truststore for outbound) 
       → egressgateway / ServiceEntry (origination TLS) → external (their cert) ; Kafka/DB TLS (truststore)
```

## Step 1 — Signature sweep (all containers, the app's namespace + istio-system)

```json
{"size":0,"query":{"bool":{"filter":[RANGE,
  {"terms":{"{ns}":["NS","istio-system"]}},
  {"query_string":{"query":"certificate OR x509 OR SSLHandshakeException OR handshake_failure OR PKIX OR \"certificate_unknown\" OR \"certificate verify failed\" OR \"unknown ca\" OR \"expired\" OR \"notAfter\" OR \"SNI\" OR \"TLS error\" OR \"CERTIFICATE_VERIFY_FAILED\" OR \"bad certificate\" OR \"SDS\" OR \"keystore\" OR \"truststore\"","fields":["{msg}","error.type"]}}]}},
 "aggs":{"by_app":{"terms":{"field":"{app}","size":15},"aggs":{"by_container":{"terms":{"field":"{container}","size":3}}}},
         "timeline":{"date_histogram":{"field":"@timestamp","fixed_interval":"1m"}},
         "samples":{"top_hits":{"size":5,"_source":["@timestamp","{ns}","{app}","{container}","{msg}"]}}}}
```
Note **which container** logs it: `istio-proxy` → mesh/gateway/egress TLS; app container → JVM truststore/keystore or direct HTTPS.

## Step 2 — Envoy-side TLS signals

Access logs: `response_code: 0` with `response_flags: UF` or `-` and `upstream_transport_failure_reason` present (e.g. `TLS error: 268435581:SSL routines:OPENSSL_internal:CERTIFICATE_VERIFY_FAILED`, `SSLV3_ALERT_HANDSHAKE_FAILURE`, `SSLV3_ALERT_CERTIFICATE_EXPIRED`, `TLSV1_ALERT_UNKNOWN_CA`).

```
{container}:"istio-proxy" AND (upstream_transport_failure_reason:* OR {msg}:(*"TLS error"* OR *CERTIFICATE_VERIFY_FAILED* OR *HANDSHAKE_FAILURE* OR *CERTIFICATE_EXPIRED* OR *UNKNOWN_CA* OR *"sslv3 alert"*))
→ terms on {app}, upstream_cluster, requested_server_name, upstream_transport_failure_reason
```

| Where | Signal | Meaning |
|---|---|---|
| ingressgateway, downstream side (`DC`/`0` at TLS, `requested_server_name` set, no route) | clients drop at handshake; `TLSV1_ALERT_UNKNOWN_CA` from clients | **gateway server cert** expired/wrong chain/missing SAN — check cert-manager (Step 5) |
| ingressgateway → app sidecar `UF` + `CERTIFICATE_VERIFY_FAILED` | mTLS between gateway and workload | istiod/SDS issue (Step 3) or PeerAuthentication mismatch |
| sidecar outbound to `*.svc.cluster.local` `UF` + TLS error | workload mTLS | SDS / DestinationRule `tls.mode` mismatch (ISTIO_MUTUAL vs DISABLE) |
| sidecar outbound to external host, `UF` + `CERTIFICATE_VERIFY_FAILED` / `UNKNOWN_CA` | **egress origination TLS**: remote cert changed/expired or CA bundle in DestinationRule `caCertificates` stale | `egress-triage` |
| sidecar outbound `UF` + `SSLV3_ALERT_HANDSHAKE_FAILURE`, SNI empty | DestinationRule missing `sni`, or protocol/version mismatch | egress-triage |
| 503 `UAEX` | ext-authz TLS or unreachable | envoy skill Step 8 |

## Step 3 — Istio mTLS / SDS / istiod

```
{ns}:"istio-system" AND {app}:"istiod" AND {msg}:(*"certificate"* OR *CSR* OR *"failed to sign"* OR *"root cert"* OR *"cacert"* OR *expired* OR *"authentication failed"* OR *"token"*)
```
```
{container}:"istio-proxy" AND {msg}:(*"SDS"* OR *"failed to fetch secret"* OR *"certificate rotation"* OR *"warming"* OR *"Envoy proxy is NOT ready"* OR *"Failed to update secret"*)
→ terms on {ns},{app},{pod}; timeline
```
- Many pods across namespaces at once → **istiod root/intermediate CA rotation** or istiod outage.
- Single workload → its SA token/JWT (`token expired`, `authentication failed`) or sidecar version skew.
- Pods stuck `NOT ready` with `warming` → SDS never delivered cert → `UH` upstream.

Root CA expiry: istiod logs `root cert expiry` warnings; 30d baseline will show the countdown.

## Step 4 — JVM (Spring Boot, Kafka clients, JDBC TLS)

```
NOT {container}:"istio-proxy" AND {msg}:(*SSLHandshakeException* OR *"PKIX path building failed"* OR *"unable to find valid certification path"* OR *"certificate_unknown"* OR *"No subject alternative"* OR *"Certificate expired"* OR *"NotAfter"* OR *"Keystore was tampered"* OR *"password was incorrect"* OR *"handshake_failure"* OR *"Received fatal alert"*)
→ terms on {app}, error.type, logger_name; extract host from msg if present
```

| Signature | Cause |
|---|---|
| `PKIX path building failed: unable to find valid certification path to requested target` | Truststore lacks the remote's CA — remote rotated to new CA, or truststore secret not updated in this env |
| `No subject alternative DNS name matching X found` | Hostname mismatch — new cert without SAN, or calling by IP/alias |
| `Received fatal alert: certificate_unknown` / `bad_certificate` | Remote rejects **our** client cert (mTLS) — our keystore cert expired/rotated |
| `Received fatal alert: handshake_failure` | Protocol/cipher mismatch (TLS 1.0/1.1 disabled on remote; JVM `jdk.tls.disabledAlgorithms`) |
| `Certificate expired` / `NotAfter: <date>` | Explicit expiry — the date is your evidence |
| `Keystore was tampered with, or password was incorrect` | Keystore secret or password secret rotated independently |
| `Kafka ... SslAuthenticationException` | Broker cert/CA rotated; client truststore stale → `kafka-triage` |
| JDBC `ORA-28860` / `ORA-29024` / `SSL handshake failed` (PG/SQL Server) | DB TLS cert rotated | `database-triage` |

## Step 5 — cert-manager / renewal pipeline (if its logs are shipped)

```
{ns}:(cert-manager OR istio-system OR NS) AND {app}:(cert-manager OR cert-manager-*) AND {msg}:(*"Certificate"* AND (*failed* OR *error* OR *"not ready"* OR *renew* OR *Issuing* OR *"order"* OR *challenge*))
```
Look for the certificate name matching the gateway/app secret; `Renewing` without subsequent `Issued` → stuck renewal → expiry is the cause.

Kubernetes events index (if shipped): `kubernetes.event.reason: (Issuing OR Reissuing OR Failed OR "RequestFailed") AND involved_object.kind: Certificate`.

## Step 6 — Edge (F5 / NGINX Plus) TLS

If gateway shows nothing but clients fail: query edge logs for `SSL_do_handshake() failed`, `no shared cipher`, `certificate verify failed`, `client sent no required SSL certificate`, `SSL: error:... alert bad certificate`. Edge cert expiry = all paths fail at once at exact time.

## Step 7 — Pin the time, prove it's a cert

1. First-seen minute from Step 1 timeline (`1m` buckets) — TLS expiries start at an exact second.
2. No image tag change in ±1h (pod lifecycle) → not a deploy.
3. 30-day async baseline: zero before, wall after → expiry/rotation. Gradual rise → partial rotation (some pods/replicas with new cert).
4. Any `NotAfter`/`expired` text with a date → quote it as evidence.

## Output

```
TLS verdict: <gateway server cert expired @T | istiod CA rotation | workload mTLS/SDS (<pods>) | JVM truststore stale for <host> | egress origination CA (<host>) | keystore/password rotation | edge cert | protocol mismatch>
Layer: <edge | ingress | mesh mTLS | app→dependency | egress | kafka | db>
Hosts/SNI: <...>  Expiry/rotation time: <T from logs>
Evidence: <query> → <ts | ns/app/container | msg[:200]> (≤3)
Gaps: <cert-manager logs not shipped | no upstream_transport_failure_reason field | cannot read secret contents>
Read-only next checks: openssl s_client -connect host:443 -servername host | istioctl proxy-config secret <pod> | kubectl get certificate -A | keytool -list -v
```
