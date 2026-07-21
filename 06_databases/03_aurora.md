# Amazon Aurora — Cloud-Native Relational

> **Who this is for**: Engineers who know [RDS](02_rds.md) and want to understand why AWS built a
> separate, cloud-native relational engine — and when the exam expects you to pick Aurora over
> plain RDS.

---

## 1. What Aurora Is

**Amazon Aurora** is AWS's cloud-native relational database engine, **MySQL- and
PostgreSQL-compatible** (you choose one at creation). It speaks the same wire protocol as those
engines — existing drivers and tools work — but the **storage layer is completely re-engineered**
for the cloud.

AWS claims up to **5x the throughput of MySQL** and **3x of PostgreSQL** on comparable hardware,
at roughly the price of a commercial database. It is part of the RDS family (you manage it through
RDS), but its architecture is fundamentally different.

> **Key insight**: The magic of Aurora is the **decoupling of compute from storage**. Database
> instances (compute) read and write to a shared, distributed storage volume. Adding a read
> replica doesn't copy data — replicas read the *same* shared storage. This is why Aurora
> failover and replication are so much faster than RDS.

---

## 2. The Storage Model — 6 Copies Across 3 AZs

Aurora storage is a distributed, self-healing, auto-expanding volume shared by all instances in
the cluster.

```
        ┌──────────── Aurora Cluster Volume (shared) ──────────────┐
        │   AZ-a            AZ-b            AZ-c                   │
        │  ┌────┐┌────┐    ┌────┐┌────┐    ┌────┐┌────┐            │
        │  │copy││copy│    │copy││copy│    │copy││copy│  = 6 copies│
        │  └────┘└────┘    └────┘└────┘    └────┘└────┘            │
        └────────▲───────────────────────────────▲─────────────────┘
                 │ writes                         │ reads
          ┌──────┴──────┐                  ┌──────┴──────┐
          │   WRITER    │                  │   READER    │  (up to 15)
          │ (1 instance)│                  │  instances  │
          └─────────────┘                  └─────────────┘
```

- **6 copies of data across 3 AZs** (2 per AZ).
- **Self-healing**: continuously scans for and repairs disk errors automatically.
- Writes need **4 of 6** copies to acknowledge; reads need **3 of 6** → survives the loss of an
  entire AZ (and more) without data loss or write interruption.
- Storage **auto-grows** in 10 GB increments up to **128 TiB** — no pre-provisioning.

⚠️ This shared-storage design is why you should not reason about Aurora the way you reason about
RDS read replicas. There's no per-replica copy of the data to fall behind by a full sync.

---

## 3. Endpoints & Read Replicas

Aurora supports up to **15 Aurora Replicas** (plus the writer), and replication lag is typically
**single-digit milliseconds** because replicas read the shared volume rather than receiving a
replicated stream.

Aurora gives you managed endpoints so the app doesn't track individual instances:

| Endpoint | Points to | Use for |
|----------|-----------|---------|
| **Writer (cluster) endpoint** | The current primary/writer | All writes |
| **Reader endpoint** | Load-balances across all replicas | Reads (auto-distributes) |
| **Custom endpoints** | A chosen subset of instances | Routing to specific instance classes |
| **Instance endpoints** | One specific instance | Diagnostics/fine control |

```
   app writes ──► WRITER endpoint ──► current primary
   app reads  ──► READER endpoint ──► [replica][replica][replica]  (load-balanced)
```

**Fast failover**: if the writer fails, Aurora promotes a replica in **~30 seconds** (often
faster), and the **writer endpoint** automatically points to the new primary. No data copy is
needed because storage is shared. You can set **failover priority tiers** to control which
replica is promoted first.

> **Rule**: Use the **reader endpoint** for read scaling (it load-balances for you) and the
> **writer endpoint** for writes. Don't hard-code instance endpoints in app config.

---

## 4. VPC Placement and Security Groups

