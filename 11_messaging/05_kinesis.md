# Amazon Kinesis — Real-Time Data Streaming

> **Who this is for**: Engineers who understand the **stream/log model** from
> [01_messaging_concepts.md](01_messaging_concepts.md) and the queue ([02_sqs.md](02_sqs.md)) and
> pub/sub ([03_sns.md](03_sns.md)) services, and now need AWS's **real-time streaming** family.
> The **Kinesis vs SQS** distinction (replay, ordering, multiple consumers, retention) is the key
> exam takeaway.

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
| Best for | Decouple + buffer + process-once | Real-time analytics, ordered multi-consumer, replay |

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

**Shards** — the unit of capacity and parallelism. Each shard ingests **1 MB/s or 1,000 records/s**
and emits **2 MB/s**. Total stream capacity = sum of shards. Two capacity modes:
- **Provisioned** — you set the shard count (and pay per shard).
- **On-demand** — Kinesis auto-scales shards with traffic (pay per throughput).

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
events. It's how you build robust multi-instance consumers.

---

## 3. Kinesis Data Firehose

**Kinesis Data Firehose** is the **near-real-time delivery** service: it loads streaming data into
destinations with **no code and no servers**. It buffers records (by size or time) and writes them
out.

```
   producers ──► Firehose ──[buffer + optional transform]──► destination
                                                              ├─ Amazon S3
                                                              ├─ Amazon Redshift
                                                              ├─ Amazon OpenSearch
                                                              └─ Splunk / HTTP endpoints
```

| | Data Streams | Firehose |
|---|---|---|
| Latency | **Real-time** (sub-second) | **Near-real-time** (buffered, ~60s min) |
| Storage / replay | Stores records (replayable) | **No storage** — delivers and forgets |
| Scaling | Shards (provisioned/on-demand) | **Fully managed, auto-scales** |
| Custom code consumers | Yes (KCL, Lambda, EFO) | No — delivers to fixed destinations |
| Transform | In your consumer | Built-in **Lambda transform**; format convert (e.g., to Parquet) |
| Use for | Build custom real-time apps | **ETL load** streaming → S3/Redshift/OpenSearch/Splunk |

✅ Firehose is the answer for "stream data into S3/Redshift/OpenSearch with minimal effort." It is
serverless and requires no shard management.

⚠️ Firehose does **not** retain data or support replay — if you need to reprocess, you need Data
Streams (or read the delivered data back from S3).

---

## 4. Analytics — Data Analytics & Managed Service for Apache Flink

**Kinesis Data Analytics** lets you run **SQL queries on streaming data** in real time (tumbling /
sliding windows, aggregations) without managing servers — e.g., a rolling 1-minute count of errors.

**Amazon Managed Service for Apache Flink** (the rebranded/expanded successor) runs **Apache Flink**
applications for advanced stream processing (stateful computations, event-time windows, complex
event processing) over Kinesis or other sources. Output goes to Kinesis, Firehose, etc.

💡 For the exam: "run **SQL / real-time analytics on a stream**" → Kinesis Data Analytics; "managed
**Apache Flink** for complex stream processing" → Managed Service for Apache Flink. Both consume
from Data Streams/Firehose.

---

## 5. Typical Pipeline

How the pieces compose into a real architecture:

```
   clickstream/IoT ─► Data Streams ─► Data Analytics (SQL aggregations)
                          │                    │
                          │                    └─► Firehose ─► S3 (data lake) / OpenSearch
                          └─► Lambda / KCL app (real-time reactions)
```

- **Data Streams** for ingestion with ordering + replay + multiple consumers.
- **Data Analytics / Flink** for real-time computation.
- **Firehose** for durable delivery into S3 / Redshift / OpenSearch / Splunk.

---

## 6. Kinesis vs SQS — When to Use Streams

| Choose | When |
|--------|------|
| **SQS** | Decouple two components, buffer work, process **once per message**, no need to replay or for multiple independent consumers. Simplest, fully managed. |
| **Kinesis Data Streams** | Need **ordering** (per key), **multiple independent consumers** of the same data, **replay** of historical records, or **high-volume real-time analytics** (clickstream, IoT, logs). |
| **Firehose** | Just need to **load** streaming data into S3/Redshift/OpenSearch/Splunk with no code. |

> **Rule of thumb**: If the words **real-time analytics, ordered, replay, multiple consumers, or
> clickstream/IoT/logs** appear → **Kinesis**. If it's **decouple / buffer / worker queue** →
> **SQS**.

---

## 7. Key Exam Points

- **Kinesis Data Streams** = ordered, replayable, durable **log** split into **shards**; many
  independent consumers; retention **24h → 365 days**.
- **Shard** = 1 MB/s or 1,000 records/s in, 2 MB/s out; **partition key** routes records and
  preserves **per-key order**; beware **hot shards**.
- **Enhanced fan-out** gives each consumer a **dedicated 2 MB/s per shard** (low-latency push).
- **KCL** handles shard distribution and checkpointing for multi-instance consumers.
- **Firehose** = serverless **near-real-time delivery** to **S3 / Redshift / OpenSearch / Splunk**;
  optional **Lambda transform**; **no storage / no replay**.
- **Kinesis Data Analytics** = SQL on streams; **Managed Service for Apache Flink** = managed Flink
  for advanced stream processing.
- **Kinesis vs SQS**: stream **retains + replays + many consumers + ordered**; SQS **deletes after
  processing, one consumer per message**.

---

## 8. Common Mistakes

- ❌ Using **SQS** when the requirement is **replay, ordering, or multiple independent consumers** —
  that's Kinesis.
- ❌ Picking a **low-cardinality partition key** → all records hit one **hot shard**, throttling the
  whole stream.
- ❌ Expecting **Firehose** to retain or replay data — it delivers and forgets; use Data Streams.
- ❌ Forgetting that adding consumers to a **shared** stream splits the 2 MB/s read budget — use
  **enhanced fan-out** when many consumers read the same shard.
- ❌ Confusing **near-real-time** (Firehose, buffered ~60s) with **real-time** (Data Streams).
- ❌ Choosing Kinesis for simple **decoupling/buffering** where SQS is cheaper and simpler.

---

**Next**: [../12_monitoring/01_cloudwatch.md — Amazon CloudWatch](../12_monitoring/01_cloudwatch.md)
