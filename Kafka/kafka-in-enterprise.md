# How Kafka works in enterprise companies

At scale, Kafka stops being "a message broker for one app" and becomes **shared infrastructure that the whole company's data flows through**. The hard problems shift from Kafka mechanics to ownership, governance, reliability and cost.

## 1. The mental shift

| Small project (your setup) | Enterprise |
|---|---|
| 1 cluster, 3 brokers | Many clusters: per environment, per region, sometimes per domain (payments, analytics) |
| You create topics by hand | Self-service via Git/Terraform with approvals and naming rules |
| One team owns everything | A **platform team** runs Kafka, and **dozens of product teams** use it |
| Messages are whatever you send | Messages are **data contracts** with schemas, owners and versioning |
| You restart it if it breaks | SLOs, on-call, runbooks, capacity planning, DR drills |

## 2. The big picture

```
 Source systems                    Kafka (central backbone)                 Consumers
┌───────────────┐                ┌──────────────────────────┐          ┌─────────────────┐
│ Order service │──events──────▶ │ orders.order.created.v1  │────────▶ │ Payment service │
│ Payment svc   │──events──────▶ │ payments.payment.done.v1 │────────▶ │ Inventory svc   │
│ Postgres/Oracle│─CDC(Debezium)▶│ db.orders.public.orders  │────────▶ │ Fraud (Flink)   │
│ Mobile/web    │──clickstream─▶ │ web.clicks.v2            │────────▶ │ Search indexer  │
│ IoT / logs    │──────────────▶ │ ...                      │────────▶ │ Data lake (S3)  │
└───────────────┘                └──────────────────────────┘          │ Data warehouse  │
                                      │            │                   │ Notifications   │
                              Schema Registry   Kafka Connect          └─────────────────┘
                              (contracts)       (no-code in/out)
```

Producers don't know who consumes their data. Adding a new consumer (say, a fraud team) requires **zero changes** to the producer. That decoupling is the main value Kafka brings to large organizations.

## 3. How enterprises run clusters

**Hosting options**
- **Managed:** Confluent Cloud, AWS MSK, Azure Event Hubs (Kafka API), Aiven. Most companies choose this to avoid operating brokers.
- **Self-hosted on Kubernetes:** Strimzi or Confluent Operator. Common in banks and companies with data-residency rules.
- **Self-hosted on VMs/bare metal:** very large companies (LinkedIn, Uber) with dedicated Kafka teams.

**Typical topology**
- Separate clusters for **dev / staging / prod**, with prod on 3 availability zones and `broker.rack` set so replicas spread across zones.
- Often separate clusters by **criticality or domain**: a payments cluster must never be affected by a noisy analytics workload.
- Large clusters can have dozens of brokers and tens of thousands of partitions, which is where controller performance and rebalancing time start to matter.
- **Tiered storage** moves old segments to object storage (S3), so retention can be weeks or months without huge broker disks.

**Why multiple clusters?** Blast radius isolation, different SLAs, different security zones, and independent upgrades.

## 4. The platform team model

Most enterprise Kafka work is organizational:

- **Self-service provisioning:** a team opens a PR to a Git repo (Terraform, Strimzi `KafkaTopic` CRDs, or Julie Ops). CI validates the name, partitions, retention and ACLs, and a human approves. Nobody runs `kafka-topics.sh` on prod.
- **Naming conventions:** `<domain>.<entity>.<event>.v<version>`, for example `payments.invoice.paid.v1`.
- **Ownership:** every topic has an owning team, a documented schema and an SLA. Unowned topics are deleted.
- **Quotas:** per-client or per-user byte-rate limits, so one buggy producer can't take down the cluster.
- **ACLs/RBAC:** a service can write only to its own topics and read only what it is granted.
- **Cost chargeback:** teams are billed for storage, throughput and partitions, which keeps people from creating thousands of useless topics.
- **Standards:** "golden path" client configs (idempotence on, `acks=all`, compression, retries and DLQ templates) shipped as shared libraries or Spring Boot starters.