Aurora is part of the RDS family, so its network placement looks like RDS even
though the storage architecture is different.

- An Aurora cluster lives in a **VPC** and uses a **DB subnet group** with subnets
  in at least two AZs.
- Aurora DB instances receive managed network interfaces/IPs from those subnets.
- The cluster/instances use **VPC security groups**. The normal rule is to allow
  the database port only from the app tier SG, Lambda SG, or ECS task SG.
- Applications connect through the **writer** and **reader** DNS endpoints, not
  through instance IPs.
- Aurora Serverless v2 is still VPC-placed relational database compute. It is
  serverless capacity scaling, not a public internet database.

For the broader ENI/security-group map, see
[ENIs, Security Groups & Service Networking](../03_networking/07_enis_security_groups_and_service_networking.md).

---

## 5. Aurora Serverless v2

**Aurora Serverless v2** automatically scales database **compute capacity** up and down based on
load, measured in **ACUs (Aurora Capacity Units)**.

- Scales in **0.5-ACU increments** without the coarse capacity changes of Serverless v1. Each ACU
  represents about **2 GiB of memory** with corresponding CPU and networking.
- You set a **minimum and maximum ACU** range; current supported configurations can reach **256
  ACUs**. The allowed range depends on the engine version and platform.
- Supported engine versions can use a **0-ACU minimum** and auto-pause; other configurations have
  a **0.5-ACU minimum**. Resume latency and feature/version support must be checked before treating
  scale-to-zero as a production design.
- Ideal for **variable, unpredictable, or spiky** workloads, dev/test, and multi-tenant apps
  where provisioning a fixed instance is wasteful.

💡 Exam clue: "relational database with **unpredictable / intermittent** load, don't want to
manage capacity" → **Aurora Serverless v2**.

### Capacity planning, not capacity avoidance

Serverless v2 still needs deliberate bounds:

1. Estimate the working set, connection count, and peak CPU/throughput. Set the **minimum** high
   enough to retain the hot buffer cache and accept the normal connection baseline.
2. Set the **maximum** high enough for a tested peak, failover, maintenance, and reader promotion.
   Some database configuration limits are derived from maximum ACUs, so reboot when the engine
   marks a changed parameter as pending.
3. Load-test the climb from the configured minimum. Scaling is faster when the instance is already
   at a higher capacity; a very low floor can lengthen recovery from a sudden surge.
4. Put latency-sensitive readers at a nonzero floor. A reader promoted during failover should have
   enough capacity to become the writer.
5. Monitor `ServerlessDatabaseCapacity`, `ACUUtilization`, connections, cache hit rate, and latency.
   A cluster repeatedly touching its maximum is under-sized even though it is called serverless.

⚠️ Scale-to-zero saves idle cost but discards the warm database cache and adds resume latency. It is
suited to interruptible dev/test or infrequent workloads, not an automatic choice for a production
API with a strict first-query SLO.

---

## 6. Aurora Global Database

**Aurora Global Database** spans multiple Regions for disaster recovery and low-latency global
reads.

```
   PRIMARY Region (us-east-1)            SECONDARY Region (eu-west-1)
   ┌──────────────────────┐   <1s       ┌──────────────────────┐
   │ writer + readers     │────────────►│ read-only replicas   │
   │ (read/write)         │  replication│ (promote on disaster)│
   └──────────────────────┘             └──────────────────────┘
                            up to 10 secondary Regions
```

- One **primary Region** (read/write) replicates to up to **10 secondary Regions** (read-only).
- Typical cross-Region replication lag **< 1 second**.
- Dedicated replication infrastructure — does not consume the primary's compute.

⚠️ Distinguish from a cross-Region **read replica** in plain RDS: Aurora Global Database is purpose-
built for fast DR and global low-latency reads, with sub-second typical lag and managed promotion.

### Planned switchover versus unplanned failover

