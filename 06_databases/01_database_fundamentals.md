# Database Fundamentals

> **Who this is for**: Engineers preparing for SAA-C03 who need the data-modeling and scaling
> vocabulary *before* touching AWS database services. No database theory assumed beyond having
> written a few SQL queries. Every later file in this section (RDS, Aurora, DynamoDB,
> ElastiCache) refers back to the concepts defined here.

---

## 1️⃣ Relational vs Non-Relational

The first fork in every database decision: does your data fit neatly into **tables with a fixed
schema and relationships**, or is it better stored as flexible documents, key-value pairs, or
graphs?

**Relational (SQL)** databases store data in tables (rows and columns) with a predefined schema.
Tables relate to each other through **foreign keys**, and you query them with **SQL**. Strong
consistency and `JOIN`s across tables are the headline features.

```
RELATIONAL: data is split across related tables (normalized)

  customers                     orders
  ┌────┬─────────┐             ┌────┬─────────────┬────────┐
  │ id │ name    │             │ id │ customer_id │ total  │
  ├────┼─────────┤             ├────┼─────────────┼────────┤
  │ 1  │ Alice   │◄────────────┤ 90 │ 1           │ 42.00  │
  │ 2  │ Bob     │   FK link    │ 91 │ 2           │ 18.50  │
  └────┴─────────┘             └────┴─────────────┴────────┘
        JOIN customers + orders to see "Alice ordered $42.00"
```

**Non-relational (NoSQL)** databases drop the rigid table model. They come in four common
flavors, each optimized for a different access pattern:

| NoSQL type | Stores data as | Good for | AWS service |
|------------|----------------|----------|-------------|
| **Key-value** | Key → opaque value | Ultra-fast lookups by key (sessions, carts) | DynamoDB, ElastiCache |
| **Document** | JSON/BSON documents | Flexible, nested records (catalogs, profiles) | DynamoDB, DocumentDB |
| **Graph** | Nodes + edges | Relationships (social graphs, fraud rings) | Neptune |
| **In-memory** | Key → value in RAM | Microsecond caching, leaderboards | ElastiCache, MemoryDB |

> **Key insight**: "NoSQL" does not mean "no schema" — it means *schema-on-read*. You can store
> items with different attributes in the same table, and the application decides how to interpret
> them. The trade-off is that the database can't enforce relationships or do `JOIN`s for you.

---

## 2️⃣ OLTP vs OLAP

Two fundamentally different *workloads*. The exam expects you to route a question to the right
service based on which workload it describes.

| | **OLTP** (Online Transaction Processing) | **OLAP** (Online Analytical Processing) |
|--|------------------------------------------|------------------------------------------|
| Purpose | Run the business (transactions) | Analyze the business (reports) |
| Operations | Many small reads/writes | Few large, complex aggregations |
| Example query | "Insert this order" / "Get user 123" | "Total revenue by region per quarter" |
| Row count touched | One or a few rows | Millions of rows |
| Optimized for | Low latency, high concurrency | Scanning huge datasets |
| AWS services | RDS, Aurora, DynamoDB | **Redshift**, Athena, EMR |

⚠️ Exam clue: words like *data warehouse*, *analytics*, *business intelligence*, *complex
reporting over petabytes* point to **OLAP / Redshift** — not RDS. Redshift is covered under
analytics: see [Analytics Services](../16_analytics_ml_survey/01_analytics_services.md).

---

## 3️⃣ ACID vs BASE

These are two consistency philosophies. Relational databases prize **ACID**; many NoSQL systems
relax it to **BASE** in exchange for scale and availability.

**ACID** — the guarantees of a reliable transaction:

| Property | Meaning |
|----------|---------|
| **Atomicity** | All steps in a transaction succeed, or none do (no half-completed transfers). |
| **Consistency** | A transaction moves the DB from one valid state to another (constraints hold). |
| **Isolation** | Concurrent transactions don't see each other's partial work. |
| **Durability** | Once committed, data survives crashes. |

**BASE** — the trade-off many distributed NoSQL systems make:

- **Basically Available** — the system stays responsive even during partial failures.
- **Soft state** — data may be in flux; replicas may temporarily disagree.
- **Eventual consistency** — given no new writes, all replicas *converge* to the same value.

```
STRONG (ACID)                        EVENTUAL (BASE)
  write ──► all replicas updated       write ──► one replica updated
            before read succeeds                 read may see old value
  ✅ always latest                                for a moment, then converges
  ❌ higher latency, harder to scale    ✅ fast, scales horizontally
                                        ⚠️ stale reads possible
```

> **Rule**: Use strong/ACID consistency when correctness is non-negotiable (banking, inventory).
> Accept eventual consistency when you need massive scale and a brief stale read is harmless
> (social feeds, view counts). DynamoDB lets you choose **per read request**.

---

## 4️⃣ Normalization Basics

**Normalization** is the relational practice of splitting data into multiple tables to eliminate
redundancy. Each fact is stored exactly once; tables are joined at query time.

```
❌ DENORMALIZED (data repeated)        ✅ NORMALIZED (data referenced)
  orders                                orders            customers
  ┌────┬──────────┬───────┐            ┌────┬─────────┐   ┌────┬─────────┐
  │ id │ customer │ total │            │ id │ cust_id │   │ id │ name    │
  ├────┼──────────┼───────┤            ├────┼─────────┤   ├────┼─────────┤
  │ 90 │ Alice    │ 42.00 │            │ 90 │ 1       │   │ 1  │ Alice   │
  │ 91 │ Alice    │ 18.50 │            │ 91 │ 1       │   └────┴─────────┘
  └────┴──────────┴───────┘            └────┴─────────┘
   "Alice" repeated → update anomaly    one place to update the name
```

