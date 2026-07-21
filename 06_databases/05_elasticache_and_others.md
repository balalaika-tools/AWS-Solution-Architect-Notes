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
- ✅ A successful coordinated write leaves the cache warm and current; reads are fast.
- ❌ Every write incurs cache-write overhead; **infrequently read** data wastes cache space.

The database and cache do not normally share one atomic transaction. Define what happens when the
database succeeds and the cache update fails: invalidate the key, retry asynchronously, or let a
short TTL bound staleness. Never acknowledge a business write only because the cache accepted it.

> **Key insight**: They're often **combined** — write-through for a warm cached copy plus a **TTL**
> to evict stale/cold data, with lazy loading as the fallback on a miss. A **TTL** bounds staleness
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
| **DocumentDB** | Document | Managed database with **MongoDB API compatibility** | "MongoDB workload, fully managed, scale storage independently" |
| **Keyspaces** | Wide-column | Managed **Apache Cassandra-compatible**, serverless | "Cassandra / CQL workload, serverless, no cluster to manage" |
| **Timestream for InfluxDB** | Time-series | Managed InfluxDB instances for real-time time-series applications | "InfluxDB APIs," "IoT sensor data," "operational metrics over time" |
| **Timestream for LiveAnalytics** | Time-series | Serverless time-series analytics service available to existing customers; closed to new customers on June 20, 2025 | Existing deployed LiveAnalytics workload; new customers evaluate Timestream for InfluxDB or another current time-series path |
| **OpenSearch Service** | Search/index | Managed text search, relevance, and log/observability indexes | "full-text search," "facets," "log exploration," "relevance ranking" |
| **MemoryDB** | In-memory | **Valkey/Redis OSS-compatible**, **durable** primary database | "Valkey/Redis API but as a durable primary DB," "microsecond reads + single-digit-ms writes, multi-AZ durable" |
| **RDS on Outposts** | Relational | RDS running on **AWS Outposts** (on-premises) | "managed relational DB that must stay on-premises for latency/data-residency" |

💡 Quick disambiguations:
- **DocumentDB** (MongoDB) vs **DynamoDB** (AWS-native NoSQL): pick DocumentDB only when the
  question explicitly says **MongoDB compatibility**; otherwise DynamoDB. Compatibility does
  not mean behavioral identity—assess commands, drivers, indexes, aggregation, transactions,
  retry semantics, and result ordering against the exact DocumentDB version.
- **MemoryDB** vs **ElastiCache for Valkey/Redis OSS**: MemoryDB is a **durable primary database**
  (data survives without a backing store). ElastiCache is a **cache** in front of another database.

> **QLDB status**: Amazon QLDB reached end of support on **July 31, 2025**. Do not design a new
> workload around it. For an append-only/audited ledger owned by one organization, evaluate a
> relational design such as Aurora PostgreSQL with permissions, history tables, cryptographic
> evidence where required, and immutable exports to S3 Object Lock. That replacement is an
> architecture and compliance decision, not a one-click engine swap.

---

## 6. Redshift Is Analytics — Not Here

**Amazon Redshift** is a petabyte-scale **OLAP data warehouse** (columnar storage, MPP). It is an
*analytics* service, not a transactional database, so it is covered separately:
**[Analytics Services](../16_analytics_ml_survey/01_analytics_services.md)**.

⚠️ Exam clue: "data warehouse," "complex analytical queries / BI / reporting over huge datasets,"
"OLAP" → **Redshift**, never RDS or DynamoDB.

---

## 7. Purpose-Built Migration & Modernization

Purpose-built does not mean "one database per noun." Start with access patterns and invariants,
then price the **data duplication, consistency boundary, operations, and exit path**. A common
modernization keeps orders/payments in a relational system of record and publishes changes to a
search, graph, cache, or analytics projection.

### 7.1 Decision matrix

