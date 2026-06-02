# Disaster Recovery Strategies

> **Who this is for**: An engineer who can back up data (see [03_backup.md](03_backup.md)) but
> must now decide *how fast the whole application recovers* from an AZ or Region failure — and
> what each option costs. This is one of the highest-yield exam topics in the entire SAA-C03.

---

## 1. Prerequisite: RTO vs RPO

Two numbers define every DR plan. Confusing them is the single most common DR mistake.

| Term | Question it answers | Measures | Driven by |
|------|---------------------|----------|-----------|
| **RPO** — Recovery Point Objective | *How much data can we lose?* | Time **before** the disaster | Backup/replication **frequency** |
| **RTO** — Recovery Time Objective | *How long until we're running again?* | Time **after** the disaster | How much infra is **pre-provisioned** |

```
        RPO window                         RTO window
   │◀──────────────────▶│              │◀──────────────────▶│
   │  data written here │              │  app is DOWN here  │
   │  is LOST            │              │  (recovering)      │
───●────────────────────✗──────────────────────────────────●────▶ time
 last good            DISASTER                         service
 backup /              strikes                         restored
 replication
   ◀── before ────────────┤                ├──────── after ──▶
        (RPO)                                      (RTO)
```

> **Key insight**: **RPO looks backward** (data already written, now lost). **RTO looks
> forward** (downtime until recovery). Lowering *either* number costs more money. DR strategy
> is the trade-off between **cost** and **how small you can make RTO/RPO**.

---

## 2. The Four DR Strategies

AWS defines four strategies. As you go down the list, **RTO and RPO shrink** but **cost and
complexity rise**. The key differentiator is **how much is already running in the recovery
Region before disaster strikes.**

| Strategy | What's running in DR Region (steady state) | RTO | RPO | Cost | Complexity |
|----------|--------------------------------------------|-----|-----|------|-----------|
| **Backup & Restore** | Nothing — just backups stored | Hours | Hours | $ | Low |
| **Pilot Light** | Core **data** replicating; minimal services (DB) on, app servers **off** | 10s of min – hours | Minutes | $$ | Medium |
| **Warm Standby** | A **scaled-down but running** full copy | Minutes | Seconds–min | $$$ | High |
| **Multi-Site Active-Active** | A **full-scale** copy serving live traffic | Near-zero | Near-zero | $$$$ | Highest |

> **Rule**: The exam phrases the question around **RTO/RPO targets** and **budget**. Map the
> keywords: *"lowest cost / can tolerate hours"* → Backup & Restore; *"minimal running infra
> but faster than restore"* → Pilot Light; *"running scaled-down copy / quick failover"* →
> Warm Standby; *"zero downtime / no data loss"* → Multi-Site Active-Active.

---

## 3. Backup & Restore

Cheapest. You keep **backups** (often cross-Region — see file 03) and, when disaster hits, you
**provision infrastructure from scratch** and **restore** the data. Slowest RTO, but almost no
ongoing cost.

```
 Region A (primary)            Region B (DR)
 ┌───────────────┐             ┌───────────────────────┐
 │ App + DB live │──backups──▶ │  S3 / Backup vault     │
 └───────────────┘             │  (data only, no infra) │
                               └───────────┬────────────┘
                         DISASTER          │ on failure:
                               └──provision EC2/RDS + restore──▶ running
```

✅ Choose when downtime of **hours** is acceptable and cost must be minimal (dev/test, non-critical apps).

---

## 4. Pilot Light

The **core** (usually the **database**) is **always on and replicating** in the DR Region, but
the **application/compute tier is switched off**. On disaster, you **start and scale** the
already-defined app servers around the live data — like lighting a furnace from a pilot flame.

```
 Region A (primary)            Region B (DR — pilot light)
 ┌───────────────┐             ┌───────────────────────────┐
 │ App servers   │             │ App servers: OFF (defined) │
 │ DB (primary)  │──replicate─▶│ DB replica: ON ◀───────────│
 └───────────────┘             └───────────────────────────┘
                         DISASTER → start & scale app servers, repoint DNS
```

✅ Faster than Backup & Restore (data is already there and current). Cheaper than Warm Standby
(no app fleet running). Good middle ground for *"keep cost low but recover faster than a full
restore."*

---

## 5. Warm Standby

A **complete, functional copy** of the environment runs in the DR Region, but **scaled down**
(fewer/smaller instances). It can serve traffic immediately on failover; you then **scale it
up** to full capacity.

