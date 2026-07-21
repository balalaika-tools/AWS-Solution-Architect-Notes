# Migration Services: Moving Workloads into AWS

> **Who this is for**: An engineer planning to move existing servers, databases, and files
> from a data center into AWS, who needs to know *which* AWS tool fits *which* job. Assumes
> you've read [01_hybrid_networking.md](01_hybrid_networking.md) — most online migration tools
> need that hybrid link first.

---

## 1. First, the Strategy: The 7 R's

Before tooling, AWS frames every workload by **how** you migrate it. The current AWS Migration
Acceleration Program uses **seven** strategies. They are choices, not maturity levels: the right
answer can be to retire or retain a workload rather than force it into AWS.

| R | Meaning | Example |
|---|---------|---------|
| **Rehost** | "Lift and shift" — move as-is, no code change | Copy a VM to EC2 unchanged |
| **Replatform** | "Lift, tinker, shift" — small optimizations | Move a self-managed MySQL to RDS |
| **Repurchase** | Drop the old product, buy a SaaS replacement | Self-hosted CRM → Salesforce |
| **Relocate** | Move infrastructure at scale without redesigning workloads | Move VMware workloads to VMware Cloud on AWS |
| **Refactor / re-architect** | Change the architecture for cloud-native capabilities | Monolith → serverless services |
| **Retire** | Turn it off — nobody uses it | Decommission a legacy report server |
| **Retain** | Leave it on-prem for now | Mainframe that can't move yet |

> **Key insight**: **Rehost** maps to Application Migration Service and **replatform** often
> maps to DMS plus a managed database. But a portfolio normally uses several strategies; decide
> per dependency group and business outcome, not per server in isolation.

---

## 2. Discovery and Assessment: AWS Transform, with Migration Hub for Existing Projects

For a **new project**, use the AWS Transform assessment/discovery workflow and the AWS Transform
Discovery Tool. **AWS Migration Hub**, Migration Hub Strategy Recommendations, and AWS Application
Discovery Service stopped accepting new customers on November 7, 2025. They remain important for
existing in-flight migrations and older exam material: Migration Hub is the dashboard that tracks
multiple migration tools, while Application Discovery Service agents/collectors build inventory.

Neither Transform nor Migration Hub moves a server or database by itself. The services below do the
replication and transfer; the assessment layer organizes evidence and decisions.

---

## 3. Application Migration Service (MGN) — Rehost Servers

**AWS Application Migration Service (MGN)** is the primary **lift-and-shift** tool. It
**continuously replicates** your on-prem (or other-cloud) servers — entire OS, apps, disks —
into AWS, then lets you **cut over** to running EC2 instances with minimal downtime.

```
 On-prem server ──block-level replication──▶ Staging area in AWS ──cutover──▶ EC2
   (agent)            (continuous)              (low-cost)              (live)
```

- Works at the **block level**, so the migrated instance is byte-identical.
- Continuous replication means a **short cutover window** (minutes), not a full re-copy.
- ⚠️ **MGN is the successor to Server Migration Service (SMS)**. SMS is deprecated — if you
  see "SMS" as an answer for new migrations, it's the wrong (old) one. MGN is the current
  answer for rehosting servers/VMs.

✅ Use MGN whenever the requirement is *"migrate existing VMs/physical servers to EC2 with
minimal downtime / minimal changes."*

---

## 4. Database Migration Service (DMS) + Schema Conversion

**AWS Database Migration Service (DMS)** migrates **databases** to AWS with the **source
staying online** during the migration. It supports one-time loads and **ongoing replication
(CDC — change data capture)** so the target stays in sync until you cut over.

Two flavors of database migration:

| Migration type | Engines | Tool needed |
|----------------|---------|-------------|
| **Homogeneous** (same engine) | Oracle → Oracle, MySQL → MySQL/RDS | **DMS** alone |
| **Heterogeneous** (different engine) | Oracle → Aurora PostgreSQL, SQL Server → MySQL | **DMS Schema Conversion** where supported, or **SCT** as the desktop fallback; then DMS for data |

**DMS Schema Conversion** converts the **schema and code** (tables, stored procedures, views) from
the source dialect to the target where supported. Use the desktop **AWS Schema Conversion Tool
(SCT)** when the source/target or feature needs that fallback. DMS then moves the **data**.

```
 Heterogeneous migration:
   Source DB ──schema/code──▶ [ DMS Schema Conversion / SCT ] ──▶ Target DB
   Source DB ──── data + ongoing CDC ──▶ [ DMS ] ──▶ Target DB
```

> **Rule**: **Same engine → DMS can move the data without cross-engine conversion. Different
> engine → schema conversion + DMS data migration.** Recognize SCT in older questions, but prefer
> DMS Schema Conversion for a supported current design.

💡 DMS Standard uses a managed **replication instance**; **DMS Serverless** automatically provisions
replication capacity. Either model needs network reach to source and target—hence the hybrid link
from file 01—and must be sized/monitored for full load plus CDC.

