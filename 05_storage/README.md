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
| [03_s3_fundamentals.md](03_s3_fundamentals.md) | Object storage | The object model, buckets/keys/prefixes, 11 9s durability, strong consistency, 5 TB max object + multipart, flat namespace, block public access vs bucket policy vs ACL vs IAM. |
| [04_s3_storage_classes_and_management.md](04_s3_storage_classes_and_management.md) | S3 in depth | Storage classes comparison, lifecycle, versioning + MFA Delete, replication, encryption (SSE-S3/KMS/C/DSSE), presigned URLs, events, Transfer Acceleration, Object Lock/Vault Lock, static websites. |
| [05_s3_advanced_features.md](05_s3_advanced_features.md) | S3 advanced | Performance (prefixes, multipart, byte-range), Storage Class Analysis, Storage Lens, Batch Operations, CORS, server access logs, Access Points, plus legacy/current-availability notes for S3 Select and Object Lambda. |
| [06_storage_gateway_and_transfer.md](06_storage_gateway_and_transfer.md) | Hybrid & transfer | Storage Gateway (File/Volume/Tape), DataSync, Transfer Family (SFTP/FTPS), Data Transfer Terminal, Snow legacy recognition, and choosing online vs physical transfer vs Direct Connect. |

---

## Reading Order

1. **EBS** — start here for the block-vs-file-vs-object mental model, then the disks attached to EC2.
2. **EFS & FSx** — shared file systems and when each beats a block volume.
3. **S3 Fundamentals** — the object model and access control basics.
4. **S3 Storage Classes & Management** — the cost/retrieval trade-offs and the data-management features.
5. **S3 Advanced Features** — performance, the analytics tools, bulk operations, and the access layer.
6. **Storage Gateway & Transfer** — getting data between on-premises and AWS.

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
