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

- Scales **in fine-grained increments**, in **fractions of a second**, with no connection drops
  (a big improvement over Serverless v1, which scaled in coarse steps and paused).
- You set a **min and max ACU** range; you pay for capacity consumed.
- Ideal for **variable, unpredictable, or spiky** workloads, dev/test, and multi-tenant apps
  where provisioning a fixed instance is wasteful.

💡 Exam clue: "relational database with **unpredictable / intermittent** load, don't want to
manage capacity" → **Aurora Serverless v2**.

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
                            up to 5 secondary Regions
```

- One **primary Region** (read/write) replicates to up to **5 secondary Regions** (read-only).
- Typical cross-Region replication lag **< 1 second**.
- **RTO < 1 minute**: a secondary Region can be promoted to primary for DR.
- Dedicated replication infrastructure — does not consume the primary's compute.

⚠️ Distinguish from a cross-Region **read replica** in plain RDS: Aurora Global Database is purpose-
built for fast DR and global low-latency reads, with sub-second lag and managed promotion.

---

## 7. Backtrack & Cloning

**Backtrack** (Aurora MySQL) rewinds the cluster to a previous point in time **in place** —
**without restoring from a backup** or creating a new cluster. Great for quickly undoing a bad
deployment or accidental bulk delete.

- ⚠️ Backtrack ≠ backup. It rewinds the *existing* cluster within a configured window (up to 72
  hours); it does not replace snapshots/PITR for long-term recovery.

**Fast database cloning** creates a new cluster that shares the source's storage using
**copy-on-write** — the clone is near-instant and initially consumes almost no extra storage
(only changed pages diverge). Ideal for spinning up staging/test copies of production data.

---

## 8. Aurora vs RDS — Decision Points

| Factor | **RDS (MySQL/PostgreSQL)** | **Aurora** |
|--------|----------------------------|------------|
| Storage | Per-instance EBS volume | Shared, distributed, 6 copies/3 AZs, auto-grow to 128 TiB |
| Read replicas | Up to 15, **async** (can lag) | Up to 15, **~ms lag** (shared storage) |
| Failover | ~60–120s, DNS flip to standby | ~30s, promote replica, shared storage |
| Reader load balancing | No (per-replica endpoint) | **Yes** (reader endpoint) |
| Cross-Region DR | Cross-Region read replica | **Global Database** (<1s lag, fast promote) |
| Serverless option | No | **Aurora Serverless v2** |
| Backtrack / fast clone | No | **Yes** |
| Cost | Lower baseline | Higher per-instance; ~20% more, but more capability |

> **Key insight**: Pick **Aurora** when you need higher performance, faster/automatic failover,
> reader endpoint load balancing, cross-Region DR with sub-second lag, serverless scaling, or
> very large storage. Pick **plain RDS** for simpler workloads, tighter budgets, or engines Aurora
> doesn't support (Oracle, SQL Server, MariaDB).

---

## 9. Cost Note

- Aurora has **no upfront storage provisioning** — you pay for what you use (storage + I/O +
  instance hours), and storage auto-grows.
- Aurora instances cost **more** than equivalent RDS instances, but you avoid over-provisioning
  storage and gain capabilities (faster failover, more replicas, reader endpoint).
- **Aurora I/O-Optimized** is a pricing mode that removes per-request I/O charges — predictable
  cost for **I/O-heavy** workloads.
- **Serverless v2** can be cheaper for spiky/idle workloads since you stop paying for unused
  fixed capacity.

---

## 10. Key Exam Points

- ✅ Aurora stores **6 copies across 3 AZs**, self-healing, auto-grows to **128 TiB**.
- ✅ Up to **15 Aurora Replicas** with **~ms** lag; **reader endpoint** load-balances reads.
- ✅ Failover is **fast (~30s)** because storage is shared.
- ✅ **Serverless v2** → autoscaling capacity for unpredictable relational workloads.
- ✅ **Global Database** → cross-Region DR, **<1s** replication, **<1min** RTO, up to 5 secondaries.
- ✅ **Backtrack** rewinds in place (Aurora MySQL); **fast clone** = copy-on-write test copies.
- ✅ Aurora still lives in a VPC/DB subnet group and uses SGs like RDS; connect through writer/reader endpoints.

---

## 11. Common Mistakes

- ❌ Choosing a cross-Region RDS read replica when the requirement is fast cross-Region DR — use
  **Aurora Global Database**.
- ❌ Treating **Backtrack** as a backup replacement — it's an in-place rewind, not durable recovery.
- ❌ Assuming Aurora supports Oracle/SQL Server — it's **MySQL/PostgreSQL-compatible only**.
- ❌ Hard-coding instance endpoints instead of using the **reader/writer** cluster endpoints.
- ❌ Treating Aurora Serverless as outside-VPC/public by default. It still uses VPC subnet groups and SGs.

---

## 12. Limits & Quick Facts

- Storage: auto-grows to **128 TiB**, **6 copies / 3 AZs**.
- Read replicas: up to **15**.
- Global Database: up to **5** secondary Regions, **<1s** lag.
- Backtrack window: up to **72 hours** (Aurora MySQL).

---

**Next**: [04_dynamodb.md — Amazon DynamoDB](04_dynamodb.md)