---

## 5. DataSync — Online Bulk File Transfer

**AWS DataSync** is an agent-based service for **moving large amounts of file/object data
online** between on-prem storage (NFS, SMB, HDFS, object stores) and AWS storage.

Targets: **S3, EFS, FSx** (and AWS-to-AWS transfers too).

- **Fast** (purpose-built protocol, parallelized, up to ~10× faster than open-source copy
  tools), with **built-in encryption, integrity checks, and scheduling**.
- Handles incremental syncs and one-time migrations.

✅ Use DataSync for *"migrate/replicate file shares or NFS data to S3/EFS/FSx over the
network."*

❌ DataSync is **online** — if the data is huge and bandwidth is the bottleneck, current
new-customer designs should evaluate **AWS Data Transfer Terminal** or partner physical-transfer
options. Older exam material may still point to **Snowball Edge**.

---

## 6. Physical Transfer — Data Transfer Terminal and Snow Legacy

When you have **petabytes** of data and moving it online would take weeks, you need a physical
transfer pattern.

- **AWS Data Transfer Terminal**: current new-customer option where you bring your own storage
  devices to a secure AWS physical location and upload over a high-throughput AWS connection.
- **Snowball Edge / Snowmobile**: legacy/existing-customer recognition. Snowcone is discontinued,
  and Snowball Edge is no longer open to new customers as of November 7, 2025.

> Covered in depth in
> [../05_storage/06_storage_gateway_and_transfer.md](../05_storage/06_storage_gateway_and_transfer.md).

💡 Quick rule of thumb: if transferring the data online would take **more than a week**, older
exam material says Snow. For current designs, translate that into "use a supported physical-transfer
option," then check Data Transfer Terminal/partners before assuming a Snow device.

---

## 7. AWS Transfer Family — Managed SFTP/FTPS/FTP

**AWS Transfer Family** provides **fully managed SFTP, FTPS, and FTP** endpoints that land
files directly in **S3 or EFS**. It's not a one-time migration tool — it's for **ongoing**
file exchange with external partners who already speak these protocols, without you running
file-transfer servers.

✅ Use when partners send/receive files **via SFTP** and you want them to land in S3 with no
server to manage.

---

## 8. Which Migration Tool? — Decision Table

| Requirement | Service |
|-------------|---------|
| Lift-and-shift **servers/VMs** to EC2, minimal downtime | **Application Migration Service (MGN)** |
| Migrate a database, **same engine**, source stays online | **DMS** |
| Migrate a database to a **different engine** | **DMS Schema Conversion** where supported, or **SCT** fallback, then **DMS** data migration |
| Assess/discover a new migration portfolio | **AWS Transform** assessment + Discovery Tool |
| Track an existing pre-November-2025 migration project | **Migration Hub** |
| Move **file/NFS/object data online** to S3/EFS/FSx | **DataSync** |
| Move **petabytes offline** (bandwidth-limited) | Current: **Data Transfer Terminal / partners**; legacy exam: **Snowball Edge** |
| **Ongoing SFTP/FTPS** file exchange into S3/EFS | **Transfer Family** |
| Recognize legacy/existing inventory collectors | **Application Discovery Service** agents/collector |

---

## 9. Portfolio Assessment and Migration Waves

A tool choice is premature until the organization knows what it owns and what depends on what.
Use a repeatable assessment pipeline:

1. **Build the inventory.** Combine CMDB and owner interviews with the AWS Transform Discovery
   Tool. For an existing project, Application Discovery Service agents/agentless collection remain
   relevant. Capture servers, processes, network connections, utilization, licenses, data stores,
   owners, criticality and regulatory constraints.
2. **Build the business case.** Use AWS Transform assessments; existing Migration Evaluator Quick
   Insights exports can supply inputs. Add one-time migration effort, license changes, network
   transfer, parallel-running cost, support skills and retirement savings; an EC2-only estimate is
   not a TCO.
3. **Select a strategy and target.** AWS Transform assessment recommendations—or Migration Hub
   Strategy Recommendations for an existing customer—can inform a 7-R decision, but architects
   validate it against application roadmap, data gravity, recovery, operational readiness and
   unsupported features.
4. **Group dependencies into waves.** Move a small, low-risk dependency group first. Keep tightly
   coupled application, identity, DNS and database components in the same wave or explicitly
   design the temporary hybrid calls. Sequence shared services before consumers.
5. **Define entry and exit criteria.** Each wave needs owners, test evidence, performance and
   security baselines, acceptable replication lag, downtime budget, rollback deadline, cost
   guardrails and a post-cutover support period.

Assessment tooling gives portfolio visibility; it does not replace governance. A migration factory
also needs an account vending/landing-zone process, reusable network and identity patterns,
security approvals, wave dashboards, exception management and accountable application owners.

### Wave selection example

