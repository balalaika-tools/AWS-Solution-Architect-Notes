# The Four DR Strategies Compared — One App, One Concrete Scenario

> **Who this is for**: SAA-C03 candidates who can recite "backup & restore, pilot light, warm
> standby, active-active" but can't picture *what's actually running* in the DR Region for
> each, what the RTO/RPO and cost trade-offs are, and exactly which AWS service performs the
> failover. We take **one app** and run it through all four strategies. Builds on
> [Disaster Recovery](../14_hybrid_migration_dr/04_disaster_recovery.md).

---

## 1. The Scenario + RTO/RPO Refresher

**The app**: a web tier (EC2 in an ASG behind an ALB), a relational database
(Aurora PostgreSQL), and static assets in S3. It runs in **us-east-1** (primary). We want it
to survive a full-Region outage by failing over to **us-west-2** (DR). The only thing that
changes across the four strategies is *how much of us-west-2 is pre-built and running*.

Two numbers drive every DR decision:

| Term | Question | Set by |
|------|----------|--------|
| **RTO** — Recovery Time Objective | *How long until we're back up?* | How much is pre-provisioned in DR |
| **RPO** — Recovery Point Objective | *How much data can we lose?* | Replication/backup **frequency** |

> **Key insight**: RTO is bought with **standing compute** (warm capacity costs money 24/7).
> RPO is bought with **replication frequency** (continuous replication ≈ zero RPO). The four
> strategies are just four points on the "how much do you pre-pay for speed" curve.

---

## 2. Backup & Restore

**Running in DR**: *nothing* — only backups stored cross-Region.

```
   us-east-1 (PRIMARY)                         us-west-2 (DR)
  ┌────────────────────┐                      ┌────────────────────────┐
  │ ALB→ASG→Aurora, S3 │── AWS Backup copy ──▶ │  backup vault          │
  │   (live)           │── S3 CRR ───────────▶ │  S3 bucket (replicated)│
  │                    │── AMIs/snapshots ───▶ │  AMIs, DB snapshots     │
  └────────────────────┘                      │  (no running compute)   │
                                              └────────────────────────┘
        DR event ──▶ provision VPC + EC2 from AMIs, restore Aurora from snapshot,
                     point Route 53 at the new endpoints  (hours)
```

- **RTO**: hours. **RPO**: hours (last backup/snapshot).
- **Cost**: **$** (lowest) — you pay only for storage of backups + S3.
- **Failover**: manual/scripted — restore snapshots, launch from AMIs, flip Route 53.
- **Services**: AWS Backup cross-Region copy, **S3 Cross-Region Replication**, cross-Region
  AMI/EBS/RDS snapshot copies.

💡 The cheapest option. Exam keyword: *"lowest cost"* + *"downtime of hours is acceptable."*

---

## 3. Pilot Light

**Running in DR**: the **data layer** is live and replicating; the **app tier is OFF**.

```
   us-east-1 (PRIMARY)                          us-west-2 (DR)
  ┌────────────────────┐                       ┌──────────────────────────────┐
  │ ALB→ASG→Aurora     │── Aurora Global DB ──▶ │ Aurora replica (RUNNING)      │
  │   (live)           │   (continuous repl)   │ ASG min=0 / app servers OFF   │
  │                    │── S3 CRR ───────────▶ │ S3 replicated, AMIs ready     │
  └────────────────────┘                       └──────────────────────────────┘
        DR event ──▶ scale ASG up from 0 (AMIs ready), promote Aurora,
                     Route 53 failover  (10s of minutes)
```

- **RTO**: 10s of minutes – hours (must boot the app tier). **RPO**: seconds–minutes (DB is
  continuously replicating).
- **Cost**: **$$** — you pay for the always-on replica DB, but **no app compute**.
- **Failover**: promote the Aurora Global DB secondary, scale the ASG up from 0, Route 53
  failover.
- **Services**: **Aurora Global Database** (or DynamoDB Global Tables), S3 CRR, pre-baked AMIs.

✅ Faster than Backup & Restore (data is already there) and cheaper than Warm Standby (no
running app servers). ⚠️ The defining trait: **app tier is OFF** until disaster.

