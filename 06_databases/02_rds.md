# Amazon RDS — Managed Relational Database

> **Who this is for**: Engineers who understand relational databases and the
> read-replica vs standby distinction from [01_database_fundamentals.md](01_database_fundamentals.md)
> and now need the AWS-managed version. The **Multi-AZ vs Read Replica** comparison here is one
> of the most frequently tested topics on the entire SAA-C03 exam.

---

## 1. What RDS Is

**Amazon RDS (Relational Database Service)** is a *managed* relational database. AWS runs the
database engine on EC2 instances for you and handles the operational chores: provisioning,
patching, backups, failover, and replication. You still manage the schema, queries, and the
instance size.

It supports these engines:

| Engine | Notes |
|--------|-------|
| **MySQL** | Open source, most common. |
| **PostgreSQL** | Open source, feature-rich. |
| **MariaDB** | MySQL fork. |
| **Oracle** | Commercial; BYOL or license-included. |
| **SQL Server** | Microsoft; license-included. |
| **Db2** | IBM Db2; BYOL-oriented enterprise workloads. |
| **Aurora** | AWS's cloud-native MySQL/PostgreSQL-compatible engine — see [03_aurora.md](03_aurora.md). |

> **Key insight**: RDS is "managed but not serverless." You still pick an **instance class**
> (vCPU/RAM) and pay for it whether or not it's busy. The database engine is the same software
> you'd run yourself — AWS just operates it. This is what distinguishes RDS from DynamoDB
> (fully serverless) and Aurora Serverless.

⚠️ Because it's running a standard engine on an instance, RDS scales **vertically** (resize the
instance) and scales **reads** horizontally (read replicas) — but it does **not** auto-shard writes.

---

## 2. Multi-AZ — High Availability (NOT Scaling)

**Multi-AZ** maintains a **synchronous standby** replica in a *different* Availability Zone.

```
        ┌──────────────── Region ────────────────┐
        │   AZ-a                    AZ-b         │
        │  ┌─────────┐  sync       ┌─────────┐   │
        │  │ PRIMARY │────────────►│ STANDBY │   │
        │  │ (active)│  replication│ (idle)  │   │
        │  └────┬────┘             └─────────┘   │
        └───────┼────────────────────────────────┘
                │  one DNS endpoint (CNAME)
                ▼
            application  ── on failure, DNS flips to standby automatically
```

Key behaviors:
- Replication is **synchronous** — the standby is always up to date.
- The standby serves **no read or write traffic**. It exists purely for availability.
- On primary failure (AZ outage, instance crash, patching), RDS **automatically fails over** by
  flipping the DNS **endpoint** to the standby — typically in 60–120 seconds.
- The application uses the **same endpoint** before and after failover — no connection-string
  change needed.

> **Rule**: Multi-AZ = **availability**, not performance. It does *not* give you extra read
> capacity. If a question says "improve availability / survive an AZ failure / automatic
> failover," the answer is Multi-AZ. If it says "scale read traffic," it's a Read Replica.

💡 **Multi-AZ DB cluster** (newer option) uses two *readable* standbys across three AZs and can
serve reads — but the classic, default Multi-AZ *instance* deployment keeps the standby idle.

---

## 3. Read Replicas — Scaling Reads

A **Read Replica** is an **asynchronous** copy used to offload read-heavy traffic from the
primary. It has its **own endpoint**.

```
        ┌─────────┐  async    ┌──────────────┐
        │ PRIMARY │──────────►│ READ REPLICA │  (own endpoint)
        │ (writes)│  (may lag)│ (SELECTs)    │
        └─────────┘           └──────────────┘
   writes go here          point reporting / read-heavy apps here
```

Key behaviors:
- Replication is **asynchronous** → replicas can **lag** behind (eventual consistency).
- Up to **15 read replicas** per source (RDS); each has its **own endpoint** the app must target.
- Replicas can be **cross-AZ or cross-Region** (cross-Region helps with global read latency and
  DR).
- A replica can be **manually promoted** to a standalone read/write database (useful for DR or
  migrations).
- Read replicas can themselves be Multi-AZ for their own availability.

⚠️ Read replicas are **not** automatic failover targets. Promotion is manual and breaks
replication. For automatic failover, that's Multi-AZ's job.

---

## 4. Multi-AZ vs Read Replica — The Comparison Table

