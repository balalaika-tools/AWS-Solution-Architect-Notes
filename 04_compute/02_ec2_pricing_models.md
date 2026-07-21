# EC2 Pricing Models

> **Who this is for**: SAA-C03 candidates. Cost optimization is one of the most heavily tested EC2 topics — the exam loves "which purchasing option is cheapest for *this* workload" questions. This file gives you the model for each option and a decision table to map workloads → the right choice. Assumes you've read [01_ec2_fundamentals.md](01_ec2_fundamentals.md).

---

## 1. The Six Purchasing Options at a Glance

You always pay the same *capacity*; what changes is the **pricing commitment** and **flexibility** you accept in return for a discount.

```
  Flexibility  ◀─────────────────────────────────────────▶  Discount
  (no commit)                                              (deep commit)

  On-Demand   Capacity      Savings    Reserved    Spot
              Reservations  Plans      Instances
  full price                ~up to 72% off            ~up to 90% off
  no commit                 1 or 3 yr commit          can be interrupted

  (orthogonal: Dedicated Hosts / Dedicated Instances = tenancy, not a discount)
```

| Option | Commitment | Discount vs On-Demand | Can be interrupted? | Best for |
|--------|-----------|----------------------|---------------------|----------|
| **On-Demand** | None | 0% (baseline) | No | Short-term, spiky, unpredictable, first-time/dev workloads |
| **Reserved Instances** | 1 or 3 yr | up to ~72% | No | Steady-state, predictable, known instance type |
| **Savings Plans** | 1 or 3 yr ($/hr) | up to ~72% | No | Steady-state with flexibility across instance families / regions / even Lambda & Fargate |
| **Spot** | None | up to ~90% | **Yes (2-min notice)** | Fault-tolerant, stateless, batch, flexible-time work |
| **Dedicated Host** | On-Demand or reserved | — | No | Compliance, BYOL licensing tied to physical sockets/cores |
| **Capacity Reservation** | None (pay while held) | 0% (combine w/ RI/SP for discount) | No | Guaranteeing capacity in a specific AZ |

---

## 2. On-Demand

Pay by the second (Linux/Windows) or hour, no commitment, no upfront. You can launch and terminate freely.

- ✅ Use for short-lived, unpredictable, or first-time workloads where you can't commit to a year.
- ❌ The most expensive per-hour option for anything long-running. Don't run a steady 24/7 fleet On-Demand if you can commit.

> **Mental model**: On-Demand is the *baseline* every other option is discounted against.

---

## 3. Reserved Instances (RIs)

A **1- or 3-year commitment** to a specific instance configuration in exchange for a big discount. You're billed for the reservation whether or not you run the instance.

### Standard vs Convertible

| | **Standard RI** | **Convertible RI** |
|--|------------------|---------------------|
| Discount | Highest (up to ~72%) | Lower (up to ~54%) |
| Change instance attributes? | Only modify (AZ, size within family, scope) | **Can exchange** for a different family/OS/tenancy of equal or greater value |
| Sell on Marketplace? | Yes | No |
| Best when | You're certain of your needs | You expect to change instance types |

### Term and payment options

- **Term**: 1 year or 3 years (3 yr = bigger discount).
- **Payment**: **All Upfront** (biggest discount) > **Partial Upfront** > **No Upfront** (smallest discount, but still discounted vs On-Demand).
- **Scope**: *Regional* (flexible across AZs in the region, no capacity reservation) or *Zonal* (locked to one AZ, **also reserves capacity** there).

⚠️ RIs are a **billing discount**, not an instance you launch — you still launch normal instances, and matching ones get the discounted rate.

---

## 4. Savings Plans

A commitment to a consistent **dollar-per-hour** spend for 1 or 3 years. More flexible than RIs because you commit to *spend*, not a specific instance.

| Type | Applies to | Flexibility | Discount |
|------|-----------|-------------|----------|
| **Compute Savings Plan** | EC2 **+ Fargate + Lambda**, any region, any family, any OS, any tenancy | Most flexible | up to ~66% |
| **EC2 Instance Savings Plan** | EC2 only, locked to an **instance family in one region** (can change size/AZ/OS) | Less flexible | up to ~72% (highest) |
| **SageMaker Savings Plan** | SageMaker ML instance usage | SageMaker only | up to ~64% |

- ✅ **Compute Savings Plan**: pick this when you want the deepest commitment discount *without* locking yourself to a family — and it even covers serverless (Lambda/Fargate).
- ✅ **EC2 Instance Savings Plan**: pick when you know you'll stay on a family (e.g. always `m5` in `us-east-1`) and want maximum discount.

> **Exam tip**: "Steady-state usage but the team keeps changing instance families" → **Savings Plan** (especially Compute) over Standard RI. "Maximum discount, fixed family forever" → EC2 Instance Savings Plan or Standard RI.

---

## 5. Spot Instances and Spot Fleet

**Spot** sells AWS's *spare* capacity at up to ~90% off On-Demand. The catch: AWS can **reclaim** the instance with a **2-minute warning** when it needs the capacity back (or when your max price is exceeded).

