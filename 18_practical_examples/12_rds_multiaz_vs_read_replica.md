# RDS Multi-AZ vs Read Replicas, Side by Side

> **Who this is for**: Engineers prepping for SAA-C03 who keep confusing
> **Multi-AZ** (high availability) with **Read Replicas** (read scaling). They
> solve different problems and the exam tests whether you know which is which.
> Assumes you've read [RDS](../06_databases/02_rds.md) and
> [RDS in private subnets](11_rds_private_access.md).

> **One sentence**: Multi-AZ is for **availability** (synchronous standby, one
> endpoint, automatic failover, *not readable*); Read Replicas are for **read
> scaling** (asynchronous copies, separate endpoints, can be cross-region, *not
> automatic failover*).

---

## 1. Multi-AZ — Synchronous Standby for HA

```
            same endpoint:  mydb.xxxx.us-east-1.rds.amazonaws.com
                                    │
        ┌───────────────────────────┴───────────────────────────┐
        ▼                                                         ▼
 ┌──────────────┐        synchronous replication         ┌──────────────┐
 │  PRIMARY      │ ─────────────────────────────────────▶ │  STANDBY     │
 │  AZ-a         │                                         │  AZ-b        │
 │  read + write │        (commit waits for standby)       │  NOT readable│
 └──────────────┘                                         └──────────────┘

 On failure:  DNS endpoint repoints to the standby (~60–120s).  Same name.
```

- **Synchronous** replication — a write isn't acknowledged until the standby has
  it. Zero data loss on failover (RPO ≈ 0).
- **One endpoint.** The standby has no separate connection string and **cannot
  serve reads** (in the classic Multi-AZ *instance* deployment).
- **Automatic failover** on AZ outage, primary crash, storage failure, or during
  patching/instance-type changes. The CNAME is repointed; the app reconnects.

```bash
# Convert an existing single-AZ instance to Multi-AZ (no rebuild)
aws rds modify-db-instance \
  --db-instance-identifier app-prod-db \
  --multi-az \
  --apply-immediately
```

⚠️ **Multi-AZ is not a read-scaling feature.** The standby sits idle for reads.
If a question says "the read load is too high," Multi-AZ is the wrong fix.

> Note: the newer **Multi-AZ DB cluster** (3 instances, 2 *readable* standbys)
> exists, but the classic, most-tested **Multi-AZ instance** has a non-readable
> standby. Default to "standby not readable" unless the question says "cluster."

---

## 2. Read Replicas — Asynchronous Copies for Read Scaling

```
 writes ──▶ ┌──────────────┐
            │  PRIMARY      │ ── async ──▶ ┌──────────────┐  reads ◀──
            │  (writer)     │              │ Read Replica1│
            │               │ ── async ──▶ ├──────────────┤  reads ◀──
            └──────────────┘              │ Read Replica2│
              endpoint A                   └──────────────┘  endpoint B, C ...
                                           (cross-region replica possible)
```

- **Asynchronous** replication — replicas can lag (replica lag, seconds). Eventual
  consistency for readers.
- **Separate endpoint per replica.** The app must be pointed at replica endpoints
  for read queries (read/write splitting in the app or via a proxy).
- **Up to 15** read replicas (RDS engines), **can be cross-region** (great for
  serving reads near distant users or as a DR seed).
- **No automatic failover.** If the primary dies, a replica does **not** auto-promote
  — you must **manually promote** one (it then becomes a standalone writable DB).

```bash
# Create a same-region read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier app-prod-db-rr1 \
  --source-db-instance-identifier app-prod-db

# Create a CROSS-REGION read replica (serve reads in eu-west-1)
aws rds create-db-instance-read-replica \
  --db-instance-identifier app-prod-db-eu \
  --source-db-instance-identifier arn:aws:rds:us-east-1:111122223333:db:app-prod-db \
  --region eu-west-1

# Promote a replica to a standalone primary (manual DR / break replication)
aws rds promote-read-replica --db-instance-identifier app-prod-db-rr1
```

---

## 3. Combine Them: Multi-AZ + Read Replicas

These are orthogonal — use both for HA *and* read scaling:

```
                       ┌─────────────┐  sync   ┌─────────────┐
   writes ────────────▶│  PRIMARY    │◀───────▶│  STANDBY    │  (HA, not readable)
                       │  AZ-a       │         │  AZ-b       │
                       └──────┬──────┘         └─────────────┘
                              │ async
                  ┌───────────┼───────────┐
                  ▼                       ▼
           ┌─────────────┐         ┌─────────────┐
           │ Read Replica│  reads  │ Read Replica│  reads
           └─────────────┘         └─────────────┘
```

💡 You can also make a **read replica itself Multi-AZ** so that promoting it
later yields an HA primary. And replicas continue replicating from the standby's
data after a primary failover transparently.

---

## 4. Decision Table

| Need / question phrase                          | Multi-AZ | Read Replica |
|-------------------------------------------------|:--------:|:------------:|
| Survive an **AZ failure** with auto failover    |   ✅     |     ❌       |
| **Zero data loss** on failover (RPO ≈ 0)        |   ✅     | ❌ (async lag)|
| **Scale read traffic** / offload reporting      |   ❌     |     ✅       |
| **Cross-region** copy / serve distant readers   |   ❌¹    |     ✅       |
| **Separate endpoint** to target                 |   ❌     |     ✅       |
| **Automatic** failover                          |   ✅     | ❌ (manual promote)|
| **Reduce maintenance downtime** (patch standby) |   ✅     |     ❌       |
| Replication type                                | synchronous | asynchronous |

¹ Classic Multi-AZ is within one region; cross-region HA is a separate design.

---

## 5. Troubleshooting / Common Exam Traps

❌ **"Reads are slow, so enable Multi-AZ."** — Wrong. The standby is not
readable. Add **read replicas** and route reads to them.

❌ **"Primary failed, the read replica took over automatically."** — Wrong.
Read replicas need **manual promotion**; they do not auto-failover. Multi-AZ is
what auto-fails-over.

⚠️ **Stale data from a replica.** Asynchronous replication means a read replica
can lag behind the primary. If a user writes then immediately reads from a
replica, they may not see their write. Send read-your-writes traffic to the
primary, or use the primary for that path.

⚠️ **High replica lag.** Caused by a write-heavy primary, an undersized replica
instance class, or long-running queries on the replica. Scale the replica up or
reduce write volume. Watch the `ReplicaLag` CloudWatch metric.

⚠️ **Failover took ~1–2 minutes and connections dropped.** Expected for Multi-AZ
instance failover. Apps must implement connection retry / short DNS TTL handling.
RDS Proxy can mask much of this.

---

## Key Exam Points

- **Multi-AZ = availability**: synchronous standby, **same endpoint**, automatic
  failover, **standby not readable** in the classic DB instance deployment.
  The newer Multi-AZ DB cluster variant has two readable standbys; assume
  classic Multi-AZ unless the question explicitly says "DB cluster."
- **Read Replica = read scaling**: asynchronous, **separate endpoint(s)**, up to
  15, **cross-region capable**, **manual promotion** (no auto-failover).
- Using Multi-AZ to "scale reads" is the classic wrong answer.
- Need HA **and** read scaling? Use **both** together.
- Cross-region DR seed / global reads ⇒ **cross-region read replica**.
- Replica reads are **eventually consistent** (replica lag).

---

**Next**: [13_ecs_fargate_behind_alb.md — ECS Fargate Service Behind an ALB](13_ecs_fargate_behind_alb.md)