```
 Region A (primary, full)      Region B (DR — warm standby, small)
 ┌────────────────────┐        ┌──────────────────────────────┐
 │ App fleet (full)   │        │ App fleet (running, scaled-down)│
 │ DB (primary)       │─repl──▶│ DB replica (running)            │
 └────────────────────┘        └───────────────┬────────────────┘
                         DISASTER → failover + scale up to full size
```

✅ Choose when RTO must be **minutes** and you can pay for an always-on (small) duplicate.

⚠️ The difference from Pilot Light: in Warm Standby the **app tier is always running** (just
small); in Pilot Light the **app tier is off**.

---

## 6. Multi-Site Active-Active

A **full-scale** environment runs in **both** Regions and **both serve live traffic**
simultaneously. If one Region fails, the other absorbs all traffic with **near-zero RTO and
RPO**. Most expensive and complex.

```
 Region A (full, live)         Region B (full, live)
 ┌────────────────────┐        ┌────────────────────┐
 │ App fleet (full)   │◀──────▶│ App fleet (full)   │
 │ DB (read/write) ◀──Aurora Global / bidi──▶ DB    │
 └─────────▲──────────┘        └─────────▲──────────┘
           └────────── Route 53 routes users to both ──────────┘
                  DISASTER in A → all traffic flows to B, no downtime
```

✅ Choose when the business demands **zero/near-zero downtime and data loss** and cost is no
object (financial trading, tier-0 systems).

---

## 7. AWS Services That Enable DR

| Service / feature | DR role |
|-------------------|---------|
| **Route 53 health checks + failover routing** | Detects a dead Region and redirects DNS to the healthy one — the failover trigger for Pilot Light, Warm Standby, Active-Active. |
| **Aurora Global Database** | Sub-second cross-Region replication; promote a secondary Region in ~1 min — backbone of low-RPO DR. |
| **RDS cross-Region read replica** | Replicates a standard RDS DB to another Region; promote on failover. |
| **S3 Cross-Region Replication (CRR)** | Keeps objects available in a second Region. |
| **DynamoDB Global Tables** | Multi-Region, active-active table replication. |
| **AWS Backup cross-Region copy** | The data layer for Backup & Restore (file 03). |
| **Elastic Disaster Recovery (DRS)** | Continuous block replication for fast, automated failover of servers (rehost-style DR). |

> **Mental model**: DR strategy = *data replication mechanism* (Aurora Global / CRR / Global
> Tables / Backup copy) + *traffic redirection* (Route 53 failover) + *how much compute you
> keep warm*.

---

## 8. Choosing for the Exam

- *"Lowest cost, downtime of hours OK"* → **Backup & Restore**
- *"Database always replicating, app servers off until needed"* → **Pilot Light**
- *"Scaled-down but always-running copy, recover in minutes"* → **Warm Standby**
- *"Zero downtime, no data loss, both Regions serving"* → **Multi-Site Active-Active**
- *"Detect failure and switch Regions"* → **Route 53 failover routing + health checks**
- *"Lowest RPO for a relational DB across Regions"* → **Aurora Global Database**

> Worked scenarios comparing all four patterns side by side:
> [../18_practical_examples/19_disaster_recovery_strategies.md](../18_practical_examples/19_disaster_recovery_strategies.md).

---

## Key Exam Points

- **RPO = data loss (backward); RTO = downtime (forward).** Lowering either raises cost.
- Four strategies, **cheapest/slowest → priciest/fastest**: **Backup & Restore → Pilot Light →
  Warm Standby → Multi-Site Active-Active.**
- The differentiator is **what's running in the DR Region**: nothing → data only → scaled-down
  full copy → full live copy.
- **Pilot Light**: app tier **OFF**, data replicating. **Warm Standby**: app tier **ON but
  small**. That one distinction is a frequent exam question.
- **Route 53 health checks + failover routing** is the standard cross-Region failover trigger.
- **Aurora Global Database** gives the lowest cross-Region RPO for relational data.

---

## Common Mistakes

- ❌ Swapping **RTO and RPO** — RPO is *data lost*, RTO is *time down*.
- ❌ Confusing **Pilot Light** (app off) with **Warm Standby** (app running, scaled down).
- ❌ Picking Multi-Site Active-Active when the requirement only needs minutes of RTO — it's
  over-engineered and over-budget.
- ❌ Choosing Backup & Restore when the stated RTO is *minutes* — restoring from scratch takes
  too long.
- ❌ Forgetting the **traffic-redirection** half: replicated data with no Route 53 failover
  still leaves users pointed at the dead Region.

---

**Next**: [../15_cost_well_architected/01_cost_optimization.md — Cost Optimization](../15_cost_well_architected/01_cost_optimization.md)