A three-tier application has web and app VMs, an Oracle database, Active Directory integration,
an SFTP partner and a nightly file feed. Migrating only the VMs first creates high-latency calls
back to the database and identity systems. A safer plan establishes hybrid DNS/identity and the
SFTP target first, rehearses DMS Schema Conversion/SCT plus data and file synchronization, then cuts over the
dependency group in one wave. Retire the source only after reconciliation and the rollback window.

---

## 10. End-to-End Cutover Runbooks

### Rehost a server estate with MGN

1. **Discover and qualify:** verify supported OS/storage, licenses, boot mode, IP dependencies,
   agents, security software, RTO/RPO, target instance family and service quotas.
2. **Prepare the landing zone:** create accounts, subnets, routes, DNS, IAM instance profiles,
   KMS keys, security groups, logging, patching and backup. Confirm the staging area and source
   egress can reach required MGN endpoints.
3. **Replicate and observe:** install/authorize the replication agent, monitor lag and staging
   health, and investigate throttling before the change window.
4. **Launch test instances:** use an isolated test subnet, validate boot, identity, middleware,
   secrets, monitoring, backups, performance and upstream/downstream calls. A successful EC2 boot
   is not application acceptance.
5. **Rehearse and approve:** record elapsed times, automate launch settings and post-launch
   actions, freeze risky source changes, and establish explicit go/no-go and rollback owners.
6. **Cut over:** stop or drain writes as required, wait for replication to converge, launch
   cutover instances, update load balancers/DNS and validate synthetic plus business transactions.
7. **Rollback or stabilize:** before the decision deadline, restore traffic to the unchanged
   source if acceptance criteria fail. Otherwise monitor closely, protect the target, right-size
   from measured data, and decommission source systems only after retention/rollback obligations.

### Replatform a database with schema conversion and DMS

1. Inventory schemas, extensions, stored code, large objects, transaction rates, character sets,
   time zones and consumers. Run DMS Schema Conversion where supported, or SCT for the legacy
   desktop fallback, and resolve or explicitly accept every incompatible object.
2. Establish network and DNS reachability from the DMS replication resources to source and target;
   use least-privilege database users, Secrets Manager where supported, TLS and adequate KMS grants.
3. Size and test the target, apply converted schema, then run DMS **full load + CDC**. Monitor task
   errors, table statistics, validation failures, latency and source transaction-log retention.
4. Rehearse application compatibility and performance with representative data. Validate counts,
   checksums or business aggregates; DMS task completion alone is not proof of correctness.
5. At cutover, quiesce writers, allow CDC latency to reach the agreed threshold, perform final
   reconciliation, switch application configuration/DNS or secrets, and run smoke/business tests.
6. Keep the source protected and unchanged until the rollback deadline. Reversing writes is a
   separate replication design; do not promise rollback after new writes reach the target unless
   that path has been built and rehearsed.

Both runbooks must include communications, escalation contacts, security validation, monitoring,
capacity checks, a timestamped decision log and post-cutover cost/operability work. “Minimal
downtime” is an outcome demonstrated by rehearsal, not a property granted automatically by MGN
or DMS.

---

## Key Exam Points

- The current framework has **7 R's**, including **relocate**; “refactor” and “re-architect”
  refer to the same strategy in current AWS migration guidance.
- **Rehost = lift-and-shift = MGN.** MGN is the **successor to SMS** (SMS is deprecated).
- **DMS** keeps the source database **available during migration** (CDC ongoing replication).
- **Heterogeneous DB migration = schema conversion + DMS.** Prefer DMS Schema Conversion where
  supported; recognize SCT as the desktop fallback and in older exam wording.
- **DataSync = online** bulk file transfer to S3/EFS/FSx. Physical-transfer choices changed:
  Data Transfer Terminal/partners for new customers, Snowball Edge mostly for existing customers
  and legacy exam recognition.
- **AWS Transform** is the current new-project assessment/discovery path. Migration Hub and
  Application Discovery Service remain existing-customer and older-exam recognition; none of them
  performs the workload migration.
- **Transfer Family** = managed **SFTP/FTPS/FTP** into S3/EFS, for ongoing partner transfers.

---

## Common Mistakes

- ❌ Choosing **SMS** for a new server migration — it's deprecated; the answer is **MGN**.
- ❌ Moving Oracle → PostgreSQL data without schema/code conversion. Use DMS Schema Conversion
  where supported, or SCT as the fallback, before DMS data cutover.
- ❌ Picking **DataSync** when the data volume vs. bandwidth requires a physical-transfer option.
  In current new-customer designs, check Data Transfer Terminal/partners; in older exam wording,
  recognize Snowball.
- ❌ Starting a new assessment on Migration Hub/Application Discovery Service after their new-
  customer closure; use AWS Transform. Existing projects can continue using those services.
- ❌ Confusing **DataSync** (migration/sync jobs) with **Transfer Family** (standing SFTP
  endpoints) or **Storage Gateway** (hybrid storage, not migration).

---

**Next**: [03_backup.md — AWS Backup: Centralized Data Protection](03_backup.md)
