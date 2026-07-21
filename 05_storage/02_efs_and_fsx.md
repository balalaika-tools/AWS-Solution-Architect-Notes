# EFS & the FSx Family — Shared File Storage

> **Who this is for**: Engineers who understand EBS (block) and want the *file storage* picture for
> SAA-C03. Read [01_ebs.md](01_ebs.md) first — especially the block-vs-file-vs-object table. This
> file covers EFS (managed NFS), the four FSx file systems, and the decision of EFS vs FSx vs EBS.

---

## 1. The File-Storage Problem

EBS attaches to one instance. But many workloads — web farms serving shared content, CMS clusters,
home directories, lift-and-shift apps expecting a network share — need **many machines mounting the
same filesystem at once** with normal file semantics (paths, permissions, locking).

That's **file storage**: a managed share you mount over **NFS** (Linux) or **SMB** (Windows).

```
            ┌──────────────┐
   EC2-A ───┤              │
   EC2-B ───┤  shared FS   │   all instances see the SAME files,
   EC2-C ───┤ /mnt/shared  │   read/write concurrently
   Lambda ──┤              │
            └──────────────┘
```

---

## 2. Amazon EFS — Managed NFS

**EFS (Elastic File System)** is a fully managed, **elastic NFS** file system for **Linux**.

- **Protocol**: NFS v4. **Linux only** (Windows clients can't mount EFS — that's FSx for Windows).
- **Multi-AZ by default.** Data is stored redundantly across multiple AZs in a Region. You create a
  **mount target in each AZ**; each mount target is an ENI with a private IP and security group.
  Instances in that AZ connect through their AZ's mount target. This is the big contrast with EBS
  (single-AZ).
- **Elastic capacity.** Grows and shrinks automatically as you add/remove files — pay only for what
  you store (per GB-month). No provisioning a size.
- **Massively concurrent.** Thousands of instances/Lambdas can mount it simultaneously.
- Secured with **security groups** on mount targets and optional **IAM** + POSIX permissions; supports
  **encryption at rest (KMS) and in transit (TLS)**.

```
                 Region us-east-1
  ┌──────────────────────────────────────────────┐
  │  AZ-a            AZ-b            AZ-c        │
  │ ┌──────┐       ┌──────┐       ┌──────┐       │
  │ │mount │       │mount │       │mount │       │  one mount target per AZ
  │ │target│       │target│       │target│       │
  │ └──┬───┘       └──┬───┘       └──┬───┘       │
  │   EC2            EC2            EC2          │
  │        all see the same EFS file system      │
  └──────────────────────────────────────────────┘
```

### Throughput modes

| Mode | Behavior | Use when |
|------|----------|----------|
| **Elastic** (recommended default) | Scales throughput up/down automatically, pay per use | Spiky/unpredictable workloads |
| **Bursting** | Throughput scales with stored size, with burst credits | Steady workloads proportional to data size |
| **Provisioned** | Set throughput independent of size | High throughput on a small dataset |

### Performance modes

- **General Purpose** — lowest latency; default; right for most apps (web serving, CMS).
- **Max I/O** — higher aggregate throughput for highly parallel workloads, at slightly higher latency
  (legacy; Elastic throughput largely supersedes the need).

### Storage classes & lifecycle

| Class | Cost | Use |
|-------|------|-----|
| **EFS Standard** | Higher | Frequently accessed, multi-AZ |
| **EFS Standard-IA** | Lower storage, retrieval fee | Infrequently accessed, multi-AZ |
| **EFS One Zone** | Cheaper (single AZ) | Non-critical / reproducible data |
| **EFS One Zone-IA** | Cheapest | Rarely accessed, single-AZ |

**Lifecycle management** automatically moves files not accessed for *N* days (e.g., 30) into an
**IA** class to cut cost, and can move them back to Standard on access.

💡 Exam tell: "shared file system for **Linux** instances across multiple AZs, **elastic**, no
capacity planning" → **EFS**.

---

## 3. The FSx Family

**Amazon FSx** provides fully managed **third-party / specialized file systems** when EFS's NFS isn't
the right fit (e.g., you need Windows/SMB, HPC performance, or NetApp/ZFS features).

