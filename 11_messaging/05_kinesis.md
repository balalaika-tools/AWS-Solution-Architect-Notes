# Amazon Kinesis & Data Firehose — Real-Time Data Streaming

> **Who this is for**: Engineers who understand the **stream/log model** from
> [01_messaging_concepts.md](01_messaging_concepts.md) and the queue ([02_sqs.md](02_sqs.md)) and
> pub/sub ([03_sns.md](03_sns.md)) services, and now need AWS's **real-time streaming and delivery**
> services. The **Kinesis vs SQS** distinction (replay, ordering, multiple consumers, retention) is
> the key exam takeaway.

---

## 1. Prerequisite: Streaming vs Batch (and vs a Queue)

**Batch processing** collects data over a window (hourly, nightly) and processes it in bulk — high
latency, simple, great for reports. **Stream processing** handles each record (or micro-batch) as
it arrives — low latency, continuous, great for real-time dashboards, anomaly detection, and
clickstreams.

```
   BATCH:    [ collect all day ] ───────► [ process at 2am ] ──► report
   STREAM:   record→ record→ record→ ──► [ process continuously ] ──► live dashboard
```

**How a stream differs from a queue (SQS)** — the single most important comparison:

| | SQS queue | Kinesis stream |
|---|---|---|
| After a record is read | **Deleted** | **Stays** until retention expires |
| Replay old data | ❌ No | ✅ Re-read from any point |
| Ordering | Standard: no; FIFO: per group | **Per shard (partition key)** |
| Consumers per record | **1** (competing pool) | **Many**, each independent |
| Retention | ≤ 14 days | **24h default → 365 days** |
| Throughput model | Per-message, auto-scales | Per **shard** (provisioned) or on-demand |
| Best for | Decouple + buffer work for competing consumers | Real-time analytics, ordered multi-consumer, replay |

> **Key insight**: A queue is a *to-do list* — you take an item and it's gone. A stream is a
> *DVR/tape* — the recording stays, many viewers watch independently, and you can rewind. Reach for
> Kinesis when you need **ordering, multiple independent consumers, replay, or high-volume
> real-time analytics**.

---

## 2. Kinesis Data Streams (KDS)

The core streaming service: an ordered, replayable, durable stream split into **shards**.

```
                       Kinesis Data Stream
   producers      ┌───────────────────────────────────┐    consumers
   ┌────────┐     │ Shard 1: │r│r│r│r│r│ ◄─ ordered    │   ┌──────────┐
   │ apps,  │────►│ Shard 2: │r│r│r│r│   ◄─ ordered    │──►│ Lambda / │
   │ agents │     │ Shard 3: │r│r│r│r│r│ ◄─ ordered    │   │ KCL app  │
   └────────┘     └───────────────────────────────────┘   └──────────┘
       partition key decides which shard a record lands in
```

**Shards** — the unit of capacity, ordering, and parallelism. In provisioned mode, each shard
supports sustained writes of **1 MiB/s or 1,000 records/s** (whichever limit is reached first) and
shared reads of **2 MiB/s** with up to five `GetRecords` calls/s. Total stream capacity is the sum
of shards, but one partition key can use only one shard's capacity.

Current capacity modes are:

- **Provisioned** — set the shard count and pay per shard-hour plus PUT payload units. Best when
  traffic is predictable and the team will monitor and reshard.
- **On-demand Standard** — Kinesis automatically adapts capacity and charges for data written/read,
  plus applicable stream and optional-feature charges. It still cannot scale one hot partition key
  beyond a shard's write limit.
- **On-demand Advantage** — adds warm throughput and a different usage-pricing model, with an
  account-level minimum billed ingestion and retrieval commitment. It can be attractive at higher
  aggregate throughput or fan-out, but can be wasteful for a small workload; validate the current
  Regional pricing and mode availability.

A record payload can be up to **10 MiB** before base64 encoding. Kinesis provides burst behavior
for intermittent large records, but do not capacity-plan sustained writes above 1 MiB/s per shard.
`PutRecords` accepts up to 500 records with a 10 MiB total request limit.

**Partition key** — supplied with each record; its hash decides the shard. All records with the
**same partition key go to the same shard**, preserving **order within that key**.

⚠️ **Hot shard / hot partition**: a poorly chosen partition key (e.g., a constant value) sends all
records to one shard, capping you at that shard's 1 MB/s regardless of total shard count. Choose a
high-cardinality, evenly distributed key.

**Retention** — **24 hours by default, up to 365 days**. Records persist for the whole window, so
consumers can **replay** by re-reading from an earlier position.

**Consumers**:
- **Shared (standard) consumers** — share the shard's 2 MB/s read throughput across all consumers
  (poll via `GetRecords`).
