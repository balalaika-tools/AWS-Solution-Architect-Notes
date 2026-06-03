# Migration Services: Moving Workloads into AWS

> **Who this is for**: An engineer planning to move existing servers, databases, and files
> from a data center into AWS, who needs to know *which* AWS tool fits *which* job. Assumes
> you've read [01_hybrid_networking.md](01_hybrid_networking.md) — most online migration tools
> need that hybrid link first.

---

## 1. First, the Strategy: The 6 R's

Before tooling, AWS frames every workload by **how** you migrate it. These are the **6 R's** —
strategies in roughly increasing order of effort:

| R | Meaning | Example |
|---|---------|---------|
| **Rehost** | "Lift and shift" — move as-is, no code change | Copy a VM to EC2 unchanged |
| **Replatform** | "Lift, tinker, shift" — small optimizations | Move a self-managed MySQL to RDS |
| **Repurchase** | Drop the old product, buy a SaaS replacement | Self-hosted CRM → Salesforce |
| **Refactor** | Re-architect for cloud-native | Monolith → serverless microservices |
| **Retire** | Turn it off — nobody uses it | Decommission a legacy report server |
| **Retain** | Leave it on-prem for now | Mainframe that can't move yet |

> **Key insight**: The exam mostly cares about **Rehost** (→ Application Migration Service) and
> **Replatform** of databases (→ DMS). "Lift and shift" is the phrase that maps to rehost.

---

## 2. AWS Migration Hub — The Dashboard

**Migration Hub** is the **single pane of glass** that tracks migration progress across
multiple AWS migration tools and Regions. It doesn't move anything itself — it **discovers**
your on-prem inventory (via the Discovery Agent / Agentless Collector) and shows the status of
servers and applications being migrated by MGN, DMS, and partners.

💡 Think of Migration Hub as project management; the services below do the actual work.

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

## 4. Database Migration Service (DMS) + Schema Conversion Tool (SCT)

**AWS Database Migration Service (DMS)** migrates **databases** to AWS with the **source
staying online** during the migration. It supports one-time loads and **ongoing replication
(CDC — change data capture)** so the target stays in sync until you cut over.

Two flavors of database migration:

| Migration type | Engines | Tool needed |
|----------------|---------|-------------|
| **Homogeneous** (same engine) | Oracle → Oracle, MySQL → MySQL/RDS | **DMS** alone |
| **Heterogeneous** (different engine) | Oracle → Aurora PostgreSQL, SQL Server → MySQL | **SCT first, then DMS** |

The **Schema Conversion Tool (SCT)** converts the **schema and code** (tables, stored
procedures, views) from the source engine's dialect to the target's. DMS then moves the
**data**.

```
 Heterogeneous migration:
   Source DB ──schema/code──▶ [ SCT ] ──converted schema──▶ Target DB
   Source DB ──── data + ongoing CDC ──▶ [ DMS ] ──▶ Target DB
```

> **Rule**: **Same engine → DMS only. Different engine → SCT (schema) + DMS (data).** This
> pairing is one of the most reliable exam questions in this whole domain.

💡 DMS runs on a **replication instance** (an EC2-backed managed resource) that needs network
reach to both source and target — hence the hybrid link from file 01.

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

❌ DataSync is **online** — if the data is huge and bandwidth is the bottleneck, use the Snow
Family instead.

---

## 6. Snow Family — Offline Transfer

When you have **petabytes** of data and shipping a physical device beats waiting weeks on a
network link, use the **Snow Family** (Snowball Edge, Snowmobile). You load data onto a rugged
device and ship it to AWS, where it's loaded into S3.

> Covered in depth in
> [../05_storage/06_storage_gateway_and_transfer.md](../05_storage/06_storage_gateway_and_transfer.md).

💡 Quick rule of thumb: if transferring the data online would take **more than a week**, Snow
is usually cheaper and faster.

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
| Migrate a database to a **different engine** | **SCT** (schema) + **DMS** (data) |
| Track overall migration progress across tools | **Migration Hub** |
| Move **file/NFS/object data online** to S3/EFS/FSx | **DataSync** |
| Move **petabytes offline** (bandwidth-limited) | **Snow Family** |
| **Ongoing SFTP/FTPS** file exchange into S3/EFS | **Transfer Family** |
| Discover on-prem inventory before migrating | **Migration Hub** discovery / collectors |

---

## Key Exam Points

- **Rehost = lift-and-shift = MGN.** MGN is the **successor to SMS** (SMS is deprecated).
- **DMS** keeps the source database **available during migration** (CDC ongoing replication).
- **Heterogeneous DB migration = SCT + DMS.** Homogeneous = **DMS alone**.
- **DataSync = online** bulk file transfer to S3/EFS/FSx; **Snow = offline** for huge data.
- **Migration Hub** only **tracks/discovers**; it doesn't perform migrations.
- **Transfer Family** = managed **SFTP/FTPS/FTP** into S3/EFS, for ongoing partner transfers.

---

## Common Mistakes

- ❌ Choosing **SMS** for a new server migration — it's deprecated; the answer is **MGN**.
- ❌ Using **DMS alone** for an Oracle → PostgreSQL move. Different engines need **SCT** first.
- ❌ Picking **DataSync** when the data volume vs. bandwidth screams **Snowball** (offline).
- ❌ Thinking **Migration Hub** moves data — it's a dashboard.
- ❌ Confusing **DataSync** (migration/sync jobs) with **Transfer Family** (standing SFTP
  endpoints) or **Storage Gateway** (hybrid storage, not migration).

---

**Next**: [03_backup.md — AWS Backup: Centralized Data Protection](03_backup.md)
