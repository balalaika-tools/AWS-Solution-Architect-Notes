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
| **Multi-Site Active-Active** | A **full-scale** copy serving live traffic | Near-zero target | Near-zero or zero target, only when the data design supports it | $$$$ | Highest |

These ranges are planning shorthand, not AWS SLAs. Actual RTO includes detection, orchestration,
client reconnection and validation; actual RPO depends on each data service's consistency and lag.

> **Rule**: The exam phrases the question around **RTO/RPO targets** and **budget**. Map the
> keywords: *"lowest cost / can tolerate hours"* → Backup & Restore; *"minimal running infra
> but faster than restore"* → Pilot Light; *"running scaled-down copy / quick failover"* →
> Warm Standby; *"near-zero objective, both sites already serving"* → Multi-Site Active-Active,
> followed by a check that the selected data model can actually meet the stated RPO.

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
simultaneously. If one Region fails, the other is already serving, which can support a near-zero
RTO target. RPO is a separate data-consistency decision; asynchronous replication can still lose
the latest writes. This is the most expensive and complex pattern.

```
 Region A (full, live)         Region B (full, live)
 ┌────────────────────┐        ┌────────────────────┐
 │ App fleet (full)   │◀──────▶│ App fleet (full)   │
 └─────────▲──────────┘        └─────────▲──────────┘
           └────────── Route 53 routes users to both ──────────┘
       both use an explicit data authority / consistency model
                  DISASTER in A → existing B capacity absorbs traffic
```

✅ Choose when the business demands a **near-zero downtime objective**, both sites must already
serve, and the selected database/object/event design can prove the required RPO. Aurora Global
Database itself is single-writer: secondary Regions are read-only until promotion, even when local
write forwarding sends writes back to the primary.

---

## 7. AWS Services That Enable DR

| Service / feature | DR role |
|-------------------|---------|
| **Route 53 health checks + failover routing** | Redirects DNS according to record/health state. For Pilot Light, an orchestrated runbook must first promote data, restore dependencies, start compute and validate it; DNS is the final traffic step, not the recovery orchestrator. |
| **Aurora Global Database** | Asynchronous cross-Region replication with one writer Region; switchover/failover can promote a secondary. Measure lag, promotion and reconnection rather than treating a target time as guaranteed. |
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
- *"Near-zero objective, both Regions already serving"* → **Multi-Site Active-Active**, with an
  explicit consistency/conflict design that proves the requested RPO
- *"Redirect clients after recovery is ready"* → **Route 53 failover routing**, optionally guarded
  by ARC routing controls; the application health/orchestration system decides when to move
- *"Lowest RPO for a relational DB across Regions"* → **Aurora Global Database**

> Worked scenarios comparing all four patterns side by side:
> [../18_practical_examples/19_disaster_recovery_strategies.md](../18_practical_examples/19_disaster_recovery_strategies.md).

---

## 9. Service-Specific Recovery Playbooks

The four strategies describe **how much environment is warm**. A real plan also defines the
promotion and consistency behavior of every stateful service.

| Data/workload | Recovery mechanism | Promotion and data concern |
|---------------|--------------------|----------------------------|
| **Aurora Global Database** | Cross-Region secondary cluster; managed switchover for planned moves or failover during an outage | Applications need a regional/cluster endpoint strategy. Measure replication lag and decide whether to wait, accept the RPO, or use the last-resort detach path. Re-establish the global topology after promotion. |
| **DynamoDB global tables** | Active-active replicas; choose MREC or, where supported and justified, MRSC consistency | Multi-Region writes require conflict/idempotency thinking. Regional application dependencies, IAM, streams and quotas do not appear merely because the table replica exists. |
| **S3 multi-Region data** | Versioning plus CRR/SRR, optionally Multi-Region Access Points and failover controls | Replication is asynchronous and existing objects need an explicit batch/backfill plan. Delete markers, KMS keys and replication failures must match the recovery intent. Keep protected backups for logical corruption. |
| **Servers/VMs** | AWS Elastic Disaster Recovery continuous block replication | Preconfigure launch settings, subnet/security/IAM, licensing and post-launch automation. Regular recovery drills must prove boot plus application dependencies, not only disk replication. |
| **Traffic** | Route 53, Global Accelerator, load balancers, or ARC controls depending on protocol and control needs | Health should represent the application and its dependencies. DNS TTL, resolver caching and connection draining mean traffic movement is not instantaneous. ARC routing controls and safety rules can reduce unsafe operator actions. |

### Route 53 ARC routing control is a recovery switch, not a monitor

