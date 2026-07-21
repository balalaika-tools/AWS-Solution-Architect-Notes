# Storage

> Block, file, and object storage on AWS — the disks behind EC2 (EBS), shared file systems (EFS, FSx), the object store that underpins half the platform (S3), and the gateways and appliances that move data in and out. The exam tests *which storage type fits which workload* and *which S3 storage class fits which access pattern* more than any other storage topic.

[![AWS](https://img.shields.io/badge/AWS-Storage-FF9900.svg?logo=amazonaws&logoColor=white)](https://docs.aws.amazon.com/whitepapers/latest/aws-overview/storage-services.html)
[![S3](https://img.shields.io/badge/Amazon-S3-569A31.svg?logo=amazons3&logoColor=white)](https://docs.aws.amazon.com/s3/)

---

Storage questions are everywhere on SAA-C03. The single most important skill is mapping a workload to the right primitive: **block** (one EC2 instance, low-latency disk → EBS), **file** (many instances sharing a POSIX/SMB tree → EFS/FSx), or **object** (anything at internet scale, accessed by HTTP API → S3). Get that decision right and the rest is detail. This section builds that decision tree from first principles, then layers on the cost/durability/retrieval trade-offs the exam loves to probe.

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_ebs.md](01_ebs.md) | Block storage | Block vs file vs object primer, then EBS: AZ-scoped network volumes, gp3/io2 Block Express/st1/sc1 types, IOPS/throughput, snapshots, encryption, resizing, Multi-Attach, EBS vs instance store. |
| [02_efs_and_fsx.md](02_efs_and_fsx.md) | File storage | EFS (managed NFS, multi-AZ, elastic, throughput modes, lifecycle) vs EBS vs instance store; the FSx family (Windows, Lustre, NetApp ONTAP, OpenZFS); when to choose EFS vs FSx. |
| [03_s3_fundamentals.md](03_s3_fundamentals.md) | Object storage | The object model, bucket namespaces/keys/prefixes, 11 9s durability, strong consistency, 50 TB max object + multipart, flat object namespace, block public access vs bucket policy vs ACL vs IAM. |
| [04_s3_storage_classes_and_management.md](04_s3_storage_classes_and_management.md) | S3 in depth | Storage classes comparison, lifecycle, versioning + MFA Delete, replication, encryption (SSE-S3/KMS/C/DSSE), presigned URLs, events, Transfer Acceleration, Object Lock/Vault Lock, static websites. |
| [05_s3_advanced_features.md](05_s3_advanced_features.md) | S3 advanced | Performance (prefixes, multipart, byte-range), Storage Class Analysis, Storage Lens, Batch Operations, CORS, server access logs, Access Points, plus legacy/current-availability notes for S3 Select and Object Lambda. |
| [06_storage_gateway_and_transfer.md](06_storage_gateway_and_transfer.md) | Hybrid & transfer | Storage Gateway (File/Volume/Tape, plus FSx gateway legacy recognition), DataSync, Transfer Family, Data Transfer Terminal, Snow status, and migration/cutover planning. |

---

## Reading Order

1. **EBS** — start here for the block-vs-file-vs-object mental model, then the disks attached to EC2.
2. **EFS & FSx** — shared file systems and when each beats a block volume.
3. **S3 Fundamentals** — the object model and access control basics.
4. **S3 Storage Classes & Management** — the cost/retrieval trade-offs and the data-management features.
5. **S3 Advanced Features** — performance, the analytics tools, bulk operations, and the access layer.
6. **Storage Gateway & Transfer** — getting data between on-premises and AWS.

---

## SAP-C02 Storage Path

The SAA path asks which storage primitive or class fits one workload. SAP-C02 adds organizational
boundaries, recovery objectives, migration sequencing, and proof that the design performs and costs
what the business expects.

Use this path for professional-level scenarios:

1. [EBS backup and migration](01_ebs.md#4-snapshots) — design incremental retention, restore
   performance, archive, accidental-deletion recovery, encrypted cross-account copies, and DLM vs
   AWS Backup with quota and restore-test planning.
2. [EFS and FSx](02_efs_and_fsx.md#3-the-fsx-family) — select Windows, Lustre, ONTAP, or OpenZFS
   from protocol and application semantics, then validate Multi-AZ/backup behavior, AD and network
   dependencies, migration tooling, throughput, and hybrid latency.
3. [S3 organization governance](03_s3_fundamentals.md#6-organization-scale-ownership-and-governance)
   and [advanced analytics/access](05_s3_advanced_features.md) — establish account ownership, Bucket
   owner enforced, organization Block Public Access, per-consumer access, delegated Storage Lens,
   Inventory/Batch remediation, Access Grants, and private-connectivity cost boundaries.
4. [S3 cross-account and multi-Region protection](04_s3_storage_classes_and_management.md#4-replication-crr--srr)
   — join replication ownership and KMS permissions to RTC, MRAP failover, Object Lock, versioning,
   and a tested accidental-deletion recovery path.
5. [Migration planning and cutover](06_storage_gateway_and_transfer.md#6-migration-planning-size-is-only-the-first-input)
   — choose online, dedicated-network, or physical transfer from data shape, change rate, measured
   bandwidth, security, validation, cutover, and rollback rather than volume alone.

For every candidate architecture, finish with the same validation loop: benchmark representative
I/O and metadata patterns, model storage/request/retrieval/transfer/KMS costs, test quotas and
failure behavior, and time a restore or cutover against the stated RTO and RPO.

---

## Prerequisites

- [Compute](../04_compute/README.md) — EBS and instance store attach to EC2; understanding instances helps.
- [Foundations](../01_foundations/README.md) — Regions and Availability Zones (EBS is AZ-scoped; EFS spans AZs).
- [IAM](../02_iam/README.md) — bucket policies and IAM policies govern S3 access.

---

## Related sections

- [Databases](../06_databases/README.md) — RDS and DynamoDB are managed storage built on the same durability ideas.
- [Hybrid, Migration & DR](../14_hybrid_migration_dr/README.md) — DataSync, physical transfer, and Direct Connect overlap with migration.
- [Practical Examples](../18_practical_examples/README.md) — hands-on builds including S3 static sites and CloudFront.