| FSx flavor | Protocol | What it is | Choose when |
|------------|----------|------------|-------------|
| **FSx for Windows File Server** | **SMB** | Fully managed native Windows file shares with Active Directory, ACLs, DFS | Windows apps need a file share; you need AD-integrated SMB |
| **FSx for Lustre** | Lustre (POSIX) | High-performance parallel filesystem for **HPC, ML, analytics**; hundreds of GB/s | Compute-intensive jobs; can **link to an S3 bucket** and present objects as files |
| **FSx for NetApp ONTAP** | NFS, SMB, **iSCSI** | Managed NetApp ONTAP — multi-protocol, snapshots, dedup, compression, cloning | Lift-and-shift from on-prem NetApp; need NFS *and* SMB on one system |
| **FSx for OpenZFS** | NFS | Managed OpenZFS — snapshots, cloning, low-latency | Migrating ZFS/NFS workloads; need ZFS features with high IOPS |

### FSx for Windows File Server

This is a managed Windows file server, not just "SMB storage." It preserves the Windows behaviors
that make a lift-and-shift work: NTFS ACLs, Active Directory identities, DFS namespaces, file
locking, and Windows-native administration.

- **Active Directory is a dependency.** Join the file system to AWS Managed Microsoft AD or a
  self-managed Microsoft AD. For self-managed AD, validate DNS, routing, firewall ports, domain
  controller reachability, the delegated service-account permissions, and the target OU before
  creation. Losing AD connectivity can prevent authentication even when the file system is healthy.
- **Multi-AZ is the production default.** It maintains file servers across AZs and fails over while
  clients continue using the same DNS name. Use Single-AZ for development, lower-cost workloads, or
  applications that provide their own replication and can tolerate the longer interruption/data risk.
- **Capacity and throughput are separate decisions.** Choose SSD for latency-sensitive and
  IOPS-heavy workloads; HDD for large, sequential general-purpose shares. Provision throughput for
  the aggregate client workload rather than sizing storage and assuming performance will follow.
- **Migration**: use DataSync for a managed initial copy plus incremental passes; it can preserve
  NTFS ACLs and audit metadata. `robocopy` is useful when Windows-native control is required, but you
  own retries, reporting, and cutover coordination.
- **Protection**: automatic/user backups are incremental and file-system-consistent using VSS;
  AWS Backup adds organization-level policies and restore testing. A backup restores as a new file
  system, so rehearse the DNS/share cutover and AD integration.

For hybrid users, direct SMB over Direct Connect or VPN avoids an extra appliance but exposes the
application to WAN latency. Existing **FSx File Gateway** customers can add an on-premises cache for
frequently accessed files, but the gateway has not been available to new customers since October
2024. New designs should validate direct SMB over resilient connectivity or a supported partner
cache rather than depend on provisioning a new FSx File Gateway.

### FSx for Lustre + S3

Lustre's signature exam feature: it **integrates with S3**. You can link an S3 bucket, and Lustre
presents the objects as files; results written back can be exported to S3. Ideal for ML training and
big-data jobs where the data lake lives in S3 but the compute needs a screaming-fast POSIX filesystem.

```
   S3 bucket (data lake)  ◄──link──►  FSx for Lustre  ◄── EC2/GPU cluster
   objects                            files (POSIX)       HPC / ML training
```

Choose the deployment and data-protection model together:

| Choice | Use when | Data-protection implication |
|--------|----------|-----------------------------|
| **Scratch** | Temporary processing, burst capacity, reproducible input in S3 | Optimized for short-lived jobs; no FSx backups. Keep authoritative input and exported results in S3. |
| **Persistent** | Long-running workloads or data that must survive file-server replacement | Data is replicated within the file system. Backups are available only when the file system is **not** linked to an S3 data repository. |
| **S3-linked data repository** | S3 is the durable system of record and compute needs a high-performance POSIX view | Import/lazy-load input and export results. Do not assume every file in Lustre is already durable in S3; define and monitor the export workflow. |