---

## 4. Warm Standby

**Running in DR**: a **scaled-down but fully running** copy of the *whole* stack.

```
   us-east-1 (PRIMARY)                          us-west-2 (DR)
  ┌────────────────────┐                       ┌──────────────────────────────┐
  │ ALB→ASG(10)→Aurora │── Aurora Global DB ──▶ │ ALB→ASG(1-2 RUNNING)→Aurora   │
  │   (full scale)     │   (continuous repl)   │ replica (running)             │
  │                    │── S3 CRR ───────────▶ │ everything LIVE but small     │
  └────────────────────┘                       └──────────────────────────────┘
        DR event ──▶ scale ASG out to full size, promote DB, Route 53 failover  (minutes)
```

- **RTO**: minutes. **RPO**: seconds–minutes.
- **Cost**: **$$$** — you pay for a small-but-always-running full stack.
- **Failover**: scale the already-running ASG up to production size, promote DB, Route 53
  failover. Can even take a trickle of live traffic to stay warm.
- **Services**: Aurora Global DB, ASG (min ≥ 1), ALB, S3 CRR, Route 53 health-check failover.

> **Rule (the most-tested distinction)**: in **Warm Standby** the **app tier is always
> running** (just small); in **Pilot Light** the **app tier is off**. Both keep the data layer
> replicating.

---

## 5. Multi-Site Active-Active

**Running in DR**: a **full-scale** copy serving **live production traffic** right now.

```
                         Route 53 (latency / weighted / geo routing)
                        ╱                                          ╲
  us-east-1 (ACTIVE)  ╱                                            ╲  us-west-2 (ACTIVE)
  ┌────────────────────┐   data choice: Aurora Global (one writer)  ┌────────────────────┐
  │ ALB→ASG→data tier  │◀──── OR DynamoDB Global Tables (MREC) ────▶│ data tier←ASG←ALB  │
  │ serving traffic    │◀────────────── S3 CRR ────────────────────▶│ serving traffic    │
  └────────────────────┘                                           └────────────────────┘
        DR event ──▶ fence failed writers, verify surviving data/capacity, then shift traffic
```

- **RTO**: near-zero. **RPO**: near-zero.
- **Cost**: **$$$$** (highest) — two full production environments.
- **Failover**: compute is already serving, but operators still need to fence writes, verify
  replication/capacity/dependencies, shift traffic safely, and reconcile asynchronous writes.
- **Services**: Route 53 (latency/weighted/geolocation routing + health checks), **DynamoDB
  Global Tables** (multi-master) or Aurora Global DB, S3 CRR.

⚠️ Hardest to build — you must handle data conflicts/consistency across active Regions.
DynamoDB MREC Global Tables are multi-writer with eventual replication and last-writer-wins
conflict resolution. Aurora Global Database normally has one writer Region; a secondary must be
promoted, so active application tiers still need a single-writer routing/failover design.

---

## 6. The Comparison Table

| Strategy | Running in DR | RTO | RPO | Cost | Complexity | Failover trigger |
|----------|---------------|-----|-----|------|-----------|------------------|
| **Backup & Restore** | Nothing (backups only) | Hours | Hours | $ | Low | Restore + provision, then Route 53 |
| **Pilot Light** | Data layer on; app **off** | 10s min–hours | Min | $$ | Medium | Scale ASG from 0, promote DB, R53 |
| **Warm Standby** | Scaled-down full stack **running** | Minutes | Sec–min | $$$ | High | Scale ASG up, promote DB, R53 |
| **Multi-Site Active-Active** | Full-scale, **serving traffic** | Near-zero | Near-zero | $$$$ | Highest | R53 stops routing to dead Region |

These RTO/RPO labels are design categories, not AWS guarantees. DNS/client caching, replication
lag, quota/capacity shortages, dependencies, operator decision time, and data validation all count
against the measured result.

> **Mnemonic**: cheapest/slowest → priciest/fastest =
> **Backup & Restore → Pilot Light → Warm Standby → Multi-Site Active-Active.**

---

