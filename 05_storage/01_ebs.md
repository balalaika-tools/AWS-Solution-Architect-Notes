# EBS — Elastic Block Store

> **Who this is for**: Engineers who know what an EC2 instance is and want the block-storage
> story for SAA-C03. We start with the prerequisite — block vs file vs object storage — because
> picking the right *type* is the most common storage question on the exam. Then we go deep on EBS:
> volume types, snapshots, encryption, resizing, and the instance-store contrast.

---

## 1. Prerequisite: Block vs File vs Object Storage

Before any AWS service, you must be able to tell these three storage paradigms apart. They differ
in **what the unit of storage is** and **how you address it**.

| | **Block storage** | **File storage** | **Object storage** |
|---|---|---|---|
| Unit | Fixed-size blocks (raw disk) | Files in a directory tree | Objects (data + metadata + key) |
| Addressed by | Block address (the OS builds a filesystem on top) | Path: `/dir/sub/file.txt` | Flat key: `reports/2026/q1.pdf` |
| Protocol | Attached as a disk (NVMe/iSCSI) | NFS (Linux) / SMB (Windows) | HTTP REST API (GET/PUT) |
| Shared access | One instance at a time (mostly) | Many clients mount it concurrently | Massively concurrent over HTTP |
| Mutability | Edit any byte in place | Edit any byte in place | Replace whole object (no partial edit) |
| Typical use | Boot volumes, databases, low-latency disk | Shared content, home dirs, lift-and-shift | Backups, media, data lakes, static assets |
| AWS service | **EBS**, instance store | **EFS**, **FSx** | **S3** |

```
BLOCK (EBS)              FILE (EFS/FSx)            OBJECT (S3)
┌────────────┐          ┌───────────────┐         ┌─────────────────┐
│ EC2 sees   │          │ many EC2 mount│         │ key → object    │
│ a raw disk │          │ same NFS tree │         │ via HTTPS API   │
│ /dev/xvdf  │          │ /mnt/shared   │         │ GET /bucket/key │
└─────┬──────┘          └──────┬────────┘         └────────┬────────┘
   1 instance               N instances              N clients, internet-scale
```

> **Rule of thumb**: A single server needs a fast disk → **block (EBS)**. Many servers must share
> the *same files* with POSIX/SMB semantics → **file (EFS/FSx)**. You access data over HTTP at any
> scale and never need a filesystem → **object (S3)**.

💡 Exam tell: "needs a filesystem mounted by multiple instances" = file storage, not EBS. "Highly
durable, internet-accessible, no filesystem" = object/S3.

---

## 2. What EBS Is

**EBS (Elastic Block Store)** provides **network-attached block storage volumes** for EC2. A volume
behaves like a raw hard disk: you attach it, format it, and the OS mounts a filesystem.

Key properties that drive exam answers:

- **AZ-scoped.** A volume lives in **one Availability Zone** and can only attach to an instance in
  that *same AZ*. To move it to another AZ, snapshot it and restore the snapshot into the new AZ.
- **Network drive, not local.** EBS is reached over the network (low-latency, but not physically
  inside the host). This is why a volume can survive instance termination and be detached/reattached.
- **Decoupled lifecycle.** By default the root volume is deleted on instance termination
  (`DeleteOnTermination = true`), but you can disable that, and additional data volumes persist.
- **One instance at a time** — except for **Multi-Attach** on io1/io2 (see §7).

```
        Availability Zone A
   ┌───────────────────────────────┐
   │  EC2 instance                 │
   │   │ (network)                 │
   │   ▼                           │
   │  EBS volume  (gp3, 100 GiB)   │  ← cannot attach to an instance in AZ-B
   └───────────────────────────────┘
```

⚠️ EBS volumes do **not** span AZs and do **not** stretch across Regions. That AZ boundary is a
favorite trap: "attach this volume to an instance in another AZ" is impossible without a snapshot.

---

## 3. EBS Volume Types

There are two families: **SSD-backed** (measured in IOPS, good for transactional/random I/O) and
**HDD-backed** (measured in throughput MB/s, good for large sequential I/O). Boot volumes must be SSD.

