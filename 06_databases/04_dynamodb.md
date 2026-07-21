# Amazon DynamoDB — Managed NoSQL

> **Who this is for**: Engineers who understand the NoSQL key-value/document model and
> horizontal scaling from [01_database_fundamentals.md](01_database_fundamentals.md) and want
> the AWS flagship NoSQL service. DynamoDB is one of the most tested services on SAA-C03 — know
> capacity modes, indexes, and "when DynamoDB beats RDS" cold.

---

## 1. What DynamoDB Is

**Amazon DynamoDB** is a fully managed, **serverless** NoSQL **key-value and document** database.
There are no servers or instances to provision — AWS handles partitioning, replication, and
scaling automatically.

Headline properties the exam expects:
- **Single-digit-millisecond** latency at any scale (microseconds with DAX).
- Virtually **unlimited horizontal scale** — handles millions of requests per second.
- Data is replicated across **3 AZs** in a Region by default for durability and availability.
- **Fully managed / serverless** — no patching, no instance sizing, no Multi-AZ to configure.

> **Key insight**: DynamoDB trades relational features (`JOIN`s, complex queries, flexible
> ad-hoc filtering) for effortless, predictable scale. You design the table around your **access
> patterns** up front, not around normalized relationships.

---

## 2. Core Concepts — Tables, Items, Attributes, Keys

```
  TABLE: Music
  ┌──────────────┬───────────────┬──────────┬──────────┐
  │ Artist (PK)  │ SongTitle (SK)│ Genre    │ Year     │  ← attributes
  ├──────────────┼───────────────┼──────────┼──────────┤
  │ "Daft Punk"  │ "Get Lucky"   │ "Disco"  │ 2013     │  ← item
  │ "Daft Punk"  │ "One More..." │ "House"  │ 2001     │  ← item (same PK)
  │ "Adele"      │ "Hello"       │ "Pop"    │ 2015     │  ← item
  └──────────────┴───────────────┴──────────┴──────────┘
```

- **Table** — a collection of items (no fixed schema beyond the key).
- **Item** — a single record (like a row), up to **400 KB**.
- **Attribute** — a field within an item (like a column); items can have different attributes.
- **Primary key** — uniquely identifies each item, in one of two forms:

| Key type | Made of | How items are located |
|----------|---------|-----------------------|
| **Partition key (simple)** | One attribute | Hashed to choose a partition; key must be unique |
| **Partition key + Sort key (composite)** | Two attributes | Same PK groups items; SK orders/queries within the group |

> **Rule**: The **partition key** determines *which physical partition* an item lives on. A good
> partition key has **high cardinality and even access** so load spreads across partitions. The
> **sort key** lets you query ranges and ordered data within a single partition key.

---

## 3. Capacity Modes — On-Demand vs Provisioned

You choose how DynamoDB throughput is billed and scaled.

| | **On-Demand** | **Provisioned** |
|--|---------------|-----------------|
| You specify | Nothing — it scales automatically | **RCUs** and **WCUs** |
| Billing | Pay per request | Pay for provisioned capacity/hour |
| Scaling | Automatic, subject to per-table/partition ramp behavior and quotas | Manual, or **Auto Scaling** to a range |
| Best for | Spiky / unpredictable / new apps | Steady, predictable traffic |
| Cost | Higher per request | Cheaper if utilization is high |

**Capacity units** (provisioned mode):
- **RCU (Read Capacity Unit)**: 1 strongly consistent read/sec of up to 4 KB (or 2 eventually
  consistent reads/sec).
- **WCU (Write Capacity Unit)**: 1 write/sec of up to 1 KB.
- **Auto Scaling** adjusts provisioned RCU/WCU between a min and max based on utilization.

⚠️ Exceeding provisioned capacity (without auto scaling/burst) causes throttling. On-demand
immediately supports up to roughly **twice the table's previous peak**; a jump beyond 2× within
30 minutes can still throttle while capacity is allocated. Pre-warm a known launch peak or raise
the table's warm throughput, and remember that one hot key still has a partition-level ceiling.

💡 Exam clue: "unpredictable traffic / new app / spiky / don't want to manage capacity" →
**On-Demand**. "Predictable steady load, cost-sensitive" → **Provisioned + Auto Scaling**.

---

## 4. Secondary Indexes — GSI vs LSI

Indexes let you query on attributes other than the primary key. This comparison is a frequent
exam target.

