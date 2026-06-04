# Storage Gateway, DataSync, Transfer Family & Physical Transfer

> **Who this is for**: Engineers who know the core AWS storage services
> ([01_ebs.md](01_ebs.md)–[04_s3_storage_classes_and_management.md](04_s3_storage_classes_and_management.md))
> and now need the **hybrid and data-transfer** toolset: how on-premises systems reach AWS storage,
> and how to move bulk data in. The exam tests *which transfer service for which volume/latency/
> connectivity scenario*. See also [../14_hybrid_migration_dr/02_migration_services.md](../14_hybrid_migration_dr/02_migration_services.md).

---

## 1. The Hybrid Storage Problem

On-premises applications often can't be rewritten to call the S3 API directly — they expect a local
file share, an iSCSI disk, or a tape library. And bulk data (terabytes to petabytes) may be too large
or too slow to push over the internet. AWS offers two categories of solution:

- **Storage Gateway** — keep using familiar protocols (NFS/SMB/iSCSI/VTL) on-prem, backed by AWS storage.
- **Transfer services** (DataSync, Transfer Family, Data Transfer Terminal, legacy Snow patterns) —
  move data **into/out of** AWS, online or offline.

---

## 2. AWS Storage Gateway

A **hybrid** virtual appliance (VM or hardware) deployed on-premises that presents standard storage
protocols while storing data in AWS. Caches frequently used data locally for low latency.

| Gateway type | On-prem protocol | Backed by | Use when |
|--------------|------------------|-----------|----------|
| **Amazon S3 File Gateway** | **NFS / SMB** file share | **S3** objects | On-prem apps want a file share but you want files stored as S3 objects (with lifecycle to Glacier) |
| **Amazon FSx File Gateway** | SMB | **FSx for Windows** | Low-latency on-prem access to Windows file shares in FSx |
| **Volume Gateway** | **iSCSI** block volumes | **EBS snapshots** in S3 | Block storage for on-prem apps; **Cached** (primary in S3, hot data local) or **Stored** (primary local, async backup to S3) |
| **Tape Gateway (VTL)** | **iSCSI VTL** (virtual tape) | S3 / Glacier | Replace physical tape backups; existing backup software writes "tapes" to AWS |

```
   On-prem app ──NFS/SMB/iSCSI/VTL──► Storage Gateway (cache) ──► AWS (S3 / EBS snap / Glacier)
```

💡 Exam tells: "on-prem app needs a **file share** but store in S3" → **S3 File Gateway**. "Replace
**physical tape** backups" → **Tape Gateway**. "On-prem **block** volumes backed by AWS" → **Volume
Gateway**.

---

## 3. AWS DataSync

**DataSync** is an **online** data-transfer service for **moving and continuously syncing large
amounts of data** between on-prem (NFS/SMB/HDFS/object) and AWS (S3, EFS, FSx) — or between AWS
storage services.

- Uses an agent on-prem; transfers over the network (internet or **Direct Connect**), with encryption,
  scheduling, integrity validation, and incremental sync.
- Can move **TB–PB** over the wire — fast, but bounded by your available bandwidth.
- Great for **one-time migrations** and **recurring replication** (e.g., nightly sync to S3).

⚠️ Don't confuse DataSync (one-time/scheduled **migration & sync**) with Storage Gateway (ongoing
**hybrid access** via standard protocols). DataSync moves data *to* AWS; Storage Gateway lets on-prem
apps keep *using* data backed by AWS.

---

## 4. AWS Transfer Family

Fully managed **SFTP / FTPS / FTP** (and AS2) endpoints that store/retrieve data directly in **S3 or
EFS**. Use it when partners or legacy systems must exchange files over standard file-transfer
protocols without you running an FTP server.

- Authentication via service-managed users, IAM, or your existing directory (AD/LDAP).
- ✅ "Third parties upload files via **SFTP** and we want them landing in S3, fully managed" → Transfer Family.

---

## 5. Offline Bulk Transfer — Snow Status and Data Transfer Terminal

When data is too large or your connection too slow/expensive to move online, the historical AWS answer
was the **Snow Family**: AWS shipped a rugged appliance, you copied data locally, shipped it back, and
AWS imported it to S3.

⚠️ **Current availability check (June 2026)**: Snow is now mostly a legacy/existing-customer answer.
**Snowcone was discontinued on November 12, 2024**, and **Snowball Edge is no longer available to
new customers as of November 7, 2025**. Existing Snowball Edge customers can continue using it, but
AWS directs new customers toward **DataSync** for online transfer, **AWS Data Transfer Terminal** or
partner solutions for physical transfer, and **Outposts** for edge compute.

### Current design choices

| Need | Current answer |
|------|----------------|
| Transfer TB-PB over a usable network | **DataSync** over internet, VPN, or Direct Connect |
| Bring your own storage devices to a secure AWS physical location for upload | **AWS Data Transfer Terminal** |
| Shipped rugged AWS appliance | **Snowball Edge** only for existing Snow customers / legacy exam recognition |
| Edge compute with AWS APIs on-premises | **AWS Outposts** for new designs; Snowball Edge only for existing Snow customers |