| Type | Family | Max IOPS | Max throughput | Best for | Notes |
|------|--------|----------|----------------|----------|-------|
| **gp3** | SSD (general purpose) | 16,000 | 1,000 MB/s | Default for most workloads, boot volumes | IOPS/throughput provisioned **independently** of size; cheaper than gp2 |
| **gp2** | SSD (general purpose) | 16,000 | 250 MB/s | Legacy general purpose | IOPS scale with size (3 IOPS/GiB), burst to 3,000 |
| **io1** | SSD (provisioned IOPS) | 64,000 | 1,000 MB/s | High-perf databases | Up to 50 IOPS/GiB; Multi-Attach capable |
| **io2 / io2 Block Express** | SSD (provisioned IOPS) | 256,000 (Block Express) | 4,000 MB/s | Largest databases, mission-critical, low-latency | Higher durability (99.999%); 1,000 IOPS/GiB; Multi-Attach |
| **st1** | HDD (throughput optimized) | 500 | 500 MB/s | Big data, log processing, data warehouses (sequential) | Cannot be a boot volume |
| **sc1** | HDD (cold) | 250 | 250 MB/s | Infrequently accessed, lowest cost | Cannot be a boot volume |

> **Key insight**: **gp3 is the modern default.** Unlike gp2 (where IOPS are tied to volume size),
> gp3 lets you dial IOPS and throughput separately from capacity — so you stop over-provisioning
> disk just to get IOPS. Move to **io2 Block Express** only when you need extreme, sustained IOPS
> (>16,000) or sub-millisecond latency for a demanding database.

✅ "Need a boot volume" or "general workload" → **gp3**.
✅ "Throughput-heavy sequential workload, big files, lowest cost per GB for active data" → **st1**.
✅ "Rarely accessed archival block data, cheapest" → **sc1**.
✅ "Mission-critical database needing 64,000+ IOPS / highest durability" → **io2 Block Express**.

⚠️ **HDD volumes (st1/sc1) cannot be boot volumes.** A boot/root volume must be an SSD type.

---

## 4. Snapshots

