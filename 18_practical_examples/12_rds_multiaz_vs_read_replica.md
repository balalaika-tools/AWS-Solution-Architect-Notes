# RDS Multi-AZ vs Read Replicas, Side by Side

> **Who this is for**: Engineers prepping for SAA-C03 who keep confusing
> **Multi-AZ** (high availability) with **Read Replicas** (read scaling). They
> solve different problems and the exam tests whether you know which is which.
> Assumes you've read [RDS](../06_databases/02_rds.md) and
> [RDS in private subnets](11_rds_private_access.md).

> **One sentence**: A classic Multi-AZ **DB instance** is for same-Region
> availability (synchronous standby, one endpoint, automatic failover, standby
> not readable); a Read Replica is an asynchronous, separately addressed copy
> for read scaling or a manual DR seed. Multi-AZ DB clusters and Aurora Global
> Database add different recovery behavior, covered below.

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

- **Synchronous** replication — committed data is replicated to the standby,
  giving a very low RPO for the failures Multi-AZ covers. An interrupted client
  can still have an **ambiguous commit**, and Multi-AZ is not protection from
  corruption, operator error, or a Regional disaster.
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

> The newer **Multi-AZ DB cluster** is a separate architecture: one writer and
> two readable instances in three AZs, using semisynchronous replication. It has
> writer/reader endpoints and failovers are typically under 35 seconds, although
> recovery load and replica lag can make them longer. The classic Multi-AZ
> instance still has one non-readable standby and typically takes 60–120 seconds.

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
  for read queries. Implement read/write splitting in the application or an
  explicitly designed routing layer; RDS Proxy does not automatically combine
  independent RDS read-replica endpoints.
- Replica quotas and feature support vary by engine and Region. Replicas **can
  be cross-Region** for local reads or as a DR seed where supported.
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

Promotion stops replication and reboots the replica; it can take several
minutes or longer. It produces a **new standalone endpoint**—RDS does not move
the source endpoint or application traffic for you. Cross-Region replication
is asynchronous, so `ReplicaLag` at the failure point is the exposed data-loss
window.

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
later yields an HA primary. Read replication is separate from the source's
Multi-AZ standby and continues against the source endpoint after its failover.

---

## 4. Four Recovery Patterns

| Pattern | Replication/topology | Recovery action | Endpoint behavior | Data-loss exposure | Cost shape |
|---------|----------------------|-----------------|-------------------|--------------------|------------|
| **Multi-AZ DB instance** | Synchronous standby in another AZ; standby not readable | RDS automatically fails over; force with `reboot-db-instance --force-failover` for a test | Same DB endpoint changes address; existing connections drop | Very low for committed data in covered failures; ambiguous in-flight transactions still need handling | Roughly a second DB instance plus synchronous I/O; no read capacity from standby |
| **Multi-AZ DB cluster** | Writer + two readable instances in three AZs; semisynchronous | RDS promotes a readable instance automatically; test with `failover-db-cluster` | Use the cluster writer endpoint for writes and reader endpoint for reads, not fixed instance endpoints | Low within the Region, but measure replica lag and application commits | Three DB instances; readers provide useful capacity, with possible cross-AZ data-transfer effects |
| **Cross-Region RDS read replica** | Engine-native asynchronous replica in another Region | Operator checks lag, promotes, and redirects applications; promotion is irreversible | Promoted DB has a different endpoint; DNS/config/service discovery must change | Up to unapplied replication lag; can grow during network or source pressure | Full replica, storage, backup, cross-Region transfer, and idle recovery capacity |
| **Aurora Global Database** | One primary Region plus asynchronous storage-based secondary Regions | Planned managed **switchover** or unplanned managed **failover** to a secondary | Regional roles/endpoints change; validate the application's global writer-routing mechanism | Switchover waits for synchronization and is zero-data-loss; unplanned failover can lose data equal to cross-Region lag | Aurora capacity in each Region, global replication I/O/transfer, backups, and optional proxy/routing layer |

These are not substitutes for backups. Multi-AZ protects availability,
cross-Region copies protect a different failure scope, and point-in-time restore
protects against logical damage. Define an RPO/RTO and failure scope before
selecting the most expensive-looking check box.

For Aurora Global Database, use a **switchover** for a planned Region rotation:
Aurora waits for the secondary to catch up and preserves the global topology.
Use a **failover** only when the primary Region cannot serve; the data loss is
bounded by observed global replication lag. Failback should be another managed
zero-data-loss switchover after the old Region is healthy and synchronized—not
an ad hoc write reversal.