```
   AWS has spare capacity ──▶ you run cheap (Spot)
                              │
   AWS needs it back ─────────┤  2-minute interruption notice
                              ▼
   Instance interrupted: stop / hibernate / terminate (your choice)
```

- **Spot interruption notice**: delivered via instance metadata and an EventBridge event ~2 minutes before reclamation. Your app should checkpoint or drain on this signal.
- **Spot Fleet / EC2 Fleet**: launches a **mix** of Spot (and optionally On-Demand) across multiple instance types and AZs to meet a target capacity, using allocation strategies (`capacity-optimized` is recommended to minimize interruptions).

### When Spot is safe (and not)

- ✅ Fault-tolerant, stateless, or checkpointable: **batch jobs, big-data processing, CI/CD workers, image/video rendering, ML training that checkpoints, containerized stateless workers**.
- ✅ Flexible start/end time work that can be retried.
- ❌ **Never** for steady-state databases, stateful single-node services, or anything that can't tolerate sudden termination.

> **Exam trigger phrase**: "fault-tolerant," "can tolerate interruptions," "batch," "cheapest possible" → **Spot**.

---

## 6. Dedicated Hosts vs Dedicated Instances

Both give you **single-tenant** hardware (no other customers on the same host), but differ in *visibility and licensing*. This is **tenancy**, separate from the discount options above.

| | **Dedicated Instance** | **Dedicated Host** |
|--|-------------------------|---------------------|
| Isolation | Hardware not shared with other accounts | Entire **physical server** dedicated to you |
| Visibility into sockets/cores | No | **Yes** — you see and control physical sockets/cores |
| BYOL licensing (per-socket/per-core, e.g. Windows/Oracle) | Limited | ✅ Supported — the main reason to use it |
| Pricing | Per-instance | Per-host (On-Demand or reserved) |
| Use case | Need isolation but not core visibility | Compliance / **bring-your-own-license** tied to physical hardware |

> **Exam tip**: "Bring my own per-socket/per-core software license" or "see the physical cores" → **Dedicated Host**. "Just need physical isolation from other customers" → **Dedicated Instance**.

---

## 7. Capacity Reservations

Reserve compute capacity in a **specific AZ** for any duration, with **no commitment term and no discount** — you pay the On-Demand rate whether or not you use it. The point is *guaranteed availability*, not cost savings.

