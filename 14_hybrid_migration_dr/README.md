# Hybrid, Migration & Disaster Recovery

> How on-prem data centers connect to AWS, how workloads move *into* AWS, and how you keep them surviving an outage once they're there.

[![AWS](https://img.shields.io/badge/AWS-Hybrid%20%7C%20Migration%20%7C%20DR-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/)

---

This section ties together three exam-heavy themes that all assume you already have a working VPC and some storage/database knowledge. It is written for a **networking-light engineer**: every file starts from the *why* (why connect on-prem at all, why back up vs replicate, what RTO/RPO actually mean) before naming a single AWS service.

The four files chain in order — hybrid connectivity gets your networks talking, migration services move the workloads, AWS Backup protects them, and DR strategies decide how fast you recover when a Region or AZ fails.

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_hybrid_networking.md](01_hybrid_networking.md) | Connecting on-prem to AWS | Why hybrid; Site-to-Site VPN vs Direct Connect (comparison table); VPN-over-DX for encryption; Customer Gateway, Virtual Private Gateway, Transit Gateway; when VPN vs DX vs both. |
| [02_migration_services.md](02_migration_services.md) | Moving workloads in | The 6 R's; Migration Hub; Application Migration Service (MGN); DMS + SCT; DataSync; Snow Family; Transfer Family; "which tool" decision table. |
| [03_backup.md](03_backup.md) | Protecting data | Backup vs replication vs snapshot; RPO; AWS Backup plans, vaults, Vault Lock (WORM); cross-Region & cross-account; tag-based selection. |
| [04_disaster_recovery.md](04_disaster_recovery.md) | Surviving an outage | RTO vs RPO timeline; the four DR strategies (Backup & Restore → Pilot Light → Warm Standby → Multi-Site) with topology diagrams and a full comparison table. |

---

## Reading Order

1. **Hybrid Networking** — you can't migrate or replicate to AWS until the networks are connected.
2. **Migration Services** — move the servers, databases, and files in.
3. **AWS Backup** — protect what you migrated; backup is the foundation of the cheapest DR strategy.
4. **Disaster Recovery** — choose how fast you need to recover and what it costs.

---

## Prerequisites

- [Networking](../03_networking/README.md) — VPCs, route tables, and especially [Transit Gateway](../03_networking/06_transit_gateway_and_advanced.md)
- [Storage](../05_storage/README.md) — EBS/EFS snapshots, S3, [Storage Gateway & Transfer](../05_storage/06_storage_gateway_and_transfer.md)
- [Databases](../06_databases/README.md) — RDS/Aurora, snapshots, read replicas, cross-Region replication
