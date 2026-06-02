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

### FSx for Lustre + S3

Lustre's signature exam feature: it **integrates with S3**. You can link an S3 bucket, and Lustre
presents the objects as files; results written back can be exported to S3. Ideal for ML training and
big-data jobs where the data lake lives in S3 but the compute needs a screaming-fast POSIX filesystem.

```
   S3 bucket (data lake)  ◄──link──►  FSx for Lustre  ◄── EC2/GPU cluster
   objects                            files (POSIX)       HPC / ML training
```

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

| Need | Choose |
|------|--------|
| Linux, NFS, multi-AZ, elastic, low ops | **EFS** |
| Windows file share with Active Directory / SMB | **FSx for Windows File Server** |
| HPC / ML / analytics, extreme throughput, S3-linked | **FSx for Lustre** |
| Multi-protocol (NFS + SMB), NetApp features, on-prem NetApp migration | **FSx for NetApp ONTAP** |
| ZFS workloads / NFS with snapshots & cloning, high IOPS | **FSx for OpenZFS** |

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
- **FSx for Lustre = HPC/ML, links to S3.** The "S3-integrated high-performance filesystem" clue.
- **FSx for NetApp ONTAP** = multi-protocol (NFS/SMB/iSCSI), NetApp migration. **FSx for OpenZFS** =
  ZFS/NFS workloads.
- EFS is the only **shared + multi-AZ + elastic** option among EFS/EBS/instance store.

---

## Common Mistakes

- ❌ Recommending EFS for Windows clients. EFS is NFS/Linux only — Windows needs FSx for Windows (SMB).
- ❌ Forgetting that EFS needs a **mount target in each AZ** that instances use.
- ❌ Using EBS Multi-Attach to "share files" — that needs a cluster filesystem; the real answer is EFS.
- ❌ Picking FSx for Lustre for general file sharing. Lustre is for HPC/ML throughput, not a CMS share.
- ❌ Assuming EFS requires capacity planning — it's elastic; you don't provision size.

---

**Next**: [03_s3_fundamentals.md — Amazon S3: The Object Storage Model](03_s3_fundamentals.md)