### Legacy exam recognition

```
   On-prem data ──copy──► [ Snow device ] ──ship──► AWS ──import──► S3
```

| Device/concept | What to remember |
|----------------|------------------|
| **Snowcone** | Small rugged device; discontinued, so only recognize it in old material |
| **Snowball Edge Storage Optimized** | Large shipped appliance for bulk migration; newer devices reached much higher capacity than the old 80 TB exam table |
| **Snowball Edge Compute Optimized** | Shipped appliance with local compute for disconnected/edge workloads; existing customers only now |
| **Snowmobile** | Exabyte-scale data-center evacuation concept (100 PB truck); legacy recognition, not a greenfield default |

⚠️ If an older exam-style question says "ship a device because the line would take weeks," **Snowball
Edge** is the recognition answer. If the question is a current architecture design with a new customer,
prefer **DataSync** or **Data Transfer Terminal** depending on whether the data can move online.

---

## 6. Choosing: Snow vs DataSync vs Direct Connect

The decision hinges on **data volume**, **available bandwidth**, **deadline**, and whether the need is
**one-time** or **ongoing**.

| Scenario | Choose |
|----------|--------|
| Moderate data, decent bandwidth, **online** one-time or recurring sync | **DataSync** |
| **Huge** data but online transfer is feasible | **DataSync** over Direct Connect/VPN/internet |
| New customer needs physical transfer with portable media | **AWS Data Transfer Terminal** or AWS Partner solution |
| Existing Snow customer / legacy exam shipped-device scenario | **Snowball Edge** |
| Exabyte-scale data-center shutdown in older material | **Snowmobile** (legacy recognition) |
| **Persistent, private, high-bandwidth** dedicated link for ongoing hybrid traffic | **Direct Connect** (often *paired with* DataSync) |
| Ongoing hybrid **access** with standard protocols (NFS/SMB/iSCSI/tape) | **Storage Gateway** |

> **Rule of thumb**: A common exam heuristic — if transferring the data over your current connection
> would take **more than ~1 week**, older material says to ship it on **Snow**. For a current new-customer
> design, check whether **Data Transfer Terminal** or a partner physical-transfer option is the supported
> equivalent. If you need a permanent fast pipe for ongoing transfers, that's **Direct Connect**;
> DataSync rides over it for the actual copy.

💡 Direct Connect provides the *link*; DataSync provides the *transfer engine*. They're complementary,
not either/or.

---

## Key Exam Points

- **Storage Gateway = hybrid access** with on-prem protocols backed by AWS: **S3 File Gateway** (NFS/
  SMB→S3), **FSx File Gateway** (SMB→FSx), **Volume Gateway** (iSCSI→EBS snapshots, Cached/Stored),
  **Tape Gateway** (VTL→S3/Glacier, replaces physical tape).
- **DataSync = online migration & sync** (NFS/SMB/HDFS/object → S3/EFS/FSx), incremental, scheduled,
  validated; runs over internet or Direct Connect.
- **Transfer Family = managed SFTP/FTPS/FTP** landing files in S3/EFS for partner file exchange.
- **Snow Family is legacy/currently limited**: Snowcone is discontinued and Snowball Edge is available
  only to existing customers. Recognize Snowball/Snowmobile in older exam wording, but for current new
  customers prefer **DataSync**, **Data Transfer Terminal**, partner solutions, or **Outposts** for edge.
- Decision: small/online → **DataSync**; huge/slow link → **Snow**; permanent fast pipe → **Direct
  Connect** (DataSync runs over it); ongoing protocol access → **Storage Gateway**. For new physical
  transfer designs, check **Data Transfer Terminal** before assuming Snow.

---

## Common Mistakes

- ❌ Confusing DataSync (move/migrate data) with Storage Gateway (ongoing hybrid access). Different jobs.
- ❌ Memorizing old Snowcone/Snowball capacities as current product choices. Snow is now mostly
  legacy/existing-customer; current new-customer designs should evaluate DataSync, Data Transfer
  Terminal, partners, or Outposts.
- ❌ Choosing Snowmobile for tens of TB — even in older material, that's Snowball Edge or DataSync;
  Snowmobile is exabyte-scale legacy recognition.
- ❌ Picking Tape Gateway when there's no existing tape-backup software — File or Volume Gateway fits better.
- ❌ Thinking Direct Connect *transfers* the data itself — it's the link; you still need DataSync (or similar) to move it.
- ❌ Running your own FTP server on EC2 when **Transfer Family** offers a fully managed SFTP/FTPS endpoint to S3/EFS.

---

**Next**: [../06_databases/01_database_fundamentals.md — Database Fundamentals: SQL vs NoSQL](../06_databases/01_database_fundamentals.md)