- Use when you must **guarantee** you can launch instances in an AZ (e.g. for a planned event, DR failover, or critical workload that can't risk an InsufficientCapacity error).
- ✅ Combine with a Savings Plan or Regional RI to get **both** the discount *and* the capacity guarantee.
- ⚠️ Don't confuse with RIs/Savings Plans — those are billing discounts; Capacity Reservations are about *reserving a slot in an AZ*.

---

## 8. Decision Table — Workload → Best Pricing Model

| Workload pattern | Best option | Why |
|------------------|-------------|-----|
| Steady 24/7 web/app server, known family, 1–3 yrs | **EC2 Instance Savings Plan** or **Standard RI** | Predictable → deepest commitment discount |
| Steady usage but families/regions keep changing | **Compute Savings Plan** | Discount with cross-family/region/serverless flexibility |
| Unpredictable, spiky, or short-term (dev/test, new project) | **On-Demand** | No commitment; pay only for what you use |
| Fault-tolerant batch / big-data / CI / rendering | **Spot (+ Spot Fleet)** | Up to 90% off; interruptions tolerable |
| Stateless containerized workers behind a queue | **Spot** (often mixed with On-Demand) | Cheap and replaceable |
| Must guarantee capacity in an AZ (event/DR) | **Capacity Reservation** (+ Savings Plan/RI for discount) | Reserves the slot, not just the price |
| Per-socket/per-core BYOL (Windows Server, Oracle) | **Dedicated Host** | Visibility into physical cores for licensing |
| Regulatory need for single-tenant hardware, no core visibility | **Dedicated Instance** | Physical isolation without host management |
| Mostly steady with predictable bursts | **Savings Plan/RI for the baseline + On-Demand or Spot for the burst** | Cover the floor cheaply, flex the peak |

```
            ┌─────────────────────────────────────────┐
            │ Can the workload tolerate interruption? │
            └───────────────┬──────────────┬──────────┘
                          yes              no
                            │               │
                          Spot     ┌────────┴─────────┐
                                   │ Long-term &      │
                                   │ predictable?     │
                                   └───┬──────────┬───┘
                                     yes          no
                                      │            │
                            Savings Plan / RI   On-Demand
```

---

## 9. Enterprise Purchasing and Capacity Planning

An enterprise should buy commitments against a measured portfolio, not one
instance at a time. In an AWS Organizations consolidated-billing family,
Reserved Instance and Savings Plans discounts can be shared across eligible
member-account usage when sharing is enabled. Central purchase improves the
chance that another workload consumes an otherwise unused commitment, but it
also requires clear chargeback rules: the account that bought the commitment
and the account whose usage received the discount may differ.

Two metrics answer different questions:

- **Utilization**: how much of the purchased commitment was consumed? Low
  utilization means money was committed but not used.
- **Coverage**: how much eligible usage received a commitment discount? Low
  coverage means too much of the steady baseline is still billed On-Demand.

Use Cost Explorer commitment recommendations only after separating durable
baseline demand from a migration spike, seasonal peak, or workload scheduled
for retirement. Then model the purchase under plausible growth and shrinkage,
including which account or business unit owns the risk.

### Price assurance and capacity assurance are separate

| Requirement | Mechanism | Scope and caution |
|-------------|-----------|-------------------|
| Discount a flexible compute baseline | Compute Savings Plans | Broadest compute flexibility, but it still does not reserve capacity. |
| Discount a stable EC2 family | EC2 Instance Savings Plans or RIs | Better discount potential, with more migration constraints. |
| Guarantee a normal instance type in an AZ | Zonal RI or On-Demand Capacity Reservation | Capacity is tied to an AZ and matching attributes; unused capacity can still cost money. |
| Share an On-Demand Capacity Reservation | Capacity Reservation sharing through AWS RAM | Share only to accounts that need the capacity and validate launch matching. |
| Reserve scarce supported accelerator capacity for a scheduled ML job | EC2 Capacity Blocks for ML | Purchase a fixed block for supported GPU/ML capacity in a future window; it is specialized, not a general RI replacement. |

A DR plan that owns only a Regional RI or a Savings Plan has a discounted price
but no assurance that the recovery AZ can launch the fleet. Conversely, a
Capacity Reservation without a matching discount provides capacity but may be
billed at the On-Demand rate. Combine the two only where the business value of
assured capacity justifies idle-reservation risk.

### Diversify interruptible capacity

For Spot, express capacity in workload units and offer several compatible
instance types across several AZs. Prefer a capacity-aware allocation strategy,
use **Capacity Rebalancing** where the orchestrator supports it, and retain an
On-Demand base for work that must continue. Diversification reduces correlated
interruptions; it does not make a stateful singleton safe on Spot.

### Commitments can block modernization

Before purchasing a one- or three-year commitment, ask whether the workload is
likely to move Regions, change processor architecture, adopt Fargate or Lambda,
or retire during that term. A larger theoretical discount can lose to a more
flexible plan if a database is moving to RDS, an x86 fleet is moving to
Graviton, or an acquisition changes account ownership. Treat commitment term,
technical roadmap, and migration wave as one decision.

---

## Key Exam Points

- ✅ **On-Demand** = no commitment, baseline price, spiky/short-term.
- ✅ **Reserved Instances** = 1/3-yr commitment to a config; **Standard** (cheaper, less flexible) vs **Convertible** (can exchange families). All/Partial/No Upfront.
- ✅ **Savings Plans** = commit to $/hr; **Compute** (most flexible, covers Lambda/Fargate) vs **EC2 Instance** (highest discount, locked family/region) vs **SageMaker**.
- ✅ **Spot** = up to 90% off, 2-minute interruption notice; only for **fault-tolerant / flexible / batch** work. Spot Fleet diversifies across types/AZs.
- ✅ **Dedicated Host** = physical server you can see (BYOL per-core licensing); **Dedicated Instance** = isolated hardware, no core visibility.
- ✅ **Capacity Reservation** = guarantee capacity in an AZ; no discount on its own — combine with Savings Plan/RI for both.
- ✅ **Utilization** measures consumption of commitments; **coverage** measures how much eligible usage received a discount.
- ✅ Consolidated billing can share eligible RI and Savings Plans benefits; define ownership and chargeback before central purchasing.
- ✅ **Capacity Blocks for ML** reserve supported accelerator capacity for a scheduled window; they are not a general-purpose EC2 discount.

---

## Common Mistakes

- ❌ Choosing Spot for anything stateful or steady-state (databases, primary app servers).
- ❌ Thinking Savings Plans / RIs *reserve capacity* — by default they're billing discounts (only Zonal RIs and Capacity Reservations guarantee capacity).
- ❌ Confusing **Convertible** (exchangeable, lower discount) with **Standard** (sellable on Marketplace, higher discount).
- ❌ Picking Dedicated Instance when the requirement is **per-socket BYOL** — that needs a **Dedicated Host**.
- ❌ Recommending RI when the workload's instance family is uncertain — a **Compute Savings Plan** is the more flexible answer.
- ❌ Forgetting you can mix: baseline on Savings Plan/RI, burst on Spot/On-Demand.
- ❌ Buying a long commitment from last month's usage without accounting for migrations, architecture changes, seasonality, or retirements.
- ❌ Assuming a billing commitment guarantees launch capacity during a zonal event or DR recovery.

---

## Relevant Limits

- **Savings Plans / RI terms**: 1 year or 3 years only.
- **Spot interruption notice**: **2 minutes** (via metadata + EventBridge).
- **Spot price**: now market-based and smoothed; you set a max price (defaults to the On-Demand price).
- **Capacity Reservations**: scoped to a single AZ; billed at On-Demand rate while active.
- Discounts quoted (~72% RI, ~66–72% Savings Plans, ~90% Spot) are approximate maximums and vary by instance/region.

---

**Next**: [Part 3: AMIs, User Data & Metadata](03_amis_userdata_metadata.md)
