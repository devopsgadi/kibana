---
name: kafka-triage
description: Diagnose Kafka producer/consumer problems from Spring Boot app logs and (if shipped) broker logs in Kibana — consumer lag, rebalance storms, commit failures, serialization/poison messages, broker unavailability, auth/TLS, topic/partition errors. Use when a ticket mentions Kafka, lag, "messages not processed", listener stopped, or an app that consumes/produces events is silently stuck.
---

# Kafka Triage (from Kibana only)

Prereq: `kibana-query-strategy`. Resolve `APP` → topics / consumer groups / bootstrap servers from the catalog; otherwise harvest them from logs (Step 1).

Kafka failures are quiet: the HTTP side stays 200 while the consumer is dead. Always run Step 3 (heartbeat) even if the ticket is about something else and the app is a consumer.

## Base

```
APP_LOGS = {ns}:"NS" AND {app}:"APP" AND NOT {container}:"istio-proxy"
KAFKA    = APP_LOGS AND (logger_name:*kafka* OR {msg}:(*kafka* OR *Kafka* OR *consumer* OR *producer* OR *partition* OR *rebalance*))
BROKER   = {ns}:"KAFKA_NS" AND {app}:(kafka OR kafka-broker OR strimzi*)     # only if brokers log to Elastic
```
Common logger fields: `logger_name`, `log.logger`, `logger`. Spring Kafka loggers: `org.apache.kafka.clients.*`, `org.springframework.kafka.*`, `o.a.k.c.c.internals.ConsumerCoordinator`.

## Step 1 — Harvest config from startup logs (no catalog)

Spring Kafka prints `ConsumerConfig values:` / `ProducerConfig values:` at boot. Pull one:
```
APP_LOGS AND {msg}:("ConsumerConfig values" OR "ProducerConfig values" OR "bootstrap.servers" OR "group.id")
```
Extract `bootstrap.servers`, `group.id`, `client.id`, `max.poll.records`, `max.poll.interval.ms`, `session.timeout.ms`, `enable.auto.commit`, `security.protocol`, `sasl.mechanism`. Needed to interpret Step 4.

## Step 2 — Error signature aggregation (tight window, then 30d baseline)

```json
{"size":0,"query":{"bool":{"filter":[RANGE,NS,APP,{"bool":{"must_not":{"term":{"{container}":"istio-proxy"}}}},
  {"query_string":{"query":"kafka OR Kafka OR rebalance OR partition OR \"consumer group\" OR offset","fields":["{msg}","logger_name"]}}]}},
 "aggs":{"sig":{"terms":{"field":"error.type.keyword","size":15,"missing":"(no exception type)"}},
         "by_logger":{"terms":{"field":"logger_name.keyword","size":10}},
         "timeline":{"date_histogram":{"field":"@timestamp","fixed_interval":"1m","min_doc_count":1}},
         "samples":{"top_hits":{"size":3,"sort":[{"@timestamp":"desc"}],"_source":["@timestamp","{pod}","{msg}"]}}}}
```

## Step 3 — Consumer heartbeat: is anything being consumed?

Count consume-side activity over time. Use whatever the app logs per record (many Spring apps log `Received message`/`Processing event`); fall back to `ConsumerCoordinator` lines.
```json
"aggs":{"consume_rate":{"date_histogram":{"field":"@timestamp","fixed_interval":"5m","min_doc_count":0,
   "extended_bounds":{"min":"FROM","max":"TO"}},
  "aggs":{"records":{"filter":{"query_string":{"query":"\"Received\" OR \"Consumed\" OR \"Processing\" OR \"offset\"","fields":["{msg}"]}}}}}}
```
Flat-lined to zero while HTTP traffic continues = **consumer dead or paused**. Note the exact minute it stopped; cross-check Step 4 for the cause at that minute.

## Step 4 — Signature decoder

