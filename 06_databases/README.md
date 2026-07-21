# Databases

> Choosing the right data store is one of the most heavily tested skills on the SAA-C03 exam. This section starts with the *theory* — relational vs non-relational, OLTP vs OLAP, ACID vs BASE, scaling and replication — then maps each AWS database service onto the problem it solves.

[![AWS](https://img.shields.io/badge/AWS-Databases-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/products/databases/)
[![RDS](https://img.shields.io/badge/RDS-Aurora-527FFF.svg?logo=amazonrds&logoColor=white)](https://aws.amazon.com/rds/)
[![DynamoDB](https://img.shields.io/badge/DynamoDB-NoSQL-4053D6.svg?logo=amazondynamodb&logoColor=white)](https://aws.amazon.com/dynamodb/)
[![ElastiCache](https://img.shields.io/badge/ElastiCache-Valkey%20%7C%20Redis%20OSS%20%7C%20Memcached-C925D1.svg?logo=amazonelasticache&logoColor=white)](https://aws.amazon.com/elasticache/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_database_fundamentals.md](01_database_fundamentals.md) | DB theory & migration | Models and scaling, then a migration framework covering compatibility, licensing, downtime, RTO/RPO, data gravity, ownership, refactor/exit cost, DMS, validation, cutover, and rollback. |
| [02_rds.md](02_rds.md) | Managed relational | Multi-AZ instance vs cluster, cross-Region replica vs replicated backups, Blue/Green, RDS Custom, Proxy failover, Database Insights, and encrypted cross-account snapshots. |
| [03_aurora.md](03_aurora.md) | Cloud-native relational | Shared storage, Serverless v2 capacity planning, Global Database switchover/failover and write forwarding, Backtrack vs PITR, and clone-based deployment tests. |
| [04_dynamodb.md](04_dynamodb.md) | Managed NoSQL | Single-table evolution, capacity and hot-key remediation, MREC/MRSC Global Tables and Region evacuation, transactions, import/export, PITR, and resilient Streams consumers. |
| [05_elasticache_and_others.md](05_elasticache_and_others.md) | In-memory & purpose-built | Cache patterns plus relational, key-value, graph, document, time-series, search, and durable in-memory modernization with consistency, durability, operations, and exit trade-offs. |

---

## Reading Order

1. **Database Fundamentals** — the vocabulary and decision framework everything else builds on. Read this first even if you know SQL.
2. **RDS** — the default managed relational engine and the Multi-AZ vs Read Replica distinction the exam loves.
3. **Aurora** — AWS's cloud-native relational engine and how it differs from RDS.
4. **DynamoDB** — the flagship NoSQL service and when it beats relational.
5. **ElastiCache & Others** — in-memory caching and the purpose-built database family.

---

## SAP-C02 Professional Study Path

For SAP-C02, service recognition is only the first step. Work each scenario from business
continuity and change constraints to an operational runbook:

| Scenario family | Read | Questions to answer before choosing |
|-----------------|------|-------------------------------------|
| **Migration and cutover** | [Database Fundamentals](01_database_fundamentals.md) → [RDS](02_rds.md) | Is the move homogeneous or heterogeneous? Which features/licenses block it? What seed + CDC + validation path meets downtime, and when does rollback stop being safe? |
| **Zero- or low-downtime change** | [RDS](02_rds.md) | Does Blue/Green support the engine/topology? What happens to connections and endpoints? How are green validation and post-switchover writes handled? |
| **Global relational resilience** | [Aurora](03_aurora.md) | Planned RPO 0 switchover or unplanned lag-based failover? How are old writes fenced, clients reconnected, DNS refreshed, and the old Region rejoined? |
| **Global serverless resilience** | [DynamoDB](04_dynamodb.md) | MREC or MRSC? What conflict/RPO behavior is acceptable, does quorum survive, and what application routing evacuates the Region? |
| **Performance diagnosis** | [RDS](02_rds.md) → [Aurora](03_aurora.md) → [DynamoDB](04_dynamodb.md) | Is the signal DB waits/SQL, connections, storage, ACU bounds, partition skew, or index capacity? Which metric proves the remediation? |
| **Purpose-built modernization** | [ElastiCache & Purpose-Built Databases](05_elasticache_and_others.md) | What remains the source of truth? Which consistency boundary, CDC/replay path, operational burden, and exit cost come with the new store? |
| **Cost optimization** | All five notes | Compare steady compute/capacity, replicas, I/O, storage/backups, transfer, idle DR, licenses, support, migration/refactor effort, and recovery-test cost—not just the headline request price. |

### End-to-end practice sequence

1. **State the invariants** — transactions, read-after-write behavior, peak and normal traffic,
   RTO/RPO, retention, residency, and compliance.
2. **Choose a source of truth** — then identify caches, indexes, graph views, or analytics copies
   as projections with measurable lag and a rebuild path.
3. **Select a topology** — instance/cluster, AZ/Region placement, readers, endpoint strategy,
   encryption keys, networking, quotas, and capacity floor/ceiling.
4. **Design the change** — assessment, seed, CDC or synchronization, validation, write freeze,
   endpoint switch, smoke tests, observation window, and a rollback data boundary.
5. **Exercise failure** — instance/AZ failover, Region evacuation, stale DNS/connections, poison
   stream records, PITR/restore, and loss of a cache or projection.
6. **Revisit cost after evidence** — use observed load, lag, waits, recovery duration, and
   operational effort to tune the design. A cheap cold backup is expensive if its restore misses
   RTO; a warm replica is wasteful if the business accepts a long recovery.

> **Professional-exam habit**: reject answers that solve only one word in the requirement. A
> replica may improve RTO but not RPO, a CDC task may shorten migration downtime but not provide
> steady-state HA, and a serverless label does not remove data modeling, quotas, or recovery work.

---

## Prerequisites

- [Foundations](../01_foundations/README.md) — Regions and Availability Zones (Multi-AZ depends on this)
- [Networking](../03_networking/README.md) — VPCs, subnets, and security groups (databases live in subnets)
- [IAM](../02_iam/README.md) — roles and policies (IAM database authentication)

---

**Next**: [01_database_fundamentals.md — Database Fundamentals](01_database_fundamentals.md)
