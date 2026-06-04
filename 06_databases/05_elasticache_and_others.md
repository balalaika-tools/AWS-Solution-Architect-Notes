# ElastiCache & Purpose-Built Databases

> **Who this is for**: Engineers who know the relational and NoSQL services and now need the
> **in-memory caching** layer plus the family of purpose-built databases. Read
> [01_database_fundamentals.md](01_database_fundamentals.md) first — the decision table maps each
> service below to a data model.

---

## 1. ElastiCache — Managed In-Memory Cache

**Amazon ElastiCache** is a managed **in-memory** data store offering **Valkey**, **Redis OSS**, and
**Memcached** engines. It delivers **microsecond/sub-millisecond** latency by keeping data in RAM,
and is the exam's default answer for "reduce database load" and "store session data."

```
   app ──► ElastiCache (RAM) ──► database (RDS/Aurora/DynamoDB)
           sub-ms cache hit          slower on cache miss
   offloads repeated reads so the database isn't hammered
```

⚠️ ElastiCache is **not** a primary database (Memcached has no persistence; Valkey/Redis OSS
persistence is for recovery, not durability guarantees). It accelerates an existing data store.
If you need a Redis-compatible durable primary database, use **MemoryDB**.

### Networking and security

ElastiCache nodes run in your VPC, normally in private subnets.

- Node-based clusters use a **cache subnet group**; ElastiCache chooses subnets
  and IPs for the cache nodes from that group.
- Access is controlled with **VPC security groups**.
- Allow inbound only from the app tier SG: Valkey/Redis OSS on `6379`, Memcached on
  `11211`, or your configured custom port.
- Do not expose cache ports to the internet. Access from another VPC/on-prem
  requires private connectivity such as peering, Transit Gateway, VPN/DX, and
  appropriate routes/security rules.
- ElastiCache Serverless takes subnet selections directly instead of using a
  cache subnet group resource, but it is still VPC-placed and SG-controlled.

For the broader ENI/security-group map, see
[ENIs, Security Groups & Service Networking](../03_networking/07_enis_security_groups_and_service_networking.md).

---

## 2. Valkey / Redis OSS vs Memcached

The core comparison the exam tests.

| Feature | **Valkey / Redis OSS** | **Memcached** |
|---------|-----------|---------------|
| Data structures | Rich (strings, lists, **sorted sets**, hashes, bitmaps, geospatial, pub/sub) | Simple key-value (strings/objects) only |
| Persistence | **Yes** (snapshots / AOF) | **No** (purely ephemeral) |
| Replication & read replicas | **Yes** | No |
| Multi-AZ + automatic failover | **Yes** | No |
| Backup & restore | **Yes** | No |
| Multithreaded | Single-threaded (per shard) | **Multithreaded** (uses multiple cores) |
| Pub/Sub, transactions | **Yes** | No |
| Scaling | Sharding (cluster mode) + replicas | Add nodes (partition data) |

> **Rule**: Choose **Valkey/Redis OSS** when you need persistence, replication, HA/failover, or
> advanced structures (leaderboards via **sorted sets**, pub/sub, geospatial). For new
> Redis-compatible ElastiCache designs, **Valkey** is the current AWS-forward default, while older
> exam material may simply say "Redis." Choose **Memcached** for a simple, multithreaded,
> horizontally scalable cache where losing the cache is harmless and you just want raw key-value
> speed across many cores.

