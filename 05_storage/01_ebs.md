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

---

**Next**: [02_efs_and_fsx.md — EFS & the FSx Family: Shared File Storage](02_efs_and_fsx.md)