- **Enhanced fan-out (EFO)** — each registered consumer gets its **own dedicated 2 MB/s per shard**,
  pushed via HTTP/2 with lower latency. Use when multiple consumers would otherwise contend for
  read throughput.

```
   Shared:   shard 2MB/s ÷ shared by all consumers   (they compete)
   EFO:      shard ──2MB/s──► consumer A
             shard ──2MB/s──► consumer B   (each gets a dedicated pipe)
```

**KCL (Kinesis Client Library)** — a library that handles consumer coordination: distributing
shards across worker instances, checkpointing progress (in DynamoDB), and rebalancing on scale
events. Current KCL versions also handle the parent/child shard relationships created by resharding.
It's how you build robust multi-instance consumers.

---

## 3. Capacity Math, Hot Shards, and Resharding

For a provisioned stream with average write rate `R` records/s and payload rate `W` MiB/s, the
minimum write shard count is:

```
write_shards = max(ceil(R / 1000), ceil(W / 1))
```

If `C` shared consumers each read the full stream at approximately the ingest rate, a stream-level
starting estimate for reads is:

```
read_shards = ceil((W × C) / 2)
required_shards = max(write_shards, read_shards)
```

This math assumes partition keys distribute load evenly. Add headroom for bursts, retries, record
size variance, and resharding time. EFO removes read-throughput contention by giving each registered
consumer 2 MiB/s per shard, but adds mode-dependent cost and consumer quotas; On-demand Standard
and Provisioned streams support 20 registered EFO consumers, while On-demand Advantage can support
more in available Regions.

### Diagnose before adding capacity

| Signal | Interpretation |
|--------|----------------|
| `WriteProvisionedThroughputExceeded` | Producers exceeded record/byte capacity, usually on one or more shards. |
| `ReadProvisionedThroughputExceeded` | Shared consumers exceeded read bytes or `GetRecords` call rate. |
| Shard-level `IncomingBytes` / `IncomingRecords` | Identifies a hot hash-key range hidden by healthy stream-wide averages. |
| `IteratorAgeMilliseconds` | A consumer is falling behind; compare it with retention and the recovery-time objective. |
| Consumer process/checkpoint metrics | Separates Kinesis throttling from slow business logic or a stuck KCL worker. |

Enable shard-level enhanced monitoring during diagnosis or continuously for critical streams.
Correlate throttles with top partition keys in producer telemetry; Kinesis metrics cannot tell you
which business key created the skew by themselves.

### Reshard deliberately

For provisioned streams, `UpdateShardCount` scales the stream broadly. Split the hash range of a hot
shard when targeted capacity is needed, and merge adjacent cold shards to reduce cost. Resharding
creates parent and child shards rather than rewriting old records; consumers must finish/checkpoint
parents before processing their children to preserve order. Use a current KCL and monitor lease
movement and iterator age throughout the change.

On-demand modes handle stream capacity changes, but they cannot repair a single hot partition key.
Change the partitioning scheme—such as salting a high-volume tenant key—only after deciding how
consumers will reconstruct order. Salting improves parallelism by weakening one-key ordering.

---

## 4. Producer and Consumer Correctness

### Producer retries and duplicate records

`PutRecords` can partially succeed: inspect every result entry and retry **only failed records**
with exponential backoff and jitter. Retrying the whole batch duplicates successful records. A
producer can also time out after Kinesis accepted a write but before the acknowledgement arrived;
the retry then creates another record with a new sequence number.

Put a stable application `event_id` in every record and make downstream state changes idempotent.
Kinesis sequence numbers order records within a shard but are not cross-retry deduplication keys.
If strict per-entity order matters, serialize writes for that key and take special care that retry
logic does not let a later record overtake an uncertain earlier write.

### KCL checkpoints and recovery

KCL uses a DynamoDB lease table to assign shards to workers and persist checkpoints. Checkpoint
**after** the consumer has durably committed all side effects for the covered records. A worker
failure before the checkpoint causes records since the previous checkpoint to be delivered again;
checkpointing before the side effect can lose processing. KCL therefore provides at-least-once
processing, not exactly-once business outcomes.

Use a distinct KCL application name/lease table for each independent consumer application. Choose
the initial position (`LATEST`, `TRIM_HORIZON`, or a timestamp) deliberately, alarm on iterator age,
and set stream retention longer than the maximum detection plus repair and catch-up window. Store
idempotency state at least as long as a record can be replayed.

---

## 5. Security, Cross-Account Access, and Multi-Region Recovery

Kinesis Data Streams encrypts traffic in transit and supports server-side encryption with an AWS
owned/managed key or a customer-managed KMS key. With a customer-managed key, producers need
`kms:GenerateDataKey`; consumers need `kms:Decrypt`. Restrict key use with the Kinesis stream ARN
encryption context and monitor key disable/rotation changes as production events.

