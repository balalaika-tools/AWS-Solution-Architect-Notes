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

A **hybrid** appliance deployed on-premises that presents standard storage protocols while storing
data in AWS. New deployments use a supported VM platform; the AWS Storage Gateway Hardware Appliance
has not been offered to new customers since May 2025. The gateway caches frequently used data locally
for low latency.

| Gateway type | On-prem protocol | Backed by | Use when |
|--------------|------------------|-----------|----------|
| **Amazon S3 File Gateway** | **NFS / SMB** file share | **S3** objects | On-prem apps want a file share but you want files stored as S3 objects (with lifecycle to Glacier) |
| **Amazon FSx File Gateway** | SMB | **FSx for Windows** | Existing-customer recognition for cached on-prem access; unavailable to new customers since October 2024 |
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

⚠️ **Current availability check (July 2026)**: Snow is now mostly a legacy/existing-customer answer.
**Snowcone was discontinued on November 12, 2024**, and **Snowball Edge is no longer available to
new customers as of November 7, 2025**. Existing Snowball Edge customers can continue using it, but
AWS directs new customers toward **DataSync** for online transfer, **AWS Data Transfer Terminal** or
partner solutions for physical transfer, and **Outposts** for edge compute.

### Current design choices

| Need | Current answer |
|------|----------------|
| Transfer TB-PB over a usable network | **DataSync** over internet, VPN, or Direct Connect |
| Eligible AWS Enterprise customer brings storage devices to a supported secure AWS physical location for upload | **AWS Data Transfer Terminal** |
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

## 6. Migration Planning: Size Is Only the First Input

Do not choose a transfer service from "terabytes" or "petabytes" alone. A successful migration must
move the initial dataset, keep up with changes, preserve required metadata, prove integrity, and
switch applications inside an agreed outage window.

### 1. Measure the transfer workload

| Input | What to measure | Why it changes the design |
|-------|-----------------|---------------------------|
| **Dataset** | Used bytes, file/object count, size distribution, sparse files, and required metadata/ACLs | Millions of tiny files can be limited by source metadata operations even when the total byte count is modest |
| **Change rate** | New/modified/deleted bytes per hour and peak write periods | If changes arrive as fast as the transfer can copy them, incremental passes never converge to a cutover-sized delta |
| **Usable bandwidth** | Measured throughput after business traffic, packet loss, source read speed, destination writes, and protocol overhead | Link speed is only a ceiling; a nominal 10-Gbps circuit does not guarantee a 10-Gbps copy |
| **Transfer window** | Calendar deadline, permitted transfer hours, and maximum application write freeze | Determines whether online seed + deltas can finish or physical transfer is required |
| **Security** | Classification, in-transit path, at-rest encryption, KMS ownership, endpoint policy, custody, and audit evidence | The fastest path is invalid if keys, principals, or physical handling violate the control boundary |
| **Validation** | Checksums, counts, bytes, ownership, ACLs, timestamps, application reads, and performance acceptance | "Task succeeded" does not prove that the target is usable by the application |

Estimate a lower bound before selecting a service:

```
transfer time (seconds) = data size (bits) / measured usable throughput (bits/second)
```

Then add time for scanning millions of entries, retries, verification, final incremental passes, and
operational contingency. Run a representative pilot; do not substitute a bandwidth calculator for
source-and-target testing.

### 2. Choose the movement path

| Condition | Primary choice | Important trade-off |
|-----------|----------------|---------------------|
| Initial copy and recurring deltas fit over the measured network | **DataSync** over internet/VPN | Managed incremental transfer, checksums, scheduling, and metadata handling; consumes WAN and source/destination performance |
| Online transfer is viable but needs predictable, ongoing private capacity | **DataSync over Direct Connect** | Direct Connect is the link and has provisioning/circuit cost and lead time; DataSync remains the transfer engine |
| Network cannot meet the seed deadline, data is on portable storage, and a supported facility is reachable | **Data Transfer Terminal** or an AWS Partner physical-transfer service | Customer transports and operates its devices; Terminal availability is location-limited and currently restricted to AWS Enterprise customers |
| Existing customer already approved for shipped AWS appliances | **Snowball Edge** | Useful for a physical seed, but unavailable to new customers and still requires online/another process for changes after the device copy |
| Partners must continue sending files with SFTP/FTPS/FTP/AS2 | **Transfer Family** | A managed ingestion endpoint, not a general filesystem-migration or bulk-copy engine |
| The application remains on-premises and needs ongoing NFS/SMB/iSCSI/VTL access | **Storage Gateway** | Hybrid access and caching, not a completed migration; gateway/cache availability remains an operational dependency |