| Target | Strong fit | Durability / consistency question | Operational ownership | Exit and refactor cost |
|--------|------------|-----------------------------------|-----------------------|------------------------|
| **RDS / Aurora** | SQL OLTP, joins, constraints, multi-row transactions | ACID primary; define replica/read-after-write behavior and Multi-AZ/Region RPO | Engine tuning, schema, queries, connections; AWS owns hosts/backups/HA machinery | Lowest for same-engine RDS; compatibility, extensions, and stored code still require tests |
| **DynamoDB** | Stable key-based access at large/variable scale | Durable multi-AZ service; per-read consistency and Global Table MREC/MRSC are explicit choices | Keys, indexes, hot-key control, quotas, Streams consumers | High if leaving: denormalized items and access-pattern APIs must be translated |
| **Neptune** | Deep relationship traversal, fraud rings, knowledge graphs | Choose transactional and replica-read behavior for the graph engine; design cross-Region recovery | Graph model/query tuning, bulk load, replicas/backups | Graph schema and Gremlin/SPARQL/openCypher queries are specialized |
| **DocumentDB** | Existing MongoDB API workload after compatibility proof | Durable cluster storage; reader endpoints can be stale, so route correctness-critical reads deliberately | Instance/cluster sizing, indexes, profiler/queries, compatibility upgrades | MongoDB API helps but unsupported/different behavior makes later movement nonzero work |
| **Timestream for InfluxDB** | High-rate timestamped metrics and retention/downsampling queries | Define backup/restore, retention, and acceptable delayed/lost telemetry | Instance sizing/HA, InfluxDB schema, compaction/cardinality | Line protocol/Flux/SQL ecosystem choices and retention transforms affect portability |
| **OpenSearch Service** | Full-text/relevance, faceting, observability search | Treat indexes as rebuildable projections when a source of truth exists; replicas/snapshots aren't relational constraints | Shards, mappings, index lifecycle, ingestion lag, cluster health | Index mappings/query DSL and reindex time can make exit expensive |
| **ElastiCache** | Disposable acceleration, sessions recoverable elsewhere | Cache misses/eviction and node loss must be safe; persistence aids recovery but is not the source-of-truth contract | TTL, invalidation, memory/eviction, hot keys, failover | Usually low when cache-aside is encapsulated; session semantics can create coupling |
| **MemoryDB** | Valkey-compatible data is the durable primary with very low latency | Successful writes use a durable Multi-AZ transaction log; reads from the primary are strongly consistent while replica reads are eventual | Data structures, shards/replicas, memory, backups, failover | Redis/Valkey structures and commands can deeply couple the application |

### 7.2 Modernization patterns and cutovers

**Relational to managed relational** — use same-engine RDS for the smallest change, Aurora only
after compatibility/performance testing, and RDS Custom only for a proven host/database dependency.
Seed with native restore or DMS full load, run CDC for a short outage, validate business totals and
unsupported objects, freeze writes, drain lag, switch the endpoint, and keep an explicit rollback
boundary. See the full framework in
[Database Fundamentals](01_database_fundamentals.md).

**Relational to DynamoDB** — first replace ad-hoc SQL with a finite list of access patterns. Build
the key/index model, dual-read and compare during a shadow period, then backfill plus CDC. Avoid
uncoordinated application **dual writes**; publish an outbox/change stream so retries are observable
and idempotent. Move one bounded aggregate at a time rather than rewriting the entire database.

**Fraud graph projection** — keep authoritative customers/payments in the OLTP database. Stream
identity, device, address, and transaction edges into Neptune; rebuild/backfill the graph, compare
known fraud queries, monitor ingestion lag, then let the decision service query the graph. A graph
alert should carry the source IDs so investigators can verify it against the system of record.

**MongoDB API migration to DocumentDB** — run a compatibility assessment and replay production
queries against a representative cluster. Convert unsupported indexes/operators, test transactions,
ordering and retry behavior, bulk-load data, then use a supported ongoing-replication method until
lag is inside the cutover threshold. "MongoDB-compatible" is a starting hypothesis, not acceptance.

**IoT/time-series split** — keep device identity and configuration in DynamoDB or relational
storage; put high-volume measurements in a current time-series target; archive raw, portable data
to S3. Test tag/series cardinality, retention/downsampling, late/out-of-order points, backup/restore,
and regional recovery. Do not select Timestream for LiveAnalytics for a new AWS customer.