| Log signature | Cause | Evidence to collect |
|---|---|---|
| `Attempt to heartbeat failed since group is rebalancing` / `(Re-)joining group` repeating | **Rebalance storm** — pods restarting, `max.poll.interval.ms` exceeded, or scaling flapping | count of `joining group` per pod per 5m; pod lifecycle agg |
| `CommitFailedException ... max.poll.interval.ms` | Poll loop too slow (downstream DB/HTTP slow inside listener) | `duration` of outbound calls in same pods (envoy skill Step 4); `database-triage` |
| `Offset commit failed on partition ... UNKNOWN_MEMBER_ID` / `ILLEGAL_GENERATION` | Consumer kicked from group; same root as above | first occurrence timestamp |
| `Error while fetching metadata ... UNKNOWN_TOPIC_OR_PARTITION` | Topic missing / wrong env topic name / ACL hides it | topic name in msg vs catalog |
| `TOPIC_AUTHORIZATION_FAILED` / `GROUP_AUTHORIZATION_FAILED` / `SaslAuthenticationException` | ACL/credential problem — usually after secret rotation | `springboot-dataapi-triage` secret check; first-seen vs deploy |
| `Connection to node -1 (bootstrap...) could not be established` / `Bootstrap broker ... disconnected` | Broker unreachable: DNS, network policy, egress, broker down | `egress-triage` if brokers external; broker logs if shipped; other apps on same bootstrap failing? |
| `SSLHandshakeException` / `certificate_unknown` / `PKIX path building failed` in kafka logger | Truststore / broker cert rotated | `certs-tls-triage` |
| `NOT_LEADER_OR_FOLLOWER` / `NotLeaderForPartition` / `LEADER_NOT_AVAILABLE` | Broker-side leader election / ISR shrink; transient unless sustained | rate over time; broker logs `Shrinking ISR` |
| `RecordTooLargeException` | Message > `max.request.size` / broker `message.max.bytes` | size in msg; which producer |
| `SerializationException` / `Error deserializing key/value for partition X at offset N` | **Poison message** — consumer stuck re-reading same offset | same `partition`+`offset` repeating → confirm stuck; count per minute |
| `ListenerExecutionFailedException` + same offset repeating | Listener throws, no error handler / DLT → infinite retry | offset in message constant over time |
| `TimeoutException: Expiring N record(s) ... has passed since batch creation` | Producer can't deliver; broker slow/unreachable | correlate with broker connectivity lines |
| `Container stopped` / `KafkaMessageListenerContainer ... Stopping` without restart | Listener container stopped after fatal error (auth, deserialization with `FATAL` handler) | last lines before stop |
| `Node X disconnected` burst across many apps simultaneously | Broker restart/rolling upgrade | multi-app query on same bootstrap host |

Match the first-seen time of the signature to the Step 3 flat-line minute. If they coincide → root cause found.

## Step 5 — Blast radius

Who else consumes the same topic or bootstrap? Query all namespaces:
```
NOT {container}:"istio-proxy" AND {msg}:("BOOTSTRAP_HOST" OR "TOPIC_NAME") AND {level}:(ERROR OR WARN)
→ terms on {ns}, {app}
```
Many apps failing together → broker/platform issue. One app → app-side (config, poison message, slow listener).

## Step 6 — Broker logs (only if `BROKER` filter returns docs)

```
BROKER AND {msg}:(*"Shrinking ISR"* OR *"under-replicated"* OR *"Preferred leader election"* OR *"Controller moved"* OR *OutOfMemory* OR *"Disk"* OR *"Too many open files"* OR *"Authentication failed"*)
```
If no broker docs: report "broker logs not in Elastic; broker-side cause cannot be confirmed from Kibana".

## Step 7 — 30-day baseline

Async `date_histogram` (`3h`) for the chosen signature + `consume_rate`. Report first-seen date, correlation with image tag change (pod lifecycle), and whether the pattern recurs (e.g. nightly batch causing rebalance).

## Output

```
Kafka verdict: <consumer dead since T | rebalance storm | poison message p=X off=N | broker unreachable | auth/TLS | producer timeout | healthy>
Config seen: bootstrap=<...> group=<...> max.poll.interval.ms=<...>
Blast radius: <apps affected>
Evidence: <query> → <ts | pod | msg[:200]> (≤3)
Gaps: <no broker logs | consume-rate proxy used | catalog miss>
Read-only next checks: kafka-consumer-groups --describe --group <g>; kubectl get kafkatopic (Strimzi)
```