💡 Memory hooks:
- **V**alkey/**R**edis OSS = rich features, replication, recovery.
- **M**emcached = **M**ultithreaded, **M**inimal, **M**emory-only (no persistence).

---

## 3. Caching Patterns — Lazy Loading vs Write-Through

How the application keeps the cache and database in sync.

**Lazy loading (cache-aside)** — populate the cache only on a miss:

```
  read ──► cache?
           ├─ HIT  ──► return value
           └─ MISS ──► read DB ──► write to cache ──► return value
```
- ✅ Only requested data is cached (memory-efficient).
- ❌ First read of any item is slow (cache miss); cache can serve **stale** data until it expires.

**Write-through** — update the cache every time the database is written:

```
  write ──► DB  AND  ──► cache (kept fresh on every write)
```
- ✅ Cache is always current; reads are fast.
- ❌ Every write incurs cache-write overhead; **infrequently read** data wastes cache space.

> **Key insight**: They're often **combined** — write-through for freshness plus a **TTL** to
> evict stale/cold data, with lazy loading as the fallback on a miss. A **TTL** bounds staleness
> in lazy loading and prevents the cache filling with dead data in write-through.

---

## 4. Caching Use Cases

| Use case | Why a cache fits |
|----------|------------------|
| **Database offload** | Serve hot reads from RAM, cut DB load and cost |
| **Session store** | Fast, shared sessions across a fleet of stateless app servers (use **Valkey/Redis OSS** for persistence/HA) |
| **Leaderboards / counters** | Valkey/Redis OSS **sorted sets** rank in real time |
| **Rate limiting** | Atomic Valkey/Redis OSS counters with TTL |
| **Pub/Sub messaging** | Valkey/Redis OSS channels for lightweight fan-out |

⚠️ Exam clue: "sub-millisecond latency / reduce read load on the database / store user sessions"
→ **ElastiCache** (Valkey/Redis OSS if it must survive node failure or needs HA).

---

## 5. Purpose-Built Databases — What / When / Exam Clue

AWS offers a database per workload. Match the clue to the service.

| Service | Model | What it is | Exam clue |
|---------|-------|------------|-----------|
| **Neptune** | Graph | Managed graph DB (Gremlin/SPARQL/openCypher) | "social network," "recommendation engine," "fraud detection," "relationships between entities" |
| **DocumentDB** | Document | Managed **MongoDB-compatible** database | "MongoDB workload, fully managed, scale storage independently" |
| **Keyspaces** | Wide-column | Managed **Apache Cassandra-compatible**, serverless | "Cassandra / CQL workload, serverless, no cluster to manage" |
| **Timestream** | Time-series | Serverless time-series DB | "IoT sensor data," "metrics over time," "time-series at scale" |
| **QLDB** | Ledger | Immutable, cryptographically **verifiable** ledger | "immutable, verifiable transaction log / audit history (single trusted owner)" |
| **MemoryDB** | In-memory | **Valkey/Redis OSS-compatible**, **durable** primary database | "Valkey/Redis API but as a durable primary DB," "microsecond reads + single-digit-ms writes, multi-AZ durable" |
| **RDS on Outposts** | Relational | RDS running on **AWS Outposts** (on-premises) | "managed relational DB that must stay on-premises for latency/data-residency" |

💡 Quick disambiguations:
- **DocumentDB** (MongoDB) vs **DynamoDB** (AWS-native NoSQL): pick DocumentDB only when the
  question explicitly says **MongoDB compatibility**; otherwise DynamoDB.
- **MemoryDB** vs **ElastiCache for Valkey/Redis OSS**: MemoryDB is a **durable primary database**
  (data survives without a backing store). ElastiCache is a **cache** in front of another database.
- **QLDB** (single trusted central authority, immutable audit) vs blockchain/Managed Blockchain
  (multiple parties, decentralized trust). The exam phrase for QLDB is "verifiable, immutable,
  cryptographically chained history owned by **one** entity."

---

## 6. Redshift Is Analytics — Not Here

**Amazon Redshift** is a petabyte-scale **OLAP data warehouse** (columnar storage, MPP). It is an
*analytics* service, not a transactional database, so it is covered separately:
**[Analytics Services](../16_analytics_ml_survey/01_analytics_services.md)**.

⚠️ Exam clue: "data warehouse," "complex analytical queries / BI / reporting over huge datasets,"
"OLAP" → **Redshift**, never RDS or DynamoDB.

---

## 7. Key Exam Points

- ✅ **ElastiCache** = sub-ms in-memory cache; default answer for "reduce DB load" and "session
  store."
- ✅ ElastiCache is private VPC networking: subnet/subnet-group placement plus SG inbound from the app SG.
- ✅ **Valkey/Redis OSS** = persistence, replication, Multi-AZ failover, advanced structures
  (sorted sets, pub/sub). **Memcached** = simple, multithreaded, no persistence.
- ✅ **Lazy loading** caches on miss (may be stale); **write-through** updates cache on every
  write (always fresh); combine with **TTL**.
- ✅ **Neptune** = graph; **DocumentDB** = MongoDB; **Keyspaces** = Cassandra; **Timestream** =
  time-series; **QLDB** = ledger; **MemoryDB** = durable Valkey/Redis OSS primary DB.
- ✅ **DynamoDB DAX** caches DynamoDB specifically; **ElastiCache** caches anything.
- ✅ **Redshift** = OLAP data warehouse (analytics), not in this section.

---

## 8. Common Mistakes

- ❌ Picking **Memcached** when the requirement needs persistence, replication, or HA — that's
  **Valkey/Redis OSS**.
- ❌ Treating **ElastiCache** as a durable primary datastore — use **MemoryDB** if you need a
  durable Valkey/Redis OSS-compatible *database*.
- ❌ Choosing **DocumentDB** for generic NoSQL when there's no MongoDB requirement — use
  **DynamoDB**.
- ❌ Routing analytics/data-warehouse questions anywhere but **Redshift**.
- ❌ Confusing **QLDB** (single-owner immutable ledger) with **Managed Blockchain**
  (multi-party decentralized).

---

## 9. Limits & Quick Facts

- ElastiCache Valkey/Redis OSS: supports read replicas + Multi-AZ with automatic failover; backups via
  snapshots.
- ElastiCache Memcached: no persistence, no replication, scales by adding nodes (multithreaded).
- MemoryDB: durable, multi-AZ, microsecond reads / single-digit-ms writes.
- Purpose-built services are each optimized for one data model — match the model to the clue.

---

**Next**: [Load Balancing Concepts](../07_ha_scaling/01_load_balancing_concepts.md)