## 7. Decision Guidance (Exam Keywords)

- *"Lowest cost, downtime of hours is acceptable"* → **Backup & Restore**
- *"Database always replicating but app servers off until needed"* → **Pilot Light**
- *"Scaled-down but always-running copy, recover in minutes"* → **Warm Standby**
- *"Lowest downtime/data-loss exposure, both Regions serve users"* → **Multi-Site Active-Active**
- *"Lowest RPO"* on its own → continuous replication (Aurora Global DB / DynamoDB Global
  Tables) — present in Pilot Light and up.
- *"Lowest RTO"* → Active-Active; *"good RTO without two full environments"* → Warm Standby.

💡 The DR pattern is always: **data replication** (Aurora Global DB / DynamoDB Global Tables /
S3 CRR / Backup copy) + **traffic redirection** (Route 53 failover) + **how much compute is
pre-running** (the cost dial).

---

## 8. Executable Regional Failover Runbook

The runbook below assumes a warm-standby application: IaC, networking, ALB/WAF, and small app
capacity are already deployed in `us-west-2`; Aurora Global Database and S3 replicate there. Give
every step an owner, command/automation link, evidence location, timeout, abort criterion, and
break-glass role. A diagram is not a runbook.

### Readiness gates before an incident

- Continuously deploy the same versioned IaC and application artifact to both Regions. Avoid
  “create during disaster” dependencies on a CI system hosted only in the failed Region.
- Pre-request EC2, EIP, NAT, ALB, Lambda, database, KMS, and service quotas. A quota is not a
  capacity reservation; use capacity reservations or diversify instance types where startup
  capacity is an RTO dependency.
- Provision Regional ACM certificates, VPC endpoints/NAT/DNS, WAF rules, IAM roles, secrets,
  parameter/config copies, alarms, dashboards, and log destinations. Test every local ARN.
- Monitor Aurora/global-table replication, S3 replication status, backup-copy completion, secret
  replica status, KMS key state, and synthetic application reads in the recovery Region.
- Preconfigure **Route 53 ARC routing controls** with safety rules, Route 53 failover/weighted
  records with tested TTLs, or Global Accelerator endpoint groups/dials. Decide which mechanism is
  authoritative; do not let two controllers fight. An ARC routing control is a resilient operator
  switch, not a monitor of endpoint health—your health/decision system chooses when to change it.

### Failover sequence

1. **Declare and freeze.** The incident commander records the start timestamp, scope, recovery
   target, and latest safe deployment. Stop application/schema/IaC changes in both Regions.
2. **Fence the old writer.** Close or revoke primary write paths using an ARC safety-controlled
   switch, database/application write token, IAM deny, or network control. Automatic health checks
   alone do not prove the old Region cannot accept writes. If fencing cannot be confirmed, choose
   explicitly between availability and split-brain/data corruption risk.
3. **Measure the recovery point.** Record the last committed transaction/canary sequence in the
   primary and last replicated sequence in DR. Check Aurora lag/global cluster state, DynamoDB
   replication metrics, S3 object replication status, and backup timestamps. The difference is
   potential asynchronous data loss; get business approval if it exceeds the RPO.
4. **Validate infrastructure and capacity.** Confirm network routes, DNS, certificates, endpoints,
   KMS grants, security policies, ALB health, and quotas. Apply the pinned IaC release, then scale
   the ASG/ECS/Lambda provisioned concurrency and database class to the tested DR capacity. Do not
   start with data promotion if the application cannot run afterward.
5. **Activate data services.** For Aurora, use a managed switchover when both Regions are healthy
   and caught up; use unplanned failover/promotion only for disaster, understanding possible loss.
   DynamoDB Global Tables have no primary to promote—route to the healthy replica and handle MREC
   conflict/lag semantics. Verify required S3 replicas exist; CRR does not make an in-flight object
   magically present.
6. **Activate local secrets and keys.** Use recovery-Region KMS key/replica ARNs and verify their
   independent policies/grants. If a Secrets Manager primary Region is unavailable and rotation
   must continue, promote the replica to a standalone secret and plan later reconciliation. Check
   the actual current credential against the promoted database before scaling clients.
