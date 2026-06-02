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
  ┌────────────────────┐        Aurora Global DB (bi-region)        ┌────────────────────┐
  │ ALB→ASG→Aurora     │◀───── DynamoDB Global Tables (multi-master)│ ALB→ASG→Aurora     │
  │ serving traffic    │◀────────────── S3 CRR ────────────────────▶│ serving traffic    │
  └────────────────────┘                                           └────────────────────┘
        DR event ──▶ Route 53 stops sending traffic to the dead Region. That's it. (near-zero)
```

- **RTO**: near-zero. **RPO**: near-zero.
- **Cost**: **$$$$** (highest) — two full production environments.
- **Failover**: there's barely a "failover" — Route 53 health checks simply stop routing to
  the dead Region; the other is already serving.
- **Services**: Route 53 (latency/weighted/geolocation routing + health checks), **DynamoDB
  Global Tables** (multi-master) or Aurora Global DB, S3 CRR.

⚠️ Hardest to build — you must handle data conflicts/consistency across active Regions
(DynamoDB Global Tables' multi-master "last writer wins" is the exam-friendly answer).

---

## 6. The Comparison Table

| Strategy | Running in DR | RTO | RPO | Cost | Complexity | Failover trigger |
|----------|---------------|-----|-----|------|-----------|------------------|
| **Backup & Restore** | Nothing (backups only) | Hours | Hours | $ | Low | Restore + provision, then Route 53 |
| **Pilot Light** | Data layer on; app **off** | 10s min–hours | Min | $$ | Medium | Scale ASG from 0, promote DB, R53 |
| **Warm Standby** | Scaled-down full stack **running** | Minutes | Sec–min | $$$ | High | Scale ASG up, promote DB, R53 |
| **Multi-Site Active-Active** | Full-scale, **serving traffic** | Near-zero | Near-zero | $$$$ | Highest | R53 stops routing to dead Region |

> **Mnemonic**: cheapest/slowest → priciest/fastest =
> **Backup & Restore → Pilot Light → Warm Standby → Multi-Site Active-Active.**

---

## 7. Decision Guidance (Exam Keywords)

- *"Lowest cost, downtime of hours is acceptable"* → **Backup & Restore**
- *"Database always replicating but app servers off until needed"* → **Pilot Light**
- *"Scaled-down but always-running copy, recover in minutes"* → **Warm Standby**
- *"Zero downtime, no data loss, both Regions serve users"* → **Multi-Site Active-Active**
- *"Lowest RPO"* on its own → continuous replication (Aurora Global DB / DynamoDB Global
  Tables) — present in Pilot Light and up.
- *"Lowest RTO"* → Active-Active; *"good RTO without two full environments"* → Warm Standby.

💡 The DR pattern is always: **data replication** (Aurora Global DB / DynamoDB Global Tables /
S3 CRR / Backup copy) + **traffic redirection** (Route 53 failover) + **how much compute is
pre-running** (the cost dial).

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Failover DNS not taking effect | Client/resolver caching the old record | Lower the failover record **TTL** (e.g. 60 s) ahead of time |
| Route 53 won't fail over | No **health check** attached, or failover policy misconfigured | Attach a health check to the primary record; use **failover** routing |
| RPO worse than promised | Relying on periodic snapshots, not continuous replication | Use Aurora Global DB / DynamoDB Global Tables for sub-minute RPO |
| DR launch fails — AMI not found | AMIs/snapshots not copied to the DR Region | Pre-copy AMIs & snapshots cross-Region; automate the copy |
| Aurora DR Region read-only after failover | Secondary not **promoted** | Promote the Aurora Global DB secondary to a standalone primary |
| Active-Active data conflicts | Concurrent writes to the same item in two Regions | Use DynamoDB Global Tables (multi-master, last-writer-wins) or partition writes by Region |

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
- "Lowest cost / hours OK" → Backup & Restore. "Zero downtime / no data loss" →
  Multi-Site Active-Active.

---

This is the final practical walkthrough — and the last file in the knowledge base. From here,
loop back to consolidate: review the **[service comparison cheat sheets](../17_exam_patterns/02_service_comparison_cheatsheets.md)**
for last-minute recall, then return to the **[repository home](../README.md)** to pick your
next review path.

**Next**: [Back to the repository home](../README.md) · [Service Comparison Cheat Sheets (for review)](../17_exam_patterns/02_service_comparison_cheatsheets.md)