For cross-account access:

- Attach a resource policy to the stream that trusts the exact external principal and actions, and
  give that principal an identity policy for the same stream ARN. Cross-account trust needs both.
- EFO sharing needs permissions on **both** the stream ARN and registered consumer ARN.
- If the stream is encrypted for cross-account use, use a customer-managed KMS key and grant the
  external producer/consumer in its key policy; the AWS managed service key cannot express that
  cross-account trust.
- Use an SDK that accepts a stream ARN for cross-account writes. Keep administration, production,
  consumption, and resource-policy changes in separate roles.

Streams and checkpoints are Regional, and Kinesis Data Streams does not provide managed
cross-Region stream replication. Choose recovery from the workload RPO/RTO:

- **Recreate on failover** — lowest steady cost, but loses the unavailable Region's buffered
  records and has the longest RTO.
- **Asynchronous replicator** — an EFO consumer reads the primary stream and writes stable event IDs
  to a secondary stream. Monitor replication iterator age; accept an RPO equal to lag and make the
  secondary duplicate-safe.
- **Producer dual write** — potentially lower RPO, but one Regional write can succeed while the
  other fails. Persist an outbox/reconciliation record rather than assuming two API calls are
  atomic.

Deploy consumer code, KCL lease tables, KMS keys, quotas, downstream state, alarms, and failover
routing in the secondary Region. Decide how checkpoints are initialized and how duplicate records
from replication, failover, and failback are reconciled.

---

## 6. Amazon Data Firehose (formerly Kinesis Data Firehose)

**Amazon Data Firehose** is the **near-real-time delivery** service: it loads streaming data into
destinations with **no code and no servers**. It buffers records (by size or time) and writes them out.
It was formerly named **Amazon Kinesis Data Firehose**; the rename did not change service endpoints,
APIs, CLI commands, IAM policy actions, or CloudWatch metrics.

```
   producers ──► Data Firehose ──[buffer + optional transform]──► destination
                                                              ├─ Amazon S3
                                                              ├─ Amazon Redshift
                                                              ├─ Amazon OpenSearch
                                                              └─ Splunk / HTTP endpoints
```

| | Data Streams | Data Firehose |
|---|---|---|
| Latency | **Real-time** (sub-second) | **Near-real-time**, based on destination and buffer hints |
| Storage / replay | Stores records (replayable) | **No storage** — delivers and forgets |
| Scaling | Shards (provisioned/on-demand) | **Fully managed, auto-scales** |
| Custom code consumers | Yes (KCL, Lambda, EFO) | No — delivers to fixed destinations |
| Transform | In your consumer | Built-in **Lambda transform**; format convert (e.g., to Parquet) |
| Use for | Build custom real-time apps | **ETL load** streaming → S3/Redshift/OpenSearch/Splunk |

✅ Data Firehose is the answer for "stream data into S3/Redshift/OpenSearch with minimal effort."
It is serverless and requires no shard management.

⚠️ Data Firehose does **not** retain data or support replay — if you need to reprocess, you need Data
Streams (or read the delivered data back from S3).

---

## 7. Stream Processing — Managed Service for Apache Flink

**Amazon Managed Service for Apache Flink** is the current managed stream-processing service. It
runs Apache Flink applications for stateful computations, event-time windows, aggregations,
enrichment, anomaly detection, and continuous ETL over live streams.

It was previously known as **Kinesis Data Analytics for Apache Flink**. Older **Kinesis Data
Analytics for SQL applications** are now legacy: new SQL applications could not be created after
October 15, 2025, and AWS began deleting those SQL applications on January 27, 2026. For current
designs, use Managed Service for Apache Flink or Flink Studio rather than the old SQL-app service.

💡 For the exam: "run **real-time analytics / windowed aggregations on a stream**" → Managed
Service for Apache Flink. If an older question says **Kinesis Data Analytics**, map that to the
current Flink service unless it is explicitly talking about the discontinued SQL-only application
model. Flink commonly reads from **Kinesis Data Streams** or **MSK** and writes results to streams,
S3, OpenSearch, or downstream applications.

---

## 8. Typical Pipeline

How the pieces compose into a real architecture:

```
   clickstream/IoT ─► Data Streams ─► Managed Service for Apache Flink
                          │                    │
                          │                    └─► Data Firehose ─► S3 (data lake) / OpenSearch
                          └─► Lambda / KCL app (real-time reactions)
```

- **Data Streams** for ingestion with ordering + replay + multiple consumers.
- **Managed Service for Apache Flink** for real-time computation.
- **Data Firehose** for durable delivery into S3 / Redshift / OpenSearch / Splunk.