A **snapshot** is a **point-in-time backup of a volume stored in Amazon S3** (in AWS-managed storage
you don't see as a bucket).

- **Incremental.** Only the blocks changed since the last snapshot are saved, so subsequent snapshots
  are fast and cheaper — but each snapshot is independently restorable (deleting one doesn't corrupt
  others).
- **Region-scoped, but copyable.** Snapshots live in a Region; you can **copy a snapshot to another
  Region** (for DR) or **share/copy to another account**. Copying is how you move a volume's data
  across AZs *and* Regions.
- **Snapshot → AMI.** You can create an **AMI** from a snapshot (or from a running instance, which
  snapshots its volumes). The AMI lets you launch new instances pre-loaded with that disk image. An
  AMI is Region-scoped and must be copied to launch in another Region.
- **Restore.** Restoring a snapshot creates a *new* volume — choose any AZ. New volumes restored from
  snapshot may have first-touch latency until blocks are lazily loaded (use **Fast Snapshot Restore**
  to pre-warm and avoid it).
- **Archive tier.** **EBS Snapshots Archive** moves rarely-needed snapshots to a cheaper tier
  (retrieval takes 24–72 h), and **Recycle Bin** lets you set retention rules to recover deleted ones.

```
Volume ──snapshot──► S3-backed store ──copy──► other Region (DR)
            │                            └──share──► other account
            └──create AMI──► launch new instances from the image
```

💡 Snapshots are the canonical answer for **EBS backup**, **cross-AZ/cross-Region volume movement**,
and the basis for **golden AMIs**. Automate them with **Data Lifecycle Manager (DLM)** or AWS Backup.

### How incremental storage affects retention

Incremental does **not** mean that a restore depends on keeping every earlier snapshot. EBS tracks
the blocks needed by each point in time, so any retained snapshot can create a complete volume. If
you delete a snapshot, EBS removes only blocks that no other snapshot still references.

This has two planning consequences:

- A daily snapshot normally adds only changed blocks, but high-churn volumes can still grow backup
  cost quickly.
- Deleting one old snapshot might save little if newer snapshots reference most of its blocks. Use
  retention policies and actual snapshot usage, not snapshot count alone, to forecast savings.

For applications that write across several volumes, take a multi-volume snapshot or coordinate an
application/filesystem freeze. A set of individually timed crash-consistent snapshots might not
represent one recoverable application point.

### Restore performance: lazy loading vs Fast Snapshot Restore

A normal volume restored from a snapshot downloads blocks when they are first read. This can make
the first scan or database startup slower. Choose one of these approaches deliberately:

| Approach | Use when | Trade-off |
|----------|----------|-----------|
| **Normal restore** | Recovery can tolerate first-touch latency | No acceleration charge; warm the required blocks before production traffic if predictable performance matters |
| **Provisioned initialization rate** | You need a predictable initialization window for selected restores | Paid initialization with a configured download rate; capacity quotas apply |
| **Fast Snapshot Restore (FSR)** | Newly created volumes must deliver provisioned performance immediately | Enable it for each **snapshot–AZ pair**; it is billed while enabled and consumes a per-Region quota |

FSR is not inherited by a copied or newly created snapshot. Enable it only in the AZs where a
recovery runbook will create volumes, and disable it after the recovery or launch wave if it is not
needed continuously.

### Standard tier, Snapshot Archive, and Recycle Bin

| Feature | Problem it solves | Important boundary |
|---------|-------------------|--------------------|
| **Snapshot Archive** | Low-cost retention for rarely restored monthly, yearly, or compliance snapshots | Archiving converts the incremental snapshot into a **full** snapshot. Restore to the standard tier takes **24–72 hours**, and the archive tier has a **90-day minimum billing period**, so it is not the tier for a tight RTO or short retention. An archived snapshot cannot be used, shared, or enabled for FSR until restored. |
| **Recycle Bin** | Recovery after an operator or automation deletes a protected volume, snapshot, or EBS-backed AMI | A retention rule must match the resource before deletion. The retained resource continues to incur normal standard/archive storage charges and becomes unrecoverable after the retention period. |

Archive economics must be calculated using the full used-block size, not the incremental size that
the snapshot occupied in the standard tier. Keep recent operational recovery points in the standard
tier and archive only points whose lower storage price justifies the slower restore and full-snapshot
footprint.

### Cross-account encrypted snapshot copies

Use a dedicated backup account to protect recovery points from compromise or deletion in the
workload account. For an encrypted EBS snapshot, sharing the snapshot alone is insufficient:

1. Encrypt the source with a **customer managed KMS key**; the AWS managed `aws/ebs` key cannot be
   shared across accounts.
2. Allow the destination account to use the source KMS key, and grant the copy role the required EBS
   and KMS decrypt/re-encrypt permissions.
3. Share the snapshot, then **copy** it in the destination account and encrypt the copy with a
   destination-owned customer managed KMS key.
4. Test a restore from the destination copy. Once copied, its availability no longer depends on the
   workload account continuing to share the source key or snapshot.

Changing KMS keys or starting a new destination copy lineage can force a full copy. Cross-Region
transfer, copied snapshot storage, and KMS requests also add cost.

### DLM vs AWS Backup

| Need | Choose | Why |
|------|--------|-----|
| EBS snapshot and EBS-backed AMI schedules inside an account/Region | **Amazon Data Lifecycle Manager** | EBS-native, tag-driven lifecycle policies with retention, archive, FSR, and copy options; no DLM service charge |
| One protection policy across EBS and other services, accounts, and Regions | **AWS Backup** | Organization backup policies, centralized job monitoring, backup vault access policies, Vault Lock, and restore testing |
| Automated copies into an isolated account | Either, after checking the workflow | DLM uses a source sharing policy plus a destination copy-event policy; AWS Backup uses cross-account vault copies within AWS Organizations |

Do not manage the same recovery points independently with both tools: ownership, retention, and
deletion become hard to reason about. Snapshots created by AWS Backup must be deleted through AWS
Backup, not the EC2 snapshot workflow.

### Quota and cost review before rollout

Before a fleet-wide policy starts, check Service Quotas in every source and destination Region for
concurrent snapshot copies, archive/restore operations, FSR-enabled snapshots, and provisioned
initialization rate. Stagger schedules so thousands of volumes do not all copy or initialize at the
same time.

Model all five cost drivers: changed-block storage in the standard tier, full-snapshot archive
storage and retrieval, cross-Region transfer and destination copies, FSR or provisioned
initialization, and extra retention in Recycle Bin. A backup is complete only after a timed restore
test proves the application can meet its RTO and RPO.

---

## 5. Encryption

EBS integrates with **AWS KMS** for encryption at rest, in transit between instance and volume, and
for snapshots.

- Encryption is **transparent** — no app changes, negligible performance impact.
- **Encrypt by default** can be enabled per-Region in the account; once on, all new volumes and
  snapshots are encrypted automatically.
- **Snapshots of an encrypted volume are encrypted**; volumes restored from an encrypted snapshot are
  encrypted; AMIs built from them are encrypted.
- You **cannot directly toggle encryption on an existing unencrypted volume**. The workflow:
  snapshot the unencrypted volume → **copy the snapshot with encryption enabled** → create a new
  (encrypted) volume from that copy → attach it.

⚠️ The "encrypt an existing volume" trap: there's no in-place flip. The path is always
**snapshot → copy with encryption → restore**. Know this sequence.

---

## 6. Resizing (Elastic Volumes)

EBS volumes are **elastic** — you can modify them on a *live* attached volume with no downtime:

- **Increase size** (capacity only grows; you cannot shrink an EBS volume).
- **Change volume type** (e.g., gp2 → gp3).
- **Change provisioned IOPS / throughput** (on gp3, io1, io2).

After increasing the volume size, you must still **extend the filesystem inside the OS**
(`growpart` + `resize2fs`/`xfs_growfs` on Linux, Disk Management on Windows) — AWS grows the disk,
not the partition.

❌ You **cannot decrease** a volume's size. To "shrink," create a smaller new volume and copy data.

---

## 7. Multi-Attach

**Multi-Attach** lets a single **io1/io2** volume attach to **multiple EC2 instances (up to 16)** in
the **same AZ** simultaneously, each with full read/write.

- Requires a **cluster-aware filesystem** (e.g., GFS2) — a standard filesystem like XFS/ext4 will
  corrupt because it assumes exclusive access.
- Same-AZ only; not supported on gp2/gp3/st1/sc1.

💡 Use it for clustered/HA applications that manage their own concurrency. For ordinary "share files
across instances," the answer is **EFS**, not Multi-Attach.

---

## 8. EBS vs Instance Store

**Instance store** is *physically attached* (local NVMe/SSD on the host) — fast, but ephemeral.

| | **EBS** | **Instance store** |
|---|---|---|
| Location | Network-attached | Physically on the host |
| Persistence | **Persists** independent of instance lifecycle | **Ephemeral** — lost on stop, hibernate, or termination |
| Survives instance stop? | Yes | **No** (data gone) |
| Detach/reattach | Yes | No |
| Snapshots | Yes (to S3) | No native snapshot |
| Performance | Very high (io2 BE up to 256k IOPS), slight network overhead | Highest possible (local) — best raw IOPS/latency |
| Resizing | Elastic | Fixed (tied to instance type) |
| Cost | Pay per provisioned GB-month | Included in instance price |
| Use when | Boot volumes, databases, anything that must survive | Scratch space, caches, buffers, replicated data |

> **Key insight**: Instance store gives the **highest raw performance** but is **ephemeral**. EBS
> trades a touch of network latency for **durability and flexibility**. If a question says "data must
> survive a stop/start" → EBS. If it says "maximum I/O, temporary scratch / cache" → instance store.

⚠️ **Stopping an instance** wipes instance store but keeps EBS. Many questions hinge on exactly this.

---

## Key Exam Points

- **Block = EBS, File = EFS/FSx, Object = S3.** "Filesystem shared by many instances" → file, not EBS.
- EBS is **AZ-scoped**; move across AZs/Regions only via **snapshots** (which are stored in S3,
  incremental, copyable cross-Region and cross-account).
- Snapshot design: use **FSR** only for immediate full-performance restores, **Snapshot Archive**
  for long-RTO retention, and **Recycle Bin** for accidental deletion. Cross-account encrypted
  copies require a shareable customer managed KMS key.
- **DLM** is EBS/AMI-specific lifecycle automation; **AWS Backup** is the organization-wide,
  multi-service choice with vault controls and restore testing.
- **gp3** is the default SSD (IOPS/throughput independent of size). **io2 Block Express** for the
  highest IOPS (up to 256,000) and durability. **st1/sc1** are HDD, throughput-oriented, **cannot boot**.
- Encrypt an existing unencrypted volume: **snapshot → copy with encryption → restore**. No in-place flip.
- Volumes are **elastic**: grow size, change type, change IOPS live — but you **cannot shrink**, and
  you must extend the filesystem inside the OS afterward.
- **Multi-Attach** = io1/io2 only, same AZ, needs a cluster-aware filesystem.
- **Instance store is ephemeral** (lost on stop/terminate); **EBS persists**.

---

## Common Mistakes

- ❌ Thinking an EBS volume can attach to an instance in another AZ. It cannot — snapshot and restore.
- ❌ Believing instance-store data survives a stop/start. It is wiped on stop, hibernate, and terminate.
- ❌ Choosing st1/sc1 for a boot volume. HDD types can't be boot volumes; use gp3.
- ❌ Expecting to shrink an EBS volume. You can only grow it.
- ❌ Using EBS Multi-Attach with ext4/XFS expecting "shared files" — it corrupts; that's EFS's job.
- ❌ Trying to encrypt a live unencrypted volume in place. Use the snapshot-copy-with-encryption path.
- ❌ Assuming an archived snapshot remains incremental or can be restored immediately. Archive
  stores a full snapshot and requires restoration to the standard tier first.
- ❌ Sharing an `aws/ebs`-encrypted snapshot cross-account. Use a customer managed KMS key, then
  copy and re-encrypt under a destination-owned key.
- ❌ Scheduling backups without testing quota headroom and timed restores. Snapshot creation is not
  proof that the recovery objective can be met.

---

**Next**: [02_efs_and_fsx.md — EFS & the FSx Family: Shared File Storage](02_efs_and_fsx.md)