| | **GSI (Global Secondary Index)** | **LSI (Local Secondary Index)** |
|--|----------------------------------|---------------------------------|
| Partition key | **Different** PK (and optional SK) | **Same** PK as table, different **SK** |
| When created | Any time (add/remove later) | **Only at table creation** |
| Capacity | **Own** RCU/WCU (separate) | Shares the table's RCU/WCU |
| Consistency | **Eventual only** | Strong or eventual |
| Per table | Up to **20** | Up to **5** |
| Use when | Query by a totally different attribute | Alternate sort order under the same PK |

```
  TABLE PK = Artist
       │
       ├── LSI:  same PK (Artist), different SK (e.g. Year)  → "Daft Punk songs by year"
       │         must exist at table creation
       │
       └── GSI:  new PK (e.g. Genre), new SK (e.g. Year)     → "all Disco songs by year"
                 can be added later, own throughput
```

> **Rule**: Need a query keyed on a *completely different attribute*, or to add the index later?
> → **GSI**. Need a different *sort key* under the *same* partition key, decided at creation? →
> **LSI**.

---

## 5. DynamoDB Streams

**DynamoDB Streams** is an ordered, time-ordered log of item-level changes (inserts, updates,
deletes) in a table, retained for **24 hours**.

```
  item change ──► DynamoDB Stream ──► Lambda (trigger)
                                  └──► replicate / aggregate / notify
```

- Commonly triggers a **Lambda** function for event-driven processing (e.g., send a welcome
  email when an item is inserted, maintain aggregates, replicate to another store).
- Underpins **Global Tables** (cross-Region replication uses Streams).
- Stream/Lambda delivery is **at least once**. Consumers must make side effects idempotent and
  tolerate duplicate or retried records.

💡 Exam clue: "react to changes in a DynamoDB table / trigger code on insert" → **Streams + Lambda**.

---

## 6. DAX — In-Memory Acceleration

**DynamoDB Accelerator (DAX)** is a fully managed, in-memory cache **built for DynamoDB**.

```
  app ──► DAX (cache) ──► DynamoDB
          microseconds     single-digit ms on cache miss
```

- Cuts read latency from **single-digit milliseconds → microseconds** for read-heavy/repeated
  reads.
- **API-compatible** with DynamoDB — minimal app changes (point the SDK at the DAX cluster).
- Caches both individual items and query/scan results; handles cache invalidation for you.

⚠️ DAX is a **write-through** cache for DynamoDB *specifically*. For caching arbitrary data
(e.g., RDS query results, sessions), use **ElastiCache** instead — see
[05_elasticache_and_others.md](05_elasticache_and_others.md).

---

## 7. Global Tables — Multi-Region Active-Active

**Global Tables** replicate a DynamoDB table across multiple Regions with **multi-active**
read/write in every Region.

```
   us-east-1  ◄──── cross-Region replication ──────►  eu-west-1
   read/write                                          read/write
```

- Every replica Region is **read AND write** (active-active).
- Great for **globally distributed, low-latency** applications and Region-level DR.
- New designs choose a consistency mode:

| Mode | Replication and conflict behavior | Design implications |
|------|-----------------------------------|---------------------|
| **Multi-Region eventual consistency (MREC)** | Asynchronous; concurrent updates use **last-writer-wins** | Lowest write latency; RPO is normally seconds and stale reads/conflicts are possible |
| **Multi-Region strong consistency (MRSC)** | Synchronous quorum; conflicting simultaneous writes fail with `ReplicatedWriteConflictException` | RPO 0 for a single-Region failure, but higher latency, constrained Region sets/topologies, and feature restrictions |

MRSC doesn't turn DynamoDB into a relational database: transactions aren't supported on MRSC
global tables, and applications must catch/retry replication conflicts. MREC remains the normal
choice for latency-sensitive global writes when last-writer-wins is acceptable or the data model
avoids writing the same item in multiple Regions.

⚠️ DynamoDB manages the Streams-based replication, but it does **not** reroute users. Distinguish
this from Aurora Global Database, which has one writer Region and read-only secondaries.

---

## 8. TTL, Backups & Consistency

**TTL (Time To Live)**: set an expiry timestamp attribute; DynamoDB auto-deletes expired items
(no WCU cost). Ideal for session data, temporary tokens, event logs.

**Backups**:
- **PITR (Point-In-Time Recovery)**: continuous backups with a configurable **1–35 day** recovery
  period; the latest restorable time is normally about five minutes behind current time.
- **On-demand backup**: full backups retained until deleted, for long-term/compliance.

Every restore creates a **new table**. Reapply or verify auto scaling, alarms, IAM/resource
policies, tags, TTL, Streams, and PITR, then redirect clients after validation.