| | **Managed switchover** | **Managed failover** |
|--|------------------------|----------------------|
| Use | Planned Region rotation, maintenance, or DR exercise | Primary Region is unavailable or unsafe |
| Data objective | RDS waits for synchronization: **RPO 0** | Usually nonzero RPO, determined by replication lag at failure |
| Region choice | Chosen synchronized secondary | Prefer an available secondary with the least lag |
| Application impact | Writes pause; connections drop and retry | Longer/variable interruption while the topology changes |

The managed operation reverses replication so the promoted Region becomes the primary. The
**global writer endpoint**, when supported by the cluster version, follows the current primary;
otherwise update regional endpoints through application configuration or DNS. In either case,
existing TCP sessions do not move. Use a low DNS-cache TTL, close stale pools, reconnect, and retry
only idempotent transactions.

For an unplanned event, first **fence the former primary** from application writes to avoid two
independent histories if it becomes reachable again. Observe `AuroraGlobalDBReplicationLag`, choose
the recovery Region, perform the managed failover, verify the writer and critical data, then open
traffic gradually. Rebuild/rejoin the old Region only after deciding which history is authoritative.
The service promotion time is not the application's RTO; DNS, client caches, connection recovery,
schema dependencies, secrets, and regional infrastructure are all on the critical path.

### Write forwarding is still single-writer

Aurora can forward supported writes issued in a secondary Region to the writer in the primary
Region for both Aurora MySQL and Aurora PostgreSQL on supported versions. This can simplify a
global application, but it is **not multi-primary or active-active**: every write still crosses the
network and commits in one primary Region.

Choose the forwarding consistency mode deliberately. Stronger read-after-write behavior requires
more coordination and latency; eventual behavior can return an older value. Test transaction and
SQL limitations, monitor forwarding latency, and keep a direct-primary path for write-heavy or
latency-sensitive operations. A Region-evacuation plan still needs failover and reconnection.

---

## 7. Backtrack, PITR & Cloning

**Backtrack** (Aurora MySQL) rewinds the cluster to a previous point in time **in place** —
**without restoring from a backup** or creating a new cluster. Great for quickly undoing a bad
deployment or accidental bulk delete.

- ⚠️ Backtrack ≠ backup. It rewinds the *existing* cluster within a configured window (up to 72
  hours); it does not replace snapshots/PITR for long-term recovery.

| Recovery method | Result | Operational choice |
|-----------------|--------|--------------------|
| **Backtrack** | Rewinds the existing Aurora MySQL cluster in place | Fast undo within the configured window; pause applications and accept that later changes are removed |
| **PITR** | Creates a **new** cluster at a chosen restorable time | Preserve the damaged source, inspect/reconcile data, or recover beyond the backtrack window |
| **Snapshot restore** | Creates a new cluster at snapshot time | Long-term retention, account/Region copy, or a known release boundary |

Backtrack is unavailable for Aurora PostgreSQL, must have been enabled when the cluster was
created, and is not a cross-Region or durable-backup strategy. PITR and snapshot restore have new
endpoints, so restoration also requires configuration, validation, and traffic switching.

**Fast database cloning** creates a new cluster that shares the source's storage using
**copy-on-write** — the clone is near-instant and initially consumes almost no extra storage
(only changed pages diverge). Ideal for spinning up staging/test copies of production data.

A deployment test with a clone should create isolated credentials/networking, disable outbound
side effects such as email and payment calls, mask sensitive data where required, apply the change,
and run representative read/write and rollback tests. Copy-on-write makes creation cheap, not the
whole test: long-running divergent writes add storage and I/O cost, and the clone is not refreshed
automatically from its source.

---

## 8. Aurora vs RDS — Decision Points

