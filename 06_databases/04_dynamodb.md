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
| Scaling | Instant, unlimited | Manual, or **Auto Scaling** to a range |
| Best for | Spiky / unpredictable / new apps | Steady, predictable traffic |
| Cost | Higher per request | Cheaper if utilization is high |

**Capacity units** (provisioned mode):
- **RCU (Read Capacity Unit)**: 1 strongly consistent read/sec of up to 4 KB (or 2 eventually
  consistent reads/sec).
- **WCU (Write Capacity Unit)**: 1 write/sec of up to 1 KB.
- **Auto Scaling** adjusts provisioned RCU/WCU between a min and max based on utilization.

⚠️ Exceeding provisioned capacity (without auto scaling/burst) causes
**`ProvisionedThroughputExceededException`** (throttling). On-demand avoids this but costs more
per request.

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
   us-east-1  ◄──── async replication (Streams) ────►  eu-west-1
   read/write                                          read/write
```

- Every replica Region is **read AND write** (active-active), with low replication latency.
- Great for **globally distributed, low-latency** applications and Region-level DR.
- Conflicts resolved **last-writer-wins**.

⚠️ Requires **DynamoDB Streams** enabled. Distinguish from Aurora Global Database, which has one
**writer** Region (read-only secondaries) — DynamoDB Global Tables are **active-active everywhere**.

---

## 8. TTL, Backups & Consistency

**TTL (Time To Live)**: set an expiry timestamp attribute; DynamoDB auto-deletes expired items
(no WCU cost). Ideal for session data, temporary tokens, event logs.

**Backups**:
- **PITR (Point-In-Time Recovery)**: continuous backups, restore to any second in the last
  **35 days**.
- **On-demand backup**: full backups retained until deleted, for long-term/compliance.

**Read consistency** (choose per read):

| | **Eventually consistent** (default) | **Strongly consistent** |
|--|-------------------------------------|-------------------------|
| Latest data? | Maybe stale for ~1s | Always latest |
| Cost | Cheaper (½ RCU) | Costs more (full RCU) |
| Availability | Higher | May fail if a replica is unreachable |

DynamoDB also supports **transactions** (ACID across multiple items) via the `TransactWrite` /
`TransactGet` APIs when you need all-or-nothing operations.

---

## 9. When DynamoDB Beats RDS

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

## 10. Common Modeling Mistakes — Hot Partitions

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

## 11. Key Exam Points

- ✅ DynamoDB = **serverless** NoSQL key-value/document, single-digit-ms, replicated across 3 AZs.
- ✅ **On-Demand** for spiky/unknown traffic; **Provisioned + Auto Scaling** for steady traffic.
- ✅ **GSI** = different PK, addable anytime, own capacity, eventual only. **LSI** = same PK,
  different SK, creation-time only, shares capacity, max 5.
- ✅ **Streams + Lambda** = event-driven reactions to table changes.
- ✅ **DAX** = microsecond caching *for DynamoDB*; **ElastiCache** caches anything else.
- ✅ **Global Tables** = multi-Region **active-active**.
- ✅ Reads are **eventually consistent by default**; opt into strong consistency per request.

---

## 12. Limits & Quick Facts

- Item size: max **400 KB**.
- GSIs per table: **20**; LSIs per table: **5**.
- PITR window: **35 days**; Streams retention: **24 hours**.
- RCU = 4 KB strongly consistent read/sec; WCU = 1 KB write/sec.

---

**Next**: [05_elasticache_and_others.md — ElastiCache & Purpose-Built Databases](05_elasticache_and_others.md)