## 5. Data contracts and schema governance

This is the biggest difference from tutorials. With 200 services you can't have free-form JSON.

- **Schema Registry** with Avro/Protobuf. Producers can't publish if the schema violates the compatibility rule.
- **Compatibility modes:** `BACKWARD` (new schema can read old data) is the most common. `FULL` is the strictest.
- **Safe evolution:** add optional fields with defaults, never rename or delete fields, and never change types. For a breaking change, create `v2` and run both versions in parallel while consumers migrate.
- **Events vs. commands vs. state:**
  - *Event:* "OrderPlaced" (a fact about the past, immutable). Preferred.
  - *Command:* "ChargeCard" (a request, usually better as a direct API).
  - *State/entity topic:* compacted topic holding the latest version of each entity (customer profile).
- **Fat vs. thin events:** thin events carry only an ID and force consumers to call back. Fat events carry the needed data and avoid coupling. Most enterprises prefer fat events plus a stable schema.

## 6. The integration ecosystem

Large companies avoid writing custom code for every pipeline:

- **Kafka Connect:** configuration-driven connectors, with hundreds available.
  - *Source:* Debezium (CDC from Postgres/MySQL/Oracle), JDBC, S3, MQ.
  - *Sink:* Elasticsearch, S3/Iceberg, Snowflake, BigQuery, Redshift.
- **Change Data Capture:** legacy databases stream every change into Kafka without touching application code. This is how monoliths get "strangled" gradually into microservices.
- **Stream processing:** Kafka Streams (a library inside your service), Flink (a separate cluster, for heavy stateful work), ksqlDB (SQL on streams). Use cases: fraud detection, real-time aggregations, joins between streams, enrichment.
- **Data lake/warehouse:** Kafka feeds S3/Iceberg/Snowflake so analytics gets near-real-time data instead of nightly batch jobs.

## 7. Reliability and disaster recovery

**Inside a cluster:** RF=3, `min.insync.replicas=2`, `acks=all`, rack awareness, and `unclean.leader.election=false`. Rolling upgrades are done one broker at a time while watching under-replicated partitions.

**Across regions (this is where the deepest complexity lives):**
- **MirrorMaker 2 / Cluster Linking:** asynchronously replicate topics (and consumer group offsets) to another cluster.
- **Active-passive:** the standby cluster is idle until failover. It is simpler, with a nonzero RPO because replication is async, so a few seconds of data can be lost.
- **Active-active:** both regions take writes. Topics are prefixed (`us.orders`, `eu.orders`), consumers read both, and you need conflict handling. It is complex and used only when truly needed.
- **Stretch cluster:** one cluster across 3 regions, RPO=0, but cross-region latency hurts every write. Only viable with low inter-region latency.
- **Failover is the hard part:** consumers must resume from translated offsets, and producers must switch bootstrap servers. Companies run **DR drills** to prove it works.

**Defining targets:** RPO (how much data you can lose) and RTO (how long you can be down) drive the whole design.

## 8. Security and compliance

- **Encryption:** TLS in transit, disk encryption at rest.
- **Authentication:** mTLS, SASL/SCRAM or OAuth (OIDC) via the company identity provider.
- **Authorization:** ACLs or RBAC per service/team.
- **PII and GDPR:** Kafka logs are immutable, so deleting one user's data is hard. Common solutions are field-level encryption with per-user keys (delete the key, which is "crypto-shredding"), tokenizing PII before it enters Kafka, and short retention on sensitive topics.
- **Audit:** who produced, who consumed, who changed ACLs.
- **Network:** private links, with no public brokers.

## 9. Operating it day to day

**Metrics that matter**
- **Consumer lag** per group, ideally measured in time as well as messages. This is the main business-facing health signal.
- Under-replicated partitions (should be 0), offline partitions (must be 0), and ISR shrink rate.
- Request latency (produce/fetch p99), disk usage, network saturation, and active controller count (must be 1).