| Feature | **Multi-AZ** | **Read Replica** |
|---------|--------------|------------------|
| Primary purpose | **High availability** | **Scale read traffic** |
| Replication | **Synchronous** | **Asynchronous** (can lag) |
| Standby serves traffic? | No (idle)* | Yes (reads only) |
| Endpoint | **Same** endpoint (DNS failover) | **Separate** endpoint per replica |
| Failover | **Automatic** | Manual promotion |
| Cross-Region | No (same Region) | **Yes** |
| Count | One standby* | Up to **15** |
| Consistency | Always current | Eventually consistent |

*Classic Multi-AZ instance deployment. The Multi-AZ *DB cluster* variant has readable standbys.

> **Key insight**: They are **complementary, not alternatives**. A production database often uses
> Multi-AZ *for availability* **and** read replicas *for read scaling* at the same time.

For a full worked scenario, see
[RDS Multi-AZ vs Read Replica](../18_practical_examples/12_rds_multiaz_vs_read_replica.md).

---

## 5. Backups — Automated vs Manual Snapshots

RDS provides two backup mechanisms:

| | **Automated backups** | **Manual snapshots** |
|--|------------------------|----------------------|
| Created by | RDS, daily + transaction logs | You, on demand |
| Restore granularity | **Point-in-time (PITR)** to any second | The moment the snapshot was taken |
| Retention | 0–35 days (0 = disabled) | Until you delete them |
| Deleted when DB is deleted? | By default; you can retain automated backups for their retention window | **No** — survive DB deletion |

```
PITR: daily snapshot + 5-min transaction logs = restore to any second in window

  day 1 ──── day 2 ──── day 3 ──── [now]
  [snap]     [snap]     [snap]
   └─── replay transaction logs to your chosen second ───►
```

- **Automated backups** enable **Point-In-Time Recovery (PITR)** — restore to any second within
  the retention window (up to 35 days). A daily snapshot plus continuous transaction logs makes
  this possible.
- **Manual snapshots** are user-initiated, retained indefinitely, and can be **copied or shared
  across Regions and accounts** (key for DR and migrations).
- ⚠️ Restoring any backup/snapshot creates a **new** RDS instance with a **new endpoint** — it
  does not overwrite the existing database.

---

## 6. Storage, Encryption & Connectivity

**Storage autoscaling**: RDS can automatically grow storage when free space runs low (set a max
limit), avoiding "disk full" outages — but it never shrinks automatically.

**Encryption at rest** (KMS):
- Uses **AWS KMS**; encrypts the volume, automated backups, snapshots, and read replicas.
- ⚠️ Must be enabled **at creation time**. To encrypt an existing unencrypted DB: snapshot it,
  copy the snapshot with encryption enabled, then restore from the encrypted snapshot.
- A read replica must share the same encryption status as its source.

**Encryption in transit**: SSL/TLS connections to the database endpoint.

**IAM database authentication**: instead of a DB password, authenticate using an IAM-generated
auth token (valid 15 minutes). Good for short-lived credentials and centralized access control;
best for low-to-moderate connection rates.

```
            ┌──────────────────────── VPC ────────────────────────┐
            │   private subnets (DB subnet group across ≥2 AZs)   │
   app ────►│  ┌──────────────┐         ┌──────────────┐          │
 (SG rule)  │  │ RDS PRIMARY  │  sync   │ RDS STANDBY  │          │
            │  └──────────────┘────────►└──────────────┘          │
            └─────────────────────────────────────────────────────┘
   RDS lives in private subnets; access is gated by a Security Group.
```

💡 RDS instances live in a **DB subnet group** (subnets in ≥2 AZs) and are protected by a
**security group**. Keep them in **private subnets**; never expose the DB to the internet.

Network placement details:

- RDS assigns managed network interfaces/IPs from the **DB subnet group**. These IPs can change
  during failover, maintenance, or replacement.
- Applications should connect to the **RDS endpoint DNS name**, never a hard-coded private IP.
- The DB security group should allow the engine port only from the application tier's SG: app SG,
  Lambda SG, or ECS task SG. Example: PostgreSQL `5432` from `sg-app`; MySQL `3306` from `sg-app`.
- RDS does not need an inbound rule from the internet for a normal architecture. Use VPN/DX,
  bastion/SSM, or private app tiers for administration.