7. **Start and verify the application.** Run backward-compatible migrations only, warm caches,
   start workers after their dependencies, and execute synthetic create/read/update workflows.
   Verify logs, traces, queue consumers, idempotency stores, external callbacks, and outbound allow
   lists—not only the ALB health endpoint. Force database clients to discard old connection pools,
   resolve the new writer endpoint, and reconnect with bounded exponential backoff; never pin a
   database IP or continue writing through a stale pooled connection.
8. **Shift traffic deliberately.** Change the ARC routing control/Global Accelerator traffic dial,
   or Route 53 record, first to a canary percentage where supported. Observe health and errors,
   then drain primary traffic. Route 53 change propagation is not the same as all recursive/client
   caches honoring the TTL.
9. **Stabilize and reconcile.** Compare business counts/checksums and the canary sequence, replay
   approved events, isolate conflicts, and keep the old writer fenced. Record traffic-restored and
   integrity-verified timestamps separately; the latter may define full recovery.

If any gate fails, stop at a safe state or roll traffic back only if the original Region is proven
healthy and has remained authoritative. Do not oscillate between Regions under partial recovery.

---

## 9. Failback Is a Controlled Migration

Never “flip DNS home” after the old Region recovers. It may contain stale data and stale secrets.

1. Keep the recovered former primary fenced and rebuild/patch it from the current IaC/artifacts.
2. Establish **reverse replication from the current writer** or add the Region back as a clean
   secondary/replica. Re-copy objects/backups and reconcile events written during the incident.
3. Prove replication lag is within the failback RPO, compare integrity checks, and run synthetic
   reads against the former primary.
4. Schedule a planned change. Freeze or quiesce writes, wait for confirmed catch-up, then use a
   managed database switchover or change the application write token exactly once.
5. Shift a traffic canary, observe, then complete the move. Keep the previous Region ready as the
   rollback target until the bake period and integrity checks pass.
6. Restore normal replication direction, autoscaling floors, routing safety rules, backup plans,
   alarms, secret rotation ownership, and cost posture. Close temporary break-glass access.

For an MREC multi-writer table, failback is traffic rebalancing rather than database promotion,
but recent cross-Region writes can still conflict. Use item/tenant Region pinning or application
versions/idempotency where last-writer-wins would be unsafe.

---

## 10. Repeatable DR Tests and Measured RTO/RPO

Create a test schedule proportional to risk—for example frequent backup restore/integrity tests,
quarterly component or AZ failure exercises, and a periodic full Regional game day. Test in an
isolated recovery environment or with explicit production safety controls; a tabletop alone does
not exercise IAM, quotas, DNS, keys, or data.

### Measure with durable markers

Emit a monotonically increasing, timestamped canary record through the real write path and let it
replicate with normal data. During the exercise record:

```text
t0  failure declared / primary traffic impaired
t1  old writer fenced
t2  data promoted or healthy replica selected
t3  first successful end-to-end DR write and read
t4  target traffic restored at acceptable error/latency
t5  reconciliation and integrity verification complete

measured service RTO = t4 - t0
measured full recovery = t5 - t0
measured RPO = last primary-committed canary - last intact DR canary
```

Check more than record presence: row/item counts by partition/time, checksums or domain totals,
referential/business invariants, S3 object count/version/metadata, queue/outbox offsets, and the
ability to decrypt/restore with recovery-Region roles. Record stale/conflicting/lost writes
explicitly instead of rounding replication lag down to “zero.”

### Inject realistic failures

- terminate or isolate instances/AZ paths and use AWS Fault Injection Service for supported
  component faults;
- block a dependency, revoke a KMS grant, disable a secret endpoint, exhaust a test quota, or
  remove a DNS path to expose hidden Regional dependencies;
- simulate loss of the deployment/control system and require the runbook's recovery artifacts;
- test a corrupted deployment/data case where **failover would copy the problem**, requiring
  point-in-time restore rather than promotion.

FIS does not create a literal AWS Region outage. The exercise must still simulate the decision,
write fencing, control-plane impairment, asynchronous loss, and traffic shift safely.