**Read consistency** (choose per read):

| | **Eventually consistent** (default) | **Strongly consistent** |
|--|-------------------------------------|-------------------------|
| Latest data? | Maybe stale for ~1s | Always latest |
| Cost | Cheaper (½ RCU) | Costs more (full RCU) |
| Availability | Higher | May fail if a replica is unreachable |

DynamoDB also supports **transactions** (ACID across multiple items in one account and Region)
via `TransactWriteItems` / `TransactGetItems` when you need all-or-nothing operations.

---

## 9. Production Workload Evolution

### 9.1 Evolve a single-table model from access patterns

An order service needs these operations: get an order, list a customer's recent orders, list an
order's lines, and find orders by status. One table can colocate the aggregate:

| Item | Base keys | Sparse index keys | Access pattern |
|------|-----------|-------------------|----------------|
| Order header | PK `ORDER#9001`, SK `META` | GSI1 `CUSTOMER#42` / `2026-07-21#9001`; GSI2 `STATUS#PENDING` / `2026-07-21#9001` | List a customer's orders or pending orders by time |
| Order line | PK `ORDER#9001`, SK `LINE#0001` | None | Query `ORDER#9001` to return the header and ordered lines together |

Add an exact access pattern before adding an index; use key conditions rather than a table scan
plus filter. Store entity type/version attributes, enforce condition expressions, and keep large
receipts in S3. A single-table model is a deliberate API contract, not a mandate: use separate
tables or a relational engine when teams need independent lifecycle/access control, queries are
unpredictable, or cross-entity constraints and reporting dominate.

### 9.2 Diagnose and remediate a hot key

**Adaptive capacity** automatically shifts capacity toward hot partitions and can let one item
use up to its partition's limit (roughly 3,000 RCUs or 1,000 WCUs). DynamoDB can also split some
hot item collections. This is helpful and free, but it cannot make an endlessly hot single key
scale without limit; LSIs can also constrain partition splitting.

Use Contributor Insights and per-table/index throttling, consumed-capacity, latency, and key
distribution to identify the item/access pattern. Then choose the smallest durable fix:

- cache read-heavy immutable items with DAX or the application;
- shard a write-hot logical key (`EVENT#123#00` … `#15`) and query/aggregate the shards;
- add time buckets for unbounded time-series collections;
- distribute monotonic ingestion keys with a hash suffix;
- remove an accidental scan or under-provisioned GSI, or increase/pre-warm capacity.

### 9.3 Evacuate a Global Table Region

Keep identical table names, encryption/IAM dependencies, Lambda consumers, quotas, and application
configuration deployed in each Region. Health-check the **whole write path**, not only the DynamoDB
endpoint. During an evacuation:

1. Stop or drain writes in the impaired Region so retries don't fight the traffic shift.
2. For MREC, inspect replication latency/pending replication and accept the measured RPO. For MRSC,
   confirm the remaining replicas/witness still form a write quorum.
3. Move clients with Global Accelerator, Route 53, or application routing and invalidate regional
   endpoint caches.
4. Reconcile business invariants and duplicate side effects; MREC last-writer-wins can hide a
   losing concurrent update.
5. Restore traffic gradually. Do not delete/re-add a replica until the documented recovery path
   and data state are understood.

Run the exercise regularly. "Active-active" describes database endpoints, not automatic
application failover or conflict-free business logic.

### 9.4 Transactions and conditional writes

Prefer a single-item conditional update for uniqueness, counters, optimistic locking, or idempotent
state transitions. Use a transaction only when an invariant spans items. A transaction can include
up to **100 unique items** and 4 MB total, consumes more capacity than ordinary operations, and
cannot span accounts or Regions. `TransactWriteItems` supports a client request token for a
10-minute idempotency window. Design retries for cancellation reasons and do not mix an item with
multiple operations in the same transaction.

### 9.5 Bulk movement, export, and recovery

- **Import from S3** creates a new table and doesn't consume that table's write capacity. Validate
  rejected/error records, keys, indexes, encryption, and counts before cutover.
- **Export to S3** can create full or incremental exports from PITR without consuming table read
  capacity. It is excellent for analytics/migration, but exported files can separate items from a
  transaction, so it is not a transactionally grouped interchange format.
- **PITR** restores a new table. For an accidental write, restore just before the event, compare or
  copy the affected items, and only replace the production endpoint when a full-table cutover is
  actually required.

### 9.6 Keep a Streams consumer moving