**Search projection** — publish catalog/content changes from the system of record to OpenSearch
through an idempotent pipeline. Version mappings and index templates, build a new index, validate
document counts/relevance/security filters, then atomically switch an alias. Retain the source so a
corrupt mapping or missed event can be replayed; don't use the search index as the only copy of an
order or payment.

**Cache or durable in-memory store** — add ElastiCache behind a cache-aside abstraction and test a
cold-cache restart, stampede protection, TTL jitter, invalidation, failover, and stale-data safety.
Choose MemoryDB only when Valkey structures themselves are authoritative and its persistence,
consistency, backup, and recovery behavior meet the data contract. A session store may use either:
ElastiCache when sessions can be recreated, MemoryDB when losing accepted session writes is not
acceptable.

### 7.3 Deployment safety checklist

For every projection or new store: define the source of truth, ordering and idempotency key,
acceptable replication lag, reconciliation query, replay point, schema/version strategy, regional
recovery, encryption and access control, cost alarm, and decommission criterion. Before deleting the
old path, export data in a portable form and prove restore/rebuild—not just backup creation.

---

## 8. Key Exam Points

- ✅ **ElastiCache** = sub-ms in-memory cache; default answer for "reduce DB load" and "session
  store."
- ✅ ElastiCache is private VPC networking: subnet/subnet-group placement plus SG inbound from the app SG.
- ✅ **Valkey/Redis OSS** = persistence, replication, Multi-AZ failover, advanced structures
  (sorted sets, pub/sub). **Memcached** = simple, multithreaded, no persistence.
- ✅ **Lazy loading** caches on miss (may be stale); **write-through** warms the cache on every
  write but needs a partial-failure policy; combine with **TTL**.
- ✅ **Neptune** = graph; **DocumentDB** = MongoDB API compatibility; **Keyspaces** = Cassandra;
  **Timestream for InfluxDB** = current managed InfluxDB path; **OpenSearch** = search index;
  **MemoryDB** = durable Valkey/Redis OSS-compatible primary DB.
- ✅ QLDB reached end of support in 2025; don't choose it for a new ledger design. Timestream for
  LiveAnalytics is closed to new customers.
- ✅ A search/cache/graph copy is commonly a **projection** from an authoritative OLTP store. Use
  idempotent CDC, measure lag, reconcile, and retain a replay/rebuild path.
- ✅ **DynamoDB DAX** caches DynamoDB specifically; **ElastiCache** caches anything.
- ✅ **Redshift** = OLAP data warehouse (analytics), not in this section.

---

## 9. Common Mistakes

- ❌ Picking **Memcached** when the requirement needs persistence, replication, or HA — that's
  **Valkey/Redis OSS**.
- ❌ Treating **ElastiCache** as a durable primary datastore — use **MemoryDB** if you need a
  durable Valkey/Redis OSS-compatible *database*.
- ❌ Choosing **DocumentDB** for generic NoSQL when there's no MongoDB requirement — use
  **DynamoDB**.
- ❌ Routing analytics/data-warehouse questions anywhere but **Redshift**.
- ❌ Treating DocumentDB API compatibility as identical MongoDB behavior without workload tests.
- ❌ Using OpenSearch as the only system of record for payments/orders rather than a rebuildable
  search projection.
- ❌ Designing a new QLDB or Timestream for LiveAnalytics deployment after their availability
  changes.
- ❌ Calling application dual writes a migration strategy without ordering, idempotency,
  reconciliation, or replay.

---

## 10. Limits & Quick Facts

- ElastiCache Valkey/Redis OSS: supports read replicas + Multi-AZ with automatic failover; backups via
  snapshots.
- ElastiCache Memcached: no persistence, no replication, scales by adding nodes (multithreaded).
- MemoryDB: durable, multi-AZ, microsecond reads / single-digit-ms writes.
- Purpose-built services are each optimized for one data model — match the model to the clue.
- QLDB end of support: **July 31, 2025**. Timestream for LiveAnalytics closed to new customers:
  **June 20, 2025**.

---

**Next**: [Load Balancing Concepts](../07_ha_scaling/01_load_balancing_concepts.md)
