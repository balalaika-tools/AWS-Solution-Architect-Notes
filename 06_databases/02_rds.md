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

It supports six engines:

| Engine | Notes |
|--------|-------|
| **MySQL** | Open source, most common. |
| **PostgreSQL** | Open source, feature-rich. |
| **MariaDB** | MySQL fork. |
| **Oracle** | Commercial; BYOL or license-included. |
| **SQL Server** | Microsoft; license-included or BYOL. |
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
| Deleted when DB is deleted? | **Yes** (unless final snapshot taken) | **No** — survive DB deletion |

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

**Maintenance windows**: a weekly window during which AWS applies patches and OS/engine upgrades.
Mandatory OS patching may cause a brief failover (less disruptive with Multi-AZ). You choose the
window to minimize impact.

---

## 9. Key Exam Points

- ✅ RDS = managed relational engine on an instance; you choose the **instance class** and pay for
  it always-on. It is **not** serverless.
- ✅ **Multi-AZ** = synchronous standby, **automatic failover**, **same endpoint** → availability.
- ✅ **Read Replicas** = asynchronous, up to **15**, **own endpoint**, **cross-Region** capable →
  read scaling. Manual promotion only.
- ✅ **Automated backups** → PITR up to 35 days; **manual snapshots** → kept indefinitely,
  copyable across Region/account.
- ✅ Encryption at rest must be set **at creation**; to encrypt later, snapshot → copy encrypted →
  restore.
- ✅ **RDS Proxy** solves Lambda/connection-exhaustion problems via pooling.
- ✅ RDS is VPC-placed through a DB subnet group; protect it with SG rules from app/Lambda/task SGs and connect by endpoint DNS.

---

## 10. Common Mistakes

- ❌ Saying Multi-AZ improves read performance — it does not (standby is idle).
- ❌ Saying a Read Replica provides automatic failover — promotion is manual.
- ❌ Trying to enable encryption on an existing unencrypted RDS in place — you must restore from
  an encrypted snapshot copy.
- ❌ Forgetting that restoring a snapshot creates a **new instance with a new endpoint**.
- ❌ Choosing larger instances to fix "too many connections" from Lambda instead of **RDS Proxy**.

---

## 11. Limits & Quick Facts

- Up to **15** read replicas per RDS source database.
- Automated backup retention: **0–35 days**; PITR within that window.
- IAM auth tokens valid for **15 minutes**.
- DB subnet group requires subnets in **≥2 AZs**.

---

**Next**: [03_aurora.md — Amazon Aurora](03_aurora.md)