Choose the storage class as well as the deployment type. On SSD/HDD file systems, capacity and the
selected throughput per unit of storage together determine aggregate performance. Persistent SSD
deployments can adjust their throughput tier; storage capacity can grow but not shrink.
**Intelligent-Tiering** instead uses elastic regional storage whose capacity grows and shrinks with
the dataset; provision total file-system throughput independently and optionally add SSD read cache
for hot data. This is useful for large, changing datasets, but uncached reads and request charges
must fit the workload. Benchmark metadata-heavy workloads separately from large sequential I/O, and
verify that clients and the network can drive the provisioned file-system throughput. Use DataSync
to migrate from another Lustre/NFS environment or between Regions/accounts.

### FSx for NetApp ONTAP

ONTAP is the choice when the **NetApp operating model or multi-protocol access** is part of the
requirement—not merely because the workload needs NFS.

- Storage virtual machines and volumes can serve NFS, SMB, and iSCSI while preserving ONTAP
  features such as snapshots, thin clones, deduplication, compression, and SnapLock.
- Choose Multi-AZ for production availability and Single-AZ when cost or an application-level
  resilience design justifies the narrower failure boundary.
- Size three performance dimensions: SSD capacity for the active working set, throughput capacity
  for the file servers, and SSD IOPS. A capacity-pool tier can hold cold data elastically, but
  reading cold blocks adds tier-access cost and latency; leave SSD headroom for writes and cache.
- Use **SnapMirror** for an efficient migration from NetApp or for ONTAP-native replication that
  preserves the storage workflow. Use **DataSync** for generic NFS/SMB sources. Native automatic or
  manual backups and AWS Backup provide point-in-time recovery; neither replaces application-level
  consistency testing.

Hybrid clients can mount ONTAP over Direct Connect or VPN, and NetApp replication can keep moving
changed blocks after the initial seed. The benefit is compatibility and a lower-change migration;
the trade-off is ONTAP expertise, WAN latency, tiering behavior, and more cost dimensions than EFS.

### FSx for OpenZFS

OpenZFS provides managed NFS with ZFS snapshots, clones, compression, and low-latency storage. It is
a strong fit for Linux applications already built around ZFS datasets or NFS and for development or
analytics environments that benefit from space-efficient clones.

- Deployment options include Multi-AZ HA, Single-AZ HA, and lower-cost Single-AZ non-HA choices;
  select from the workload's AZ-failure and recovery requirements.
- Choose the storage class from the access pattern. **Intelligent-Tiering** grows and shrinks
  automatically and separates throughput from stored capacity; an optional SSD read cache reduces
  latency for the hot working set, while uncached access has higher latency and request cost.
  **Provisioned SSD** keeps the full dataset on low-latency SSD and requires capacity planning.
  With provisioned SSD, size storage capacity, throughput capacity, and IOPS from measured demand.
  Clones save capacity initially, but changed blocks and snapshots still consume storage.
- Use DataSync for migration from on-premises ZFS or other NFS servers. Automatic and user-initiated
  backups are incremental, restore to a new file system, and can be managed through AWS Backup.
- Direct hybrid NFS access over Direct Connect/VPN is possible, but it does not erase WAN latency.
  If the application is sensitive to metadata round trips, migrate the compute with the data or use
  a replication/cache design instead of treating a remote mount as local storage.

⚠️ **FSx for Windows = SMB.** If a question says "Windows servers need a shared drive with AD
authentication," the answer is FSx for Windows File Server, **not EFS** (EFS is NFS/Linux only).

---

## 4. EFS vs EBS vs Instance Store

| | **EFS** | **EBS** | **Instance store** |
|---|---|---|---|
| Type | File (NFS) | Block | Block (local) |
| Attaches to | Many instances at once | One instance (Multi-Attach: io1/io2) | One instance |
| Scope | **Multi-AZ** (Region) | **Single AZ** | The host |
| Capacity | **Elastic** (auto-grow) | Provisioned (grow only) | Fixed |
| OS | Linux | Any | Any |
| Persistence | Durable, multi-AZ | Durable in its AZ | **Ephemeral** |
| Use | Shared files across many instances | Boot/DB disk for one instance | Scratch/cache |

