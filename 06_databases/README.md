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
| [01_database_fundamentals.md](01_database_fundamentals.md) | DB theory | SQL vs NoSQL, OLTP vs OLAP, ACID vs BASE, normalization, scaling, replication, failover, and the engine-selection decision table. |
| [02_rds.md](02_rds.md) | Managed relational | RDS engines, Multi-AZ (HA) vs Read Replicas (scaling), backups & PITR, encryption, IAM auth, RDS Proxy, RDS Custom. |
| [03_aurora.md](03_aurora.md) | Cloud-native relational | Aurora storage model, 15 read replicas, Serverless v2, Global Database, Backtrack, cloning, Aurora vs RDS. |
| [04_dynamodb.md](04_dynamodb.md) | Managed NoSQL | Keys & items, capacity modes (RCU/WCU), GSI vs LSI, Streams, DAX, Global Tables, TTL, consistency, hot partitions. |
| [05_elasticache_and_others.md](05_elasticache_and_others.md) | In-memory & purpose-built | ElastiCache Valkey/Redis OSS vs Memcached, caching patterns, plus Neptune, DocumentDB, Keyspaces, Timestream, QLDB, MemoryDB. |

---

## Reading Order

1. **Database Fundamentals** — the vocabulary and decision framework everything else builds on. Read this first even if you know SQL.
2. **RDS** — the default managed relational engine and the Multi-AZ vs Read Replica distinction the exam loves.
3. **Aurora** — AWS's cloud-native relational engine and how it differs from RDS.
4. **DynamoDB** — the flagship NoSQL service and when it beats relational.
5. **ElastiCache & Others** — in-memory caching and the purpose-built database family.

---

## Prerequisites

- [Foundations](../01_foundations/README.md) — Regions and Availability Zones (Multi-AZ depends on this)
- [Networking](../03_networking/README.md) — VPCs, subnets, and security groups (databases live in subnets)
- [IAM](../02_iam/README.md) — roles and policies (IAM database authentication)

---

**Next**: [01_database_fundamentals.md — Database Fundamentals](01_database_fundamentals.md)