| Factor | **RDS (MySQL/PostgreSQL)** | **Aurora** |
|--------|----------------------------|------------|
| Storage | Per-instance EBS volume | Shared, distributed, 6 copies/3 AZs, auto-grow to 128 TiB |
| Read replicas | Up to 15, **async** (can lag) | Up to 15, **~ms lag** (shared storage) |
| Failover | ~60–120s, DNS flip to standby | ~30s, promote replica, shared storage |
| Reader load balancing | No (per-replica endpoint) | **Yes** (reader endpoint) |
| Cross-Region DR | Cross-Region read replica | **Global Database** (typically <1s lag, managed failover) |
| Serverless option | No | **Aurora Serverless v2** |
| Backtrack / fast clone | No | **Yes** |
| Cost | Often lower baseline; conventional storage/I/O pricing | Often higher baseline; compare compute, replicas, storage, and Standard vs I/O-Optimized I/O pricing |

> **Key insight**: Pick **Aurora** when you need higher performance, faster/automatic failover,
> reader endpoint load balancing, cross-Region DR with sub-second lag, serverless scaling, or
> very large storage. Pick **plain RDS** for simpler workloads, tighter budgets, or engines Aurora
> doesn't support (Oracle, SQL Server, MariaDB, Db2).

---

## 9. Cost Note

- Aurora has **no upfront storage provisioning** — you pay for what you use (storage + I/O +
  instance hours), and storage auto-grows.
- Aurora instances cost **more** than equivalent RDS instances, but you avoid over-provisioning
  storage and gain capabilities (faster failover, more replicas, reader endpoint).
- **Aurora I/O-Optimized** is a pricing mode that removes per-request I/O charges — predictable
  cost for **I/O-heavy** workloads.
- **Serverless v2** can be cheaper for spiky/idle workloads because capacity follows demand, but
  you still pay for the configured minimum (unless a supported scale-to-zero configuration pauses).

---

## 10. Key Exam Points

- ✅ Aurora stores **6 copies across 3 AZs**, self-healing, auto-grows to **128 TiB**.
- ✅ Up to **15 Aurora Replicas** with **~ms** lag; **reader endpoint** load-balances reads.
- ✅ Failover is **fast (~30s)** because storage is shared.
- ✅ **Serverless v2** → autoscaling capacity for unpredictable relational workloads; scale-to-zero
  is available on supported engine/platform versions, not a universal guarantee.
- ✅ **Global Database** → cross-Region DR, typically **<1s** replication, up to **10** secondaries.
  Planned switchover targets RPO 0; unplanned failover RPO depends on observed lag.
- ✅ A global writer endpoint follows the primary, but clients must still refresh DNS, reconnect,
  and retry safely. Write forwarding remains single-writer and adds cross-Region latency.
- ✅ **Backtrack** rewinds in place (Aurora MySQL); **fast clone** = copy-on-write test copies.
- ✅ Aurora still lives in a VPC/DB subnet group and uses SGs like RDS; connect through writer/reader endpoints.

---

## 11. Common Mistakes

- ❌ Choosing a cross-Region RDS read replica when the requirement is fast cross-Region DR — use
  **Aurora Global Database**.
- ❌ Treating **Backtrack** as a backup replacement — it's an in-place rewind, not durable recovery.
- ❌ Assuming Aurora supports Oracle/SQL Server/MariaDB/Db2 — it's **MySQL/PostgreSQL-compatible only**.
- ❌ Hard-coding instance endpoints instead of using the **reader/writer** cluster endpoints.
- ❌ Treating Aurora Serverless as outside-VPC/public by default. It still uses VPC subnet groups and SGs.
- ❌ Calling secondary-Region write forwarding active-active; writes still commit in the primary.
- ❌ Promising RPO 0 for an unplanned Global Database failure without checking replication lag.

---

## 12. Limits & Quick Facts

- Storage: auto-grows to **128 TiB**, **6 copies / 3 AZs**.
- Read replicas: up to **15**.
- Global Database: up to **10** secondary Regions, typically **<1s** lag.
- Backtrack window: up to **72 hours** (Aurora MySQL).

---

**Next**: [04_dynamodb.md — Amazon DynamoDB](04_dynamodb.md)