Physical transfer is not automatically faster. Include time to stage data to the device, transport it
to a facility, upload/import, validate, and then copy the changes created while the seed was in
motion. A high change rate usually requires an online incremental tool even when the baseline travels
physically.

### 3. Secure and validate the copy

- Encrypt DataSync's network path and configure target encryption explicitly. Grant the DataSync
  role only the required source/destination and KMS permissions; test key policy access before the
  migration window.
- Decide which POSIX ownership/mode, Windows ACL/audit data, timestamps, links, tags, and object
  metadata must survive. Source and destination systems do not always express the same semantics.
- Use DataSync checksum verification for transferred data by default. Use a full point-in-time
  comparison when the mode and destination support it and the extra scan fits the window.
- Reconcile independent totals and sample business-critical files. Then run application correctness
  and performance tests with production-like identities—not only an administrator account.
- Keep task reports, CloudWatch logs, manifests, checksum evidence, and exception decisions for the
  migration record.

### 4. Cut over with a rollback boundary

A low-downtime file migration normally follows this sequence:

1. Run the full seed while the source remains live.
2. Run repeated **changed-only** passes and measure whether the remaining delta is converging.
3. Rehearse the runbook, including DNS/mount changes, identity/ACL checks, application smoke tests,
   and explicit go/no-go owners.
4. At cutover, stop or fence source writes, take a final backup, run and verify the final delta, then
   redirect clients to the target.
5. Keep the source intact and read-only through a defined observation period. Decommission it only
   after reconciliation, application acceptance, backup, and restore testing pass.

Define rollback **before** cutover. Specify the failure thresholds, decision owner, traffic/mount
reversal, and what happens to writes accepted by the target. If the target accepts new writes, simply
pointing clients back makes the old source stale; either reverse-sync those changes, use a designed
dual-write mechanism, or treat the migration as fix-forward after that boundary.

> **Default**: Use DataSync for the seed and incremental synchronization when a measured pilot fits
> the window. Add Direct Connect for justified ongoing network requirements. Use physical transfer
> for a seed only when the end-to-end physical timeline beats the online plan and the final delta and
> rollback procedures are still feasible.

---

## Key Exam Points

- **Storage Gateway = hybrid access** with on-prem protocols backed by AWS: **S3 File Gateway** (NFS/
  SMB→S3), **Volume Gateway** (iSCSI→EBS snapshots, Cached/Stored), and **Tape Gateway**
  (VTL→S3/Glacier, replaces physical tape). FSx File Gateway is existing-customer-only.
- **DataSync = online migration & sync** (NFS/SMB/HDFS/object → S3/EFS/FSx), incremental, scheduled,
  validated; runs over internet or Direct Connect.
- **Transfer Family = managed SFTP/FTPS/FTP** landing files in S3/EFS for partner file exchange.
- **Snow Family is legacy/currently limited**: Snowcone is discontinued and Snowball Edge is available
  only to existing customers. Recognize Snowball/Snowmobile in older exam wording, but for current new
  customers prefer **DataSync**, **Data Transfer Terminal**, partner solutions, or **Outposts** for edge.
- Migration selection uses dataset shape, change rate, measured bandwidth, transfer/cutover window,
  encryption and metadata needs, validation, and rollback—not volume alone.
- **DataSync** is the usual online seed/delta engine; Direct Connect can provide the link. Physical
  transfer solves the baseline only and still needs a plan for validation and changes made in flight.

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
- ❌ Dividing bytes by advertised link speed and treating that as the migration plan. File count,
  source/destination limits, change rate, verification, and retry time determine the real window.
- ❌ Cutting over without fencing source writes or defining how target-side writes return during a
  rollback. Two writable copies without conflict handling create data loss, not resilience.

---

**Next**: [../06_databases/01_database_fundamentals.md — Database Fundamentals: SQL vs NoSQL](../06_databases/01_database_fundamentals.md)