A poison record can block a Lambda shard until the stream's **24-hour** record expires. Enable
partial batch responses and, where useful, batch bisection; set finite retry attempts and record
age; send exhausted records to an on-failure S3 destination or SQS/SNS destination for replay.
Alarm on iterator age, errors, throttles, and destination delivery. Preserve event IDs or a
business idempotency key because retried batches can deliver successful records again.

### 9.7 Choose capacity for cost, then re-evaluate

Use **on-demand** while traffic is unknown, highly variable, or operational simplicity is worth
the request premium. Move a stable baseline to **provisioned + Auto Scaling** after measuring daily
RCU/WCU shape; set minimums for known bursts because target tracking reacts after utilization.
Reserved capacity can reduce the bill for committed **single-Region provisioned capacity on
Standard table class**, but it is a one- or three-year billing commitment—not a reserved physical
partition and not capacity for on-demand or replicated global-table writes. Include GSIs, global
replication, backups/PITR, Streams, data transfer, table class, and DAX in the comparison.

---

## 10. When DynamoDB Beats RDS

| Choose **DynamoDB** when… | Choose **RDS/Aurora** when… |
|---------------------------|-----------------------------|
| Massive, unpredictable, internet-scale traffic | Complex `JOIN`s and ad-hoc queries |
| Simple access patterns by key | Strong relational integrity / many relationships |
| Single-digit-ms latency required at any scale | Reporting and flexible SQL filtering |
| Serverless, zero capacity management | Existing SQL app / migration |
| Key-value or document data (sessions, carts, IoT) | OLTP with rich transactions across tables |

> **Key insight**: DynamoDB shines when you know your queries in advance and need scale +
> consistent low latency. It struggles with ad-hoc analytical queries — those belong in a
> relational DB or a warehouse (Redshift).

---

## 11. Common Modeling Mistakes — Hot Partitions

The classic DynamoDB failure is the **hot partition**: a partition key with poor distribution
that funnels most traffic to one physical partition, causing throttling even when total
provisioned capacity looks sufficient.

```
  ❌ BAD partition key: "status" (only 3 values)
     ┌─ "active"  ◄── 95% of traffic hammers ONE partition → throttled
     ├─ "pending"
     └─ "closed"

  ✅ GOOD partition key: "user_id" (millions of values, even spread)
     load distributes across many partitions
```

- ❌ Low-cardinality partition keys (status, country, boolean) → hot partitions.
- ❌ Sequential keys (timestamps, auto-increment) concentrate writes on one partition. Mitigate
  with **write sharding** (append a suffix/hash to spread keys).
- ❌ Trying to model relational data with many `JOIN`-style access patterns — redesign around a
  **single-table** access-pattern model or use a relational DB.
- ❌ Forgetting an item is capped at **400 KB** — store large blobs in **S3** and keep a pointer
  in DynamoDB.

---

## 12. Key Exam Points

- ✅ DynamoDB = **serverless** NoSQL key-value/document, single-digit-ms, replicated across 3 AZs.
- ✅ **On-Demand** for spiky/unknown traffic; **Provisioned + Auto Scaling** for steady traffic.
- ✅ **GSI** = different PK, addable anytime, own capacity, eventual only. **LSI** = same PK,
  different SK, creation-time only, shares capacity, max 5.
- ✅ **Streams + Lambda** = event-driven reactions to table changes.
- ✅ **DAX** = microsecond caching *for DynamoDB*; **ElastiCache** caches anything else.
- ✅ **Global Tables** = multi-Region **active-active**.
- ✅ Reads are **eventually consistent by default**; opt into strong consistency per request.
- ✅ Global Tables support **MREC** (asynchronous, last-writer-wins) and **MRSC** (synchronous,
  RPO 0 for a single-Region failure, more restrictions and conflict errors).
- ✅ PITR retention is configurable from **1–35 days** and every restore creates a new table.
- ✅ Adaptive capacity helps skew but cannot remove a single hot key's partition ceiling.
- ✅ Reserved capacity is a billing commitment for eligible provisioned capacity, not reserved
  infrastructure.

---

## 13. Limits & Quick Facts

- Item size: max **400 KB**.
- GSIs per table: **20**; LSIs per table: **5**.
- PITR recovery period: **1–35 days**; Streams retention: **24 hours**.
- RCU = 4 KB strongly consistent read/sec; WCU = 1 KB write/sec.
- Transaction: up to **100 unique items**, **4 MB**, one account and Region.

---

**Next**: [05_elasticache_and_others.md — ElastiCache & Purpose-Built Databases](05_elasticache_and_others.md)