For the broader ENI/security-group map, see
[ENIs, Security Groups & Service Networking](../03_networking/07_enis_security_groups_and_service_networking.md).

---

## 7. RDS Proxy — Connection Pooling

**RDS Proxy** is a fully managed, highly available database proxy that **pools and shares**
database connections.

```
   many Lambda / app instances        RDS Proxy            RDS / Aurora
   ┌───┐ ┌───┐ ┌───┐ ┌───┐         ┌──────────┐         ┌──────────┐
   │   │ │   │ │   │ │   │ ──────► │  pooled  │ ──────► │ database │
   └───┘ └───┘ └───┘ └───┘ 1000s   │  conns   │  few    │          │
                                   └──────────┘         └──────────┘
   thousands of short-lived clients   reuses a small set of DB connections
```

- Solves **connection exhaustion** — especially with **Lambda**, which can spawn thousands of
  concurrent functions each opening a DB connection.
- Reduces failover time by up to ~66% (it holds connections and redirects them).
- Integrates with **Secrets Manager** for credentials and supports **IAM authentication**.
- Runs inside your **VPC**, not internet-accessible.

⚠️ Exam clue: "serverless app / Lambda exhausting database connections" → **RDS Proxy**.

---

## 8. RDS Custom & Maintenance

**RDS Custom** is for legacy/commercial apps (Oracle, SQL Server) that need **OS and database
access** to customize the underlying environment (install agents, custom patches) while still
getting much of RDS's automation. It's a middle ground between full RDS and self-managing on EC2.

RDS Custom currently supports **Oracle and SQL Server only**. You own more patching, licensing,
host changes, and troubleshooting than with standard RDS. RDS monitors a **support perimeter**;
a change that breaks required automation can put the instance into
`unsupported-configuration` until you repair it. Use RDS Custom for a proven host/database
dependency, not merely because a team is accustomed to server access.

**Maintenance windows**: a weekly window during which AWS applies patches and OS/engine upgrades.
Mandatory OS patching may cause a brief failover (less disruptive with Multi-AZ). You choose the
window to minimize impact.

---

## 9. Professional RDS Design Scenarios

### 9.1 Multi-AZ DB instance or Multi-AZ DB cluster?

Both survive an instance or AZ failure, but they have different cost, engine, read, and change
management characteristics.

| | **Multi-AZ DB instance deployment** | **Multi-AZ DB cluster deployment** |
|--|-------------------------------------|------------------------------------|
| Topology | One writer + one non-readable standby in another AZ | One writer + two readable replicas in three AZs |
| Replication | Synchronous storage/database replication | Engine-native **semisynchronous** replication; commit needs acknowledgment from at least one reader |
| Read scaling | None from the standby; add separate read replicas | Reader endpoint can use both readable replicas |
| Failover | Commonly 60–120 seconds; DNS still matters | Typically under 35 seconds, then clients reconnect to the writer endpoint |
| Engines | Broad RDS engine coverage | RDS for MySQL and PostgreSQL on supported versions/classes |
| Cost floor | Two DB instances | Three DB instances; higher baseline, but readers provide usable capacity |
| Change limitations | Supports RDS Blue/Green for eligible engines | RDS Blue/Green Deployments do **not** support Multi-AZ DB clusters |

Choose the **instance deployment** for normal HA when separate read capacity is unnecessary or
the engine is unsupported by clusters. Choose the **cluster deployment** when the application
can use two readers and faster failover justifies three-instance cost. Do not count cluster
readers as zero-lag strong-read targets: semisynchronous acknowledgment does not mean every
reader has already applied the transaction.

In both cases, clients use DNS endpoints and must retry connections/transactions after
failover. Test with the real driver, connection pool, JVM DNS TTL, and transaction retry logic;
the service failover time is not the complete application RTO.

### 9.2 Cross-Region replica or replicated automated backups?

Suppose a PostgreSQL database in `eu-west-1` needs disaster recovery in `eu-central-1`:

| Requirement | **Cross-Region read replica** | **Cross-Region automated backup replication** |
|-------------|-------------------------------|-----------------------------------------------|
| Recovery posture | Warm, running database; can serve regional reads | Cold snapshots + transaction logs in the destination Region |
| RPO | Asynchronous replica lag | Latest copied snapshot/log; monitor destination restorable time |
| RTO | Promote replica, redirect application, reconfigure HA/backups | Restore a new DB instance, then redirect application |
| Cost | Instance, storage, I/O, and transfer continuously | Backup storage/copy/transfer; no idle DB compute |
| Best fit | Low RTO or useful cross-Region reads | Lower-cost DR with a longer restore window |

Automated backup replication copies **snapshots and transaction logs**, so it supports PITR in
the destination; it is not a readable replica. A replica promotion and a backup restore both
produce a different write endpoint from the original Region. Pre-provision networking,
security groups, parameter/option groups, secrets, KMS access, DNS failover, and application
configuration, then run recovery drills. "The data is copied" is only the RPO half of DR.

### 9.3 Low-downtime engine change with Blue/Green Deployments

Use **RDS Blue/Green Deployments** for eligible RDS for MySQL, MariaDB, or PostgreSQL changes
such as an engine upgrade or parameter change:

1. RDS creates a synchronized **green** staging environment in the same account and Region.
2. Apply the upgrade and run production-like validation against green without sending
   production writes there.
3. Check replication/CDC lag, long-running transactions, external jobs, extensions, and
   switchover guardrails. Freeze unsupported DDL before the change.
4. Start the managed switchover. RDS stops new writes, catches green up, renames resources and
   endpoints, and makes green production. Existing connections are interrupted and must retry.
5. Keep old blue briefly for investigation, but do not call it instant rollback. Once green
   accepts writes, returning to blue requires reconciling those writes or accepting data loss.

Blue/Green is not supported for RDS for Oracle, SQL Server, or Db2, and not for Multi-AZ DB
clusters or environments with cross-Region read replicas. Feature/version limitations differ
by engine—especially PostgreSQL physical versus logical replication—so validate the exact
topology before promising a zero-downtime upgrade. You also pay for both environments while the
deployment exists.

### 9.4 RDS Custom or standard RDS?

A vendor's Oracle application requires an OS agent, database filesystem changes, and a
certified patch sequence. Standard RDS deliberately blocks that access; **RDS Custom** is the
managed landing zone. Document every customization, preserve Systems Manager connectivity and
required roles, monitor support-perimeter events, and test backup/restore after changes.

If the requirement is only SQL access, parameter groups, extensions supported by RDS, or a
monitoring integration already provided by AWS, choose standard RDS. RDS Custom trades away
some automation and increases operational/exit cost; self-managed EC2 remains the fallback
when even the Custom support perimeter is too restrictive.

### 9.5 What RDS Proxy does during failover

RDS Proxy pools connections and tracks the current database target without waiting for an
application's cached DNS entry. For Multi-AZ DB instances it can reduce failover time by up to
66%, automatically route to the new target, and preserve many client connections. It can also
follow an eligible Blue/Green switchover when the blue database was registered before the
deployment was created.

It does **not** make every SQL operation transparent:

- An in-flight transaction can fail or have an uncertain outcome. Retry only with idempotency
  or transaction-outcome checks.
- Session state, temporary tables, large statements, and some engine operations can **pin** a
  client to one database connection, reducing multiplexing. Monitor
  `DatabaseConnectionsCurrentlySessionPinned`.
- Pool limits can still exhaust the database. Size `MaxConnectionsPercent`, reserve headroom
  for administration/monitoring, and apply client backoff.

Use RDS Proxy for connection storms and faster target discovery; still implement database
error handling and test failover.

### 9.6 Diagnose database load with CloudWatch Database Insights

The Performance Insights console reaches end of life on **July 31, 2026** and redirects to
**CloudWatch Database Insights**. The Performance Insights API and its configuration parameters
continue, while Database Insights becomes the operating experience:

- **Standard mode** provides core load monitoring and flexible retention comparable to
  Performance Insights.
- **Advanced mode** adds fleet-level views, 15 months of telemetry, on-demand analysis, lock
  diagnostics, and execution-plan capture at additional cost.

Start diagnosis with **DB load (average active sessions)** and wait categories, not CPU alone:

1. Compare DB load with vCPU and find the time the regression began.
2. Identify top waits, SQL statements, users, hosts, and databases.
3. Correlate them with CloudWatch CPU, free memory, connections, storage latency/IOPS,
   throughput, replica lag, and network metrics.