- **Normalized** (relational/OLTP): less storage, no update anomalies, but reads require `JOIN`s.
- **Denormalized** (NoSQL/OLAP): data duplicated so a single read returns everything — faster
  reads, no joins, but writes must update every copy.

💡 NoSQL data modeling is essentially *deliberate denormalization*: you shape the data around the
queries you'll run, not around tidy relationships.

---

## 5️⃣ Vertical vs Horizontal Scaling

When a database can't keep up, you scale it one of two ways.

```
VERTICAL (scale up)                  HORIZONTAL (scale out)
  ┌──────┐   ┌────────────┐          ┌────┐ ┌────┐ ┌────┐ ┌────┐
  │ db   │──►│ BIGGER db  │          │ db │ │ db │ │ db │ │ db │
  │ 4 CPU│   │ 32 CPU     │          └────┘ └────┘ └────┘ └────┘
  └──────┘   └────────────┘          add more nodes, spread the load
  one machine, more power            many machines, shared work
```

| | **Vertical (scale up)** | **Horizontal (scale out)** |
|--|-------------------------|----------------------------|
| How | Bigger instance (more CPU/RAM) | More nodes |
| Ceiling | Limited by the largest instance | Effectively unlimited |
| Downtime | Often needs a reboot/resize | Add nodes live |
| Fits | Relational (RDS instance class) | NoSQL (DynamoDB partitions) |

> **Key insight**: Relational databases scale *up* easily but *out* with difficulty (sharding is
> hard). NoSQL databases like DynamoDB are built to scale *out* automatically. This is the core
> reason the exam pushes you toward DynamoDB for "unpredictable, massive, internet-scale" traffic.

---

## 6️⃣ Replication, Read Replicas & Failover

Three related concepts that recur in every managed-database file.

**Replication** = copying data to additional database instances. Two motivations, often confused:

```
READ REPLICA (scaling)               STANDBY (high availability)
  ┌─────────┐  async copy            ┌─────────┐  sync copy   ┌─────────┐
  │ PRIMARY │────────────► replica   │ PRIMARY │─────────────►│ STANDBY │
  │ (writes)│             (reads)    │ (active)│              │ (idle,  │
  └─────────┘                        └─────────┘   failover   │  hot)   │
  offload SELECTs to replicas         promote standby ◄───────┘ on crash
  app reads scale; replica may lag    same endpoint, no read traffic
```

- **Read replica**: an **asynchronous** copy used to **offload read traffic** (scaling). It has
  its own endpoint and may lag slightly behind the primary. Promoting it is a manual operation.
- **Standby (primary/standby failover)**: a **synchronous** copy kept in lockstep for **high
  availability**. It serves *no* traffic until the primary fails, then it is **automatically
  promoted** and the application keeps using the same endpoint.

> **Rule**: Replication for **scaling reads** ≠ replication for **availability**. AWS implements
> these as two different features — **Read Replicas** (scaling) and **Multi-AZ** (HA). The exam
> tests this distinction constantly; see [RDS](02_rds.md).

---

## 7️⃣ Choosing a Database — Decision Table

The single most useful table in this section. Match the scenario to the model.

| If you need… | Pick this model | AWS service |
|--------------|-----------------|-------------|
| Complex queries, `JOIN`s, transactions, strict schema (OLTP) | **Relational** | RDS / Aurora |
| Data warehouse / analytics over huge datasets (OLAP) | **Columnar / analytics** | Redshift |
| Massive scale, simple key lookups, single-digit-ms latency, serverless | **Key-value** | DynamoDB |
| Flexible nested documents, MongoDB compatibility | **Document** | DynamoDB / DocumentDB |
| Relationships between entities (social, recommendations, fraud) | **Graph** | Neptune |
| Sub-millisecond reads, caching, leaderboards, sessions | **In-memory** | ElastiCache / MemoryDB |
| Time-series (IoT sensor data, metrics) | **Time-series** | Timestream |
| Immutable, cryptographically verifiable audit log | **Ledger** | QLDB |

💡 Quick heuristic for the exam:
- "Relational" / "SQL" / "ACID transactions" → **RDS or Aurora**
- "Millions of requests, predictable single-digit-ms, no servers to manage" → **DynamoDB**
- "Cache" / "reduce database load" / "sub-millisecond" / "session store" → **ElastiCache**
- "Data warehouse" / "analytics" → **Redshift**

---

## 8️⃣ Key Exam Points

- ✅ **Relational** = fixed schema, `JOIN`s, ACID, scales **up** → RDS/Aurora.
- ✅ **NoSQL** = flexible schema, denormalized, scales **out** → DynamoDB.
- ✅ **OLTP** = transactions (RDS/Aurora/DynamoDB); **OLAP** = analytics (Redshift).
- ✅ **ACID** = strong consistency; **BASE** = eventual consistency for scale/availability.
- ✅ **Read Replica** = async, scales reads, own endpoint. **Standby/Multi-AZ** = sync, HA only,
  same endpoint, automatic failover.
- ✅ Relational scales **vertically** (bigger instance); NoSQL scales **horizontally** (more nodes).

---

## 9️⃣ Common Mistakes

- ❌ Choosing RDS for "internet-scale, unpredictable, key-based" workloads — that's DynamoDB.
- ❌ Choosing DynamoDB when the question requires complex `JOIN`s or multi-table transactions —
  that's relational.
- ❌ Thinking a **Read Replica** provides high availability — it doesn't fail over automatically
  and may have stale data. Use **Multi-AZ** for HA.
- ❌ Routing analytics/data-warehouse questions to RDS instead of **Redshift**.
- ❌ Assuming "NoSQL = no schema." It's schema-on-read; you still design around access patterns.

---

**Next**: [02_rds.md — Amazon RDS](02_rds.md)