Use dashboards spanning both Regions: replication lag/errors, ALB/API health, database/queue
state, KMS/secret access, capacity/scaling failures, DNS/ARC/Global Accelerator state, synthetic
transactions, and business success metrics. Preserve CloudTrail and automation output as evidence.

### Close the loop

Define rollback criteria before the test. Afterward, publish measured objectives, timeline,
manual steps, failed assumptions, data discrepancies, and cost/capacity observations. Every lesson
gets an owner and due date and changes IaC, alarms, quotas, architecture, or the runbook. Re-run the
failed step. A DR exercise is complete when the fix is proven, not when the meeting ends.

---

## 11. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Failover DNS not taking effect | Client/resolver caching the old record | Lower the failover record **TTL** (e.g. 60 s) ahead of time |
| Route 53 won't fail over | No **health check** attached, or failover policy misconfigured | Attach a health check to the primary record; use **failover** routing |
| RPO worse than promised | Relying on periodic snapshots, not continuous replication | Use Aurora Global DB / DynamoDB Global Tables for sub-minute RPO |
| DR launch fails — AMI not found | AMIs/snapshots not copied to the DR Region | Pre-copy AMIs & snapshots cross-Region; automate the copy |
| Aurora DR Region read-only after failover | Secondary not **promoted** | Promote the Aurora Global DB secondary to a standalone primary |
| Active-Active data conflicts | Concurrent writes to the same item in two Regions | Use DynamoDB Global Tables (multi-master, last-writer-wins) or partition writes by Region |
| DR stack deploys but cannot scale | Quota was copied as documentation, not raised/tested; instance type lacks capacity | Pre-raise quotas, diversify/reserve capacity, and include a full-scale launch test |
| Both Regions accept writes after failover | Health-based routing changed traffic but did not fence the old writer | Use an authoritative write token/routing control and safety rules before promotion |
| Data decrypt fails only in DR | Local KMS replica policy/grants or secret state was never tested | Provision independent Regional authorization and run synthetic decrypt/credential tests continuously |
| Failback corrupts newer recovery data | Old primary was routed back before reverse replication/reconciliation | Rebuild as a secondary, catch up from the current writer, then perform a planned switchover |

❌ A common trap: choosing Active-Active for a "minimize cost" requirement — it's the *most*
expensive. ✅ Match the strategy to the **stated RTO/RPO and cost constraint**, not to "best."

---

## Key Exam Points

- Four strategies, cheapest/slowest → priciest/fastest: **Backup & Restore → Pilot Light →
  Warm Standby → Multi-Site Active-Active**.
- **RTO** is bought with standing compute; **RPO** with replication frequency.
- **Pilot Light vs Warm Standby**: both replicate data continuously; Pilot Light's **app tier
  is off**, Warm Standby's **app tier is always running** (small).
- **Active-Active** = both Regions serve live traffic; near-zero RTO/RPO; highest cost; needs
  multi-master data (DynamoDB Global Tables) and Route 53 routing.
- Failover almost always involves **Route 53 health checks + failover/latency routing**.
- Data layer: **Aurora Global Database** (sub-second cross-Region replication), **DynamoDB
  Global Tables** (multi-master), **S3 CRR**, **AWS Backup cross-Region copy**.
- "Lowest cost / hours OK" → Backup & Restore. The strictest tested RTO/RPO can point to
  Multi-Site Active-Active, but the architecture still needs write fencing, conflict semantics,
  capacity, dependency, failback, and measured recovery tests.
- Regional failover order is **fence → measure replication/RPO → validate capacity/infra →
  activate data/secrets/keys → start/test app → shift traffic → reconcile**.
- Failback rebuilds the former primary from the current authority and catches it up before a
  planned switchover. DNS reversal alone is unsafe.
- Measure RTO/RPO with durable canaries and business integrity checks; feed every failed assumption
  back into IaC, quotas, alarms, architecture, and the runbook.

---

**Next**: [20_vpc_endpoints_private_aws_services.md — Private AWS Service Access with VPC Endpoints](20_vpc_endpoints_private_aws_services.md)