> **Key insight**: EFS is the only one of the three that is **shared, multi-AZ, and elastic**. If
> the requirement is "multiple instances share files and it must survive an AZ failure," EFS is the
> answer — EBS can't span AZs and instance store isn't durable.

---

## 5. Choosing EFS vs FSx

Choose the filesystem by protocol and application semantics first, then validate availability,
migration, performance, backup, and hybrid access.

| Requirement | Default choice | Validate before committing |
|-------------|----------------|----------------------------|
| Linux NFS share, regional multi-AZ access, elastic capacity, minimal administration | **EFS** | General-purpose latency, throughput mode, lifecycle retrieval cost, POSIX/IAM permissions |
| Windows SMB share with NTFS ACLs and AD identities | **FSx for Windows File Server** | Multi-AZ vs Single-AZ, AD/DNS reachability, SSD vs HDD, throughput capacity, ACL-preserving migration |
| HPC/ML/analytics parallel I/O or an S3-backed compute filesystem | **FSx for Lustre** | Scratch vs persistent, S3 export responsibility, backup eligibility, metadata and aggregate throughput |
| NFS + SMB/iSCSI, NetApp features, or low-change migration from ONTAP | **FSx for NetApp ONTAP** | Multi-AZ, SSD working set, capacity-pool behavior, SnapMirror/DataSync path, operations skill |
| ZFS/NFS workload needing snapshots and fast clones | **FSx for OpenZFS** | HA deployment type, throughput/IOPS, DataSync migration, clone/snapshot growth |

✅ Default Linux shared file system → **EFS**.
✅ Anything Windows/SMB → **FSx for Windows**.
✅ "High-performance computing," "machine learning training," "linked to S3" → **FSx for Lustre**.

---

## Key Exam Points

- **EFS = managed NFS, Linux only, multi-AZ, elastic.** Mount target **per AZ**; secured by security
  groups on mount-target ENIs; encryption via KMS (rest) and TLS (transit).
- EFS **throughput modes**: Elastic (default), Bursting, Provisioned. **Lifecycle** moves cold files
  to IA to save cost; **One Zone** classes trade AZ-resilience for lower price.
- **FSx for Windows = SMB + Active Directory.** Don't answer EFS for a Windows file share.
- **FSx for Windows Multi-AZ** is the production default; self-managed AD makes DNS, network
  reachability, and delegated service-account permissions part of storage availability.
- **FSx for Lustre = HPC/ML, links to S3.** Scratch is temporary; persistent is for longer-lived
  data, but FSx backups are unavailable when the file system is linked to S3.
- **FSx for NetApp ONTAP** = multi-protocol (NFS/SMB/iSCSI), NetApp migration. **FSx for OpenZFS** =
  ZFS/NFS workloads. Size throughput/IOPS separately from capacity and choose an HA deployment.
- EFS is the only **shared + multi-AZ + elastic** option among EFS/EBS/instance store.

---

## Common Mistakes

- ❌ Recommending EFS for Windows clients. EFS is NFS/Linux only — Windows needs FSx for Windows (SMB).
- ❌ Forgetting that EFS needs a **mount target in each AZ** that instances use.
- ❌ Using EBS Multi-Attach to "share files" — that needs a cluster filesystem; the real answer is EFS.
- ❌ Picking FSx for Lustre for general file sharing. Lustre is for HPC/ML throughput, not a CMS share.
- ❌ Assuming EFS requires capacity planning — it's elastic; you don't provision size.
- ❌ Treating FSx for Windows as available when AD is unreachable. Authentication, DNS, and the
  self-managed AD service account are runtime dependencies.
- ❌ Using Lustre Scratch as the only copy of valuable data, or assuming a linked S3 bucket contains
  results that were never exported.
- ❌ Sizing an FSx file system by terabytes alone. Throughput capacity, IOPS, client/network limits,
  deployment type, and backup/cutover behavior also determine whether the design works.

---

**Next**: [03_s3_fundamentals.md — Amazon S3: The Object Storage Model](03_s3_fundamentals.md)