An ARC routing control is an on/off state connected to a special Route 53 health check used by the
application's DNS records. The control itself does **not** measure application health. Monitoring
and the recovery commander decide when the standby is ready, then change the control state through
ARC's highly available data plane. A cluster exposes five Regional endpoints; the runbook/CLI tries
them in succession instead of depending on one endpoint or the management console.

Use an **assertion safety rule** to prevent all cells from being turned off, and a **gating safety
rule** when a deliberate master control must permit a class of failover changes. Configure clusters,
control panels, routing controls, Route 53 records and safety rules before the incident, then test
the exact data-plane credentials and commands. Routing control changes DNS health state; clients
still observe TTL/caching and connection behavior, and a Pilot Light remains unsafe to route to
until its data, dependencies and compute have passed validation.

The **data plane** that serves requests during an incident must not depend on a failed primary
Region's **control-plane** API. Pre-create what the RTO cannot tolerate creating: accounts,
networking, KMS keys, secrets or secret replicas, certificates, IAM roles, quotas, observability,
runbook access and minimum compute. Keep infrastructure definitions and deployment artifacts in a
location the recovery team can reach independently.

### Dependency-ordered regional failover

1. Declare the incident and freeze conflicting changes. Establish one recovery commander and a
   durable decision log.
2. Confirm the recovery Region, operator access, service quotas and blast radius. Do not fail over
   into an unhealthy or capacity-constrained Region.
3. Fence or make the old writer read-only where possible. This prevents **split brain** while
   promoting data stores.
4. Promote data services and validate the newest consistent recovery point. Restore secrets/keys
   and shared identity/network dependencies before application consumers.
5. Scale/start compute, then run internal synthetic and business-transaction tests.
6. Shift a small portion of traffic when the front door supports it; observe errors, latency and
   data correctness before completing the shift.
7. Communicate the measured RTO/RPO and any accepted data loss. Continue integrity checks and
   preserve evidence.

Failback is another migration, not the reverse of a DNS change. Rebuild or resynchronize the old
Region, establish one write authority, rehearse the direction change, move traffic gradually and
remove temporary capacity only after stabilization.

---

## 10. Proving Recovery Readiness

A document that has never run is a hypothesis. Test at increasing scope:

- **Component restore:** restore a backup, promote a replica, assume the recovery role and decrypt
  data in an isolated environment.
- **Game day:** execute the human runbook with a declared scenario, observers, communications and
  timed checkpoints. Include a missed approval, expired credential or absent owner—not just the
  happy path.
- **Failure injection:** where safe, use AWS Fault Injection Service or controlled changes to stop
  instances, impair an AZ dependency or exercise application fallback. Define abort conditions
  and exclude destructive data tests unless explicitly designed.
- **Regional exercise:** validate dependency order, standby capacity, quotas, certificates,
  keys/secrets, DNS/Global Accelerator behavior, client reconnection, scaling and operational
  access. A health-check flip alone is insufficient.

Measure **actual** RTO from declaration to restored business service and **actual** RPO from the
newest consistent committed record—not from a vendor target. Validate counts, checksums or
business aggregates, plus asynchronous queues and downstream reconciliation. Record manual steps,
unexpected dependencies and recovery capacity, then update automation and repeat the failed phase.

Each runbook needs prerequisites, exact commands or automation, IAM role, expected evidence,
timeouts, abort/rollback conditions, communication owners and a tested failback path. Review it
after architecture, staffing, Region, KMS, DNS or quota changes.

---

## Key Exam Points

- **RPO = data loss (backward); RTO = downtime (forward).** Lowering either raises cost.
- Four strategies, **cheapest/slowest → priciest/fastest**: **Backup & Restore → Pilot Light →
  Warm Standby → Multi-Site Active-Active.**
- The differentiator is **what's running in the DR Region**: nothing → data only → scaled-down
  full copy → full live copy.
- **Pilot Light**: app tier **OFF**, data replicating. **Warm Standby**: app tier **ON but
  small**. That one distinction is a frequent exam question.
- **Route 53/ARC redirects traffic after recovery prerequisites are ready**; it does not promote
  data or start a Pilot Light by itself.
- **Aurora Global Database** gives the lowest cross-Region RPO for relational data.
- Pre-create every control-plane dependency that cannot fit inside the RTO, and promote stateful
  services before compute and traffic.
- Prove RTO/RPO through timed recovery and data-integrity tests; architecture diagrams are not
  recovery evidence.

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
- ❌ Promoting two writers or failing back without fencing and reconciliation, creating split
  brain or silent data loss.

---

**Next**: [../15_cost_well_architected/01_cost_optimization.md — Cost Optimization](../15_cost_well_architected/01_cost_optimization.md)