For an RDS cross-Region replica, planned recovery should quiesce source writes,
wait for lag to reach zero, promote, then change application routing. After an
unplanned promotion, make the new primary Multi-AZ if needed and create a new
replica in the reverse direction before calling DR complete. A promotion does
not create an automatic failback path.

---

## 5. Measure Failover, Promotion, and Failback

Do not use the console's status change as the RTO. Build a small canary that
writes a monotonically increasing transaction ID and reads it back through the
same endpoint the application uses. For every game day, capture:

1. `T0`: the last confirmed commit and the instant the fault/recovery action
   starts.
2. Time to first connection error, time for RDS events/alarms to fire, and time
   to the first successful **read and write** after recovery.
3. The highest durable transaction ID on the new writer. Separate confirmed
   loss from an ambiguous response whose transaction actually committed.
4. DNS answers and client connection-pool behavior throughout the event.
5. Time to restore redundancy—new standby/replica healthy—not merely time to
   accept traffic.

Use safe, approved mechanisms:

```bash
# Classic Multi-AZ DB instance: controlled failover in the same Region
aws rds reboot-db-instance \
  --db-instance-identifier app-prod-db \
  --force-failover

# Multi-AZ DB cluster: promote a readable instance
aws rds failover-db-cluster \
  --db-cluster-identifier app-prod-cluster
```

For a cross-Region replica, rehearse both a planned promotion with writes
quiesced and an isolated unplanned scenario using nonproduction data. For
Aurora Global Database, test a managed switchover regularly and tabletop the
data-loss decision for a managed failover. Record the achieved RTO/RPO and
compare it with the business target; service documentation's “typical” time is
not an SLA for your connection pools, DNS cache, and transaction recovery.

### Application connection behavior

- Always connect to the role-following DB/cluster/proxy endpoint. Direct
  instance endpoints stay with that instance and can defeat failover routing.
- Existing sockets do not migrate. Evict broken pool connections, re-resolve
  DNS, and retry with bounded exponential backoff plus jitter. For JVM clients,
  AWS recommends DNS cache TTL no greater than 60 seconds.
- Put a total deadline around retries so an outage does not consume every
  application worker. Apply circuit breaking/load shedding where appropriate.
- Make write operations idempotent or reconcile with a transaction/business
  key. Retrying an ambiguous timeout can otherwise create duplicate orders or
  payments.
- RDS Proxy can shorten or mask parts of same-Region reconnection and protect
  the DB from a connection storm. It does not choose a cross-Region primary for
  you and does not eliminate application transaction handling.

---

## 6. Decision Table

| Need / question phrase                          | Multi-AZ | Read Replica |
|-------------------------------------------------|:--------:|:------------:|
| Survive an **AZ failure** with auto failover    |   ✅     |     ❌       |
| Very low RPO for a covered AZ/instance failure  |   ✅     | ❌ (async lag)|
| **Scale read traffic** / offload reporting      |   ❌     |     ✅       |
| **Cross-region** copy / serve distant readers   |   ❌¹    |     ✅       |
| **Separate endpoint** to target                 |   ❌     |     ✅       |
| **Automatic** failover                          |   ✅     | ❌ (manual promote)|
| **Reduce maintenance downtime** (patch standby) |   ✅     |     ❌       |
| Replication type                                | synchronous | asynchronous |

¹ Classic Multi-AZ is within one region; cross-region HA is a separate design.

---

## 7. Troubleshooting / Common Exam Traps

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

⚠️ **A promoted cross-Region replica is healthy, but the app still fails.**
Promotion creates a writable standalone DB with its own endpoint. Update the
approved application routing/configuration, validate TLS hostname verification,
and rebuild replication in the reverse direction for failback.

⚠️ **An Aurora Global failover lost the latest writes.** An unplanned
cross-Region failover is allowed to lose data that had not reached the
secondary. Alert on global replication lag and define who can accept that loss;
use a managed switchover for planned moves.

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
- **Multi-AZ DB cluster** = three AZs, two readable instances, role-following
  cluster endpoints, and typically faster failover than a classic Multi-AZ instance.
- **Aurora Global Database** distinguishes zero-data-loss planned switchover
  from unplanned failover with lag-dependent data loss.
- Test the complete application path and measure durable transactions, DNS,
  reconnect, promotion, restored redundancy, and failback—not just RDS status.

---

**Next**: [13_ecs_fargate_behind_alb.md — ECS Fargate Service Behind an ALB](13_ecs_fargate_behind_alb.md)