4. Inspect execution plans and locks, then change one cause—index/query, connection pool,
   instance/storage, or blocking transaction—and verify the load profile.

Scaling the instance can hide an inefficient query; adding an index can worsen a write-heavy
workload. Database Insights supplies evidence for the choice rather than replacing database
analysis.

### 9.7 Move an encrypted snapshot across accounts

An RDS snapshot encrypted with the default `aws/rds` KMS key **cannot be shared**. Use this
handoff for a migration from source account A to target account B:

1. If necessary, copy the source snapshot in A and encrypt the copy with a **customer-managed
   KMS key** in the snapshot's Region.
2. Add B to the KMS key policy (and IAM permissions) for the cryptographic operations RDS needs.
3. Share the manual encrypted snapshot with B. Automated backups are not shared directly;
   first copy the required recovery point to a manual snapshot.
4. In B, copy the shared snapshot and re-encrypt it with a KMS key owned by B.
5. Restore B's copy to a new DB instance, recreate non-snapshot dependencies, validate, and
   switch the application endpoint.
6. Remove the share and source-key access only after B's copy and restore are verified.

This is a **copy migration**, not CDC. Its data-loss window begins at snapshot time, and restore
time contributes to the cutover unless the target is restored and caught up separately.

---

## 10. Key Exam Points

- ✅ RDS = managed relational engine on an instance; you choose the **instance class** and pay for
  it always-on. It is **not** serverless.
- ✅ **Multi-AZ** = synchronous standby, **automatic failover**, **same endpoint** → availability.
  Classic Multi-AZ DB instance standby is not readable; the newer Multi-AZ DB cluster has two
  readable standbys.
- ✅ **Read Replicas** = asynchronous, up to **15**, **own endpoint**, **cross-Region** capable →
  read scaling. Manual promotion only.
- ✅ **Automated backups** → PITR up to 35 days; **manual snapshots** → kept indefinitely,
  copyable across Region/account.
- ✅ Encryption at rest must be set **at creation**; to encrypt later, snapshot → copy encrypted →
  restore.
- ✅ **RDS Proxy** solves Lambda/connection-exhaustion problems via pooling.
- ✅ Multi-AZ DB **cluster** = writer + two readable replicas in three AZs, typically faster
  failover and higher baseline cost; it is not supported by RDS Blue/Green Deployments.
- ✅ Cross-Region replica = warm/readable and manually promoted. Replicated automated backups =
  cold snapshots/logs with destination-Region PITR and a longer RTO.
- ✅ **Blue/Green** creates synchronized staging for eligible MySQL/MariaDB/PostgreSQL changes;
  switchover is brief but connections retry, and post-write rollback is not automatic.
- ✅ Use **CloudWatch Database Insights** for current performance diagnosis. Performance
  Insights API compatibility remains, but its console reaches end of life July 31, 2026.
- ✅ Cross-account encrypted snapshots require a customer-managed source KMS key; the target
  should copy and re-encrypt with its own key before restore.
- ✅ RDS is VPC-placed through a DB subnet group; protect it with SG rules from app/Lambda/task SGs and connect by endpoint DNS.

---

## 11. Common Mistakes

- ❌ Saying Multi-AZ improves read performance — it does not (standby is idle).
- ❌ Saying a Read Replica provides automatic failover — promotion is manual.
- ❌ Trying to enable encryption on an existing unencrypted RDS in place — you must restore from
  an encrypted snapshot copy.
- ❌ Forgetting that restoring a snapshot creates a **new instance with a new endpoint**.
- ❌ Choosing larger instances to fix "too many connections" from Lambda instead of **RDS Proxy**.
- ❌ Assuming RDS Proxy preserves an in-flight transaction through failover, or that Blue/Green
  provides automatic rollback after new production writes.
- ❌ Sharing an encrypted snapshot that uses `aws/rds`—copy it under a customer-managed key.
- ❌ Calling cross-Region backup replication a warm standby; recovery still requires restore.

---

## 12. Limits & Quick Facts

- Up to **15** read replicas per RDS source database.
- Automated backup retention: **0–35 days**; PITR within that window.
- IAM auth tokens valid for **15 minutes**.
- DB subnet group requires subnets in **≥2 AZs**.

---

**Next**: [03_aurora.md — Amazon Aurora](03_aurora.md)