---

## 9. Service and Cost Decision

| Choose | When | Main cost/operations drivers |
|--------|------|------------------------------|
| **SQS** | Decouple components and distribute each work item to a competing consumer; no independent replay consumers. | Requests, payload size, KMS, and consumer compute; least stream-specific operational work. |
| **Kinesis Data Streams — provisioned** | Need per-key order, replay, and independent consumers with predictable traffic. | Shard-hours, PUT payload units, retention, EFO consumer-shard/data retrieval, and resharding operations. |
| **Kinesis Data Streams — on-demand** | Same stream semantics with variable traffic or less capacity management. | Data written/read, stream or minimum-commitment rules by mode, retention, and data transfer; compare Standard and Advantage at aggregate account usage. |
| **Amazon MSK** | Need Kafka APIs/ecosystem, Kafka Connect, configurable topic retention/partitions, portability, or managed cross-Region replication. | Provisioned broker/storage/network capacity or MSK Serverless usage, partition limits, upgrades/configuration, and Kafka operational expertise. |
| **Data Firehose** | Need managed delivery to S3/Redshift/OpenSearch/Splunk/HTTP rather than a replayable application log. | GB ingested/delivered, transformation/format conversion, backup, destination cost, and buffer latency; no custom consumer fleet. |

There is no universally cheapest stream. Compare a representative month including average and peak
ingest, record-size billing units, number of full-stream consumers, retention and replay volume,
cross-Region transfer, encryption calls, and operator time. Kinesis is usually the lower-friction
AWS-native application stream; MSK is justified by Kafka compatibility and ecosystem requirements;
Firehose is simpler when delivery—not consumer-controlled replay—is the outcome.

> **Rule of thumb**: If the words **real-time analytics, ordered, replay, multiple consumers, or
> clickstream/IoT/logs** appear → **Kinesis**. If it's **decouple / buffer / worker queue** →
> **SQS**.

---

## 10. Key Exam Points

- **Kinesis Data Streams** = ordered, replayable, durable **log** split into **shards**; many
  independent consumers; retention **24h → 365 days**.
- **Provisioned shard** = 1 MiB/s or 1,000 records/s sustained write, 2 MiB/s shared read;
  **partition key** routes records and
  preserves **per-key order**; beware **hot shards**.
- Size provisioned streams from both byte and record rates, then validate partition-key skew.
- **Enhanced fan-out** gives each consumer a **dedicated 2 MB/s per shard** (low-latency push).
- **KCL** handles leases, resharding, and checkpoints, but recovery can replay records; consumers
  must be idempotent.
- **Amazon Data Firehose** (formerly Kinesis Data Firehose) = serverless **near-real-time delivery**
  to **S3 / Redshift / OpenSearch / Splunk**; optional **Lambda transform**; **no storage / no replay**.
- **Managed Service for Apache Flink** = current managed stream processing; older Kinesis Data
  Analytics SQL applications are legacy/discontinued.
- **Kinesis vs SQS**: stream **retains + replays + many consumers + ordered**; SQS **deletes after
  processing, one consumer per message**.
- Cross-account streams need resource + identity policies and customer-managed KMS permissions;
  multi-Region replication is application-managed.

---

## 11. Common Mistakes

- ❌ Using **SQS** when the requirement is **replay, ordering, or multiple independent consumers** —
  that's Kinesis.
- ❌ Picking a **low-cardinality partition key** → all records hit one **hot shard**, throttling the
  whole stream.
- ❌ Expecting **Data Firehose** to retain or replay data — it delivers and forgets; use Data Streams.
- ❌ Forgetting that adding consumers to a **shared** stream splits the 2 MB/s read budget — use
  **enhanced fan-out** when many consumers read the same shard.
- ❌ Confusing **near-real-time buffered delivery** (Data Firehose) with real-time, consumer-driven
  processing (Data Streams).
- ❌ Choosing Kinesis for simple **decoupling/buffering** where SQS is cheaper and simpler.
- ❌ Choosing old **Kinesis Data Analytics SQL applications** for a new design — use Managed Service
  for Apache Flink.
- ❌ Retrying an entire partially failed `PutRecords` batch and duplicating successful records.
- ❌ Checkpointing before side effects are durable, or assuming a KCL checkpoint creates
  exactly-once processing.
- ❌ Adding shards while retaining one hot partition key—the key still maps to one shard.
- ❌ Treating a second Regional stream as a replica without a monitored replicator, checkpoint
  strategy, stable event ID, and failback plan.

---

**Next**: [../12_monitoring/01_cloudwatch.md — Amazon CloudWatch](../12_monitoring/01_cloudwatch.md)