**Tooling:** Prometheus + JMX exporter + Grafana, Burrow or Kafka Lag Exporter for lag, Cruise Control for automatic partition rebalancing, and alerts routed to PagerDuty.

**Typical incidents**
- Consumer lag explosion (slow downstream DB, poison message, rebalance storm).
- Hot partition from a bad key.
- Broker disk full (retention misconfigured, or a runaway producer).
- Under-replicated partitions after a broker or network issue.
- Producer retry storm after an outage.

**Capacity planning:** estimate peak messages/sec × average size × retention × RF for disk, plus partition count for parallelism, then add 30-50% headroom.

## 10. Real-world usage patterns

- **LinkedIn** (where Kafka was born): activity tracking, metrics and data pipelines, with trillions of messages a day.
- **Uber:** trip events, driver/rider matching, surge pricing and real-time analytics, with multi-region replication for resilience.
- **Netflix:** a large event pipeline (clickstream, device logs) feeding analytics and personalization.
- **Banks and fintechs:** payment events, fraud scoring in milliseconds, audit trails and event sourcing, with strict security and DR requirements.
- **E-commerce:** order/inventory/shipping choreography via events, search indexing, and recommendation feeds.

## 11. Common enterprise mistakes

1. **Using Kafka as a database or request/response channel.** It is a log, so don't build synchronous RPC over it.
2. **No schema governance.** Free-form JSON means one change breaks 10 teams.
3. **Too many partitions "just in case."** This causes slow failovers and memory overhead.
4. **One giant shared cluster with no quotas.** One bad actor affects everyone.
5. **Ignoring idempotency.** At-least-once delivery means duplicates will happen.
6. **Assuming "exactly-once" covers external systems.** It doesn't extend to your DB or third-party APIs.
7. **No retry/DLQ strategy.** One bad message blocks a partition forever.
8. **No DR testing.** A DR plan that has never been exercised doesn't work.
9. **Topic sprawl.** Without ownership and cleanup, you end up with thousands of orphaned topics.

## 12. Maturity path

| Stage | What it looks like |
|---|---|
| 1 | One team, one cluster, a few topics |
| 2 | Multiple teams, shared cluster, Schema Registry, basic ACLs |
| 3 | Platform team, GitOps topic provisioning, quotas, monitoring and on-call |
| 4 | Data contracts, CDC, Connect, stream processing, data lake integration |
| 5 | Multi-region DR/active-active, tiered storage, chargeback, a company-wide event catalog |

## 13. What you can simulate on your own machine

- [ ] Create topics with the naming convention `domain.entity.event.v1`, defined as code (a script or YAML) instead of by hand.
- [ ] Add Schema Registry compatibility rules and test a breaking vs. non-breaking schema change.
- [ ] Add Debezium (Kafka Connect) on your Postgres and stream table changes into Kafka.
- [ ] Add an S3-compatible sink (MinIO) to mimic a data lake feed.
- [ ] Set producer/consumer quotas and watch throttling happen.
- [ ] Add ACLs with SASL so one service can't read another's topics.
- [ ] Add Prometheus + Grafana with JMX exporter and build a lag dashboard.
- [ ] Spin up a second small cluster and replicate a topic with MirrorMaker 2 (a mini DR setup).
- [ ] Break things deliberately: kill brokers, slow a consumer, send a poison message, then write a runbook for each.

## 14. How to structure a system-design answer

Structure any system-design answer in this order:
1. **Requirements:** throughput, latency, ordering needs, durability (RPO/RTO), retention.
2. **Topics and keys:** one topic per event type, with the key chosen for ordering and even distribution.
3. **Partitions and consumers:** sized from throughput and parallelism.
4. **Reliability:** RF, ISR, `acks`, idempotency, retry/DLQ.
5. **Schema and evolution:** Schema Registry and compatibility.
6. **Integration:** Connect/CDC, stream processing, sinks.
7. **Ops:** monitoring, lag, scaling, DR.
8. **Trade-offs:** say what you chose *not* to do and why.
