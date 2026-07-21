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
| [01_hybrid_networking.md](01_hybrid_networking.md) | Connecting on-prem to AWS | VPN and Direct Connect; VIFs and gateways; dedicated/hosted connections; SiteLink and encryption; resilient BGP designs; an evidence-driven troubleshooting runbook. |
| [02_migration_services.md](02_migration_services.md) | Moving workloads in | The 7 R's; AWS Transform assessment and wave planning; Migration Hub/Discovery Service for existing projects; MGN, DMS Schema Conversion/SCT and cutover runbooks. |
| [03_backup.md](03_backup.md) | Protecting data | Backup vs replication vs snapshot; RPO; AWS Backup plans, vaults, Vault Lock (WORM); cross-Region & cross-account; tag-based selection. |
| [04_disaster_recovery.md](04_disaster_recovery.md) | Surviving an outage | RTO vs RPO timeline; the four DR strategies (Backup & Restore → Pilot Light → Warm Standby → Multi-Site) with topology diagrams and a full comparison table. |

---

## Reading Order

1. **Hybrid Networking** — you can't migrate or replicate to AWS until the networks are connected.
2. **Migration Services** — move the servers, databases, and files in.
3. **AWS Backup** — protect what you migrated; backup is the foundation of the cheapest DR strategy.
4. **Disaster Recovery** — choose how fast you need to recover and what it costs.

---

## SAP-C02 Professional Path

This section is the primary path for **Domain 4: Accelerate Workload Migration and
Modernization**. Read it after the networking, storage and database prerequisites, then use this
checklist to turn service recognition into architecture decisions:

| Domain 4 task | Where to practice it here | Evidence you should be able to produce |
|---------------|---------------------------|----------------------------------------|
| Assess existing workloads and migration readiness | [Migration Services §9](02_migration_services.md#9-portfolio-assessment-and-migration-waves) | Dependency inventory, TCO assumptions, target 7-R decision, risks and entry criteria |
| Select migration approaches and services | [Migration Services §8](02_migration_services.md#8-which-migration-tool--decision-table) and [§10](02_migration_services.md#10-end-to-end-cutover-runbooks) | Tool/target decision, wave plan, security/network prerequisites, tested rollback |
| Determine a modernization strategy | The 7-R decision plus the purpose-built target choices in [Databases](../06_databases/README.md), [Containers](../09_containers/README.md) and [Serverless](../10_serverless/README.md) | Why replatform/refactor creates enough operational or business value to justify change |
| Design migration governance | [Migration Services §9](02_migration_services.md#9-portfolio-assessment-and-migration-waves) and [AWS Backup §10](03_backup.md#10-organization-wide-backup-governance) | Landing-zone/account ownership, wave controls, security exceptions, audit and recovery roles |
| Validate and operate the migrated workload | [Migration Services §10](02_migration_services.md#10-end-to-end-cutover-runbooks) and [DR §10](04_disaster_recovery.md#10-proving-recovery-readiness) | Rehearsal results, performance/security acceptance, measured RTO/RPO, cost and operations handoff |
| Optimize after migration | Rehost stabilization steps in [Migration Services §10](02_migration_services.md#10-end-to-end-cutover-runbooks) plus [Cost Optimization](../15_cost_well_architected/01_cost_optimization.md) | Rightsizing and commitment decisions based on production measurements, not pre-migration guesses |

For SAP study, work one application scenario from assessment through failback: document its
hybrid network, seven-R decision, dependency wave, target data/compute design, cutover controls,
backup isolation, regional recovery sequence and post-migration optimization. If any link is
missing, the design is not yet production-ready.

---

## Prerequisites

- [Networking](../03_networking/README.md) — VPCs, route tables, and especially [Transit Gateway](../03_networking/06_transit_gateway_and_advanced.md)
- [Storage](../05_storage/README.md) — EBS/EFS snapshots, S3, [Storage Gateway & Transfer](../05_storage/06_storage_gateway_and_transfer.md)
- [Databases](../06_databases/README.md) — RDS/Aurora, snapshots, read replicas, cross-Region replication
