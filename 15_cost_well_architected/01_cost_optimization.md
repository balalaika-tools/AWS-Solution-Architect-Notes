# Cost Optimization — Drivers, Tools & Pricing Levers

> **Who this is for**: Engineers prepping for SAA-C03 who can launch resources but haven't
> formalized *how the bill is built* or *which tool answers which cost question*. Assumes
> you've seen [EC2 pricing models](../04_compute/README.md), [S3 storage classes](../05_storage/README.md),
> and [VPC endpoints / data transfer](../03_networking/README.md). The exam tests *tool
> selection* ("which service alerts on a budget threshold?") and *lever selection* ("how do
> we cut cost for an idle dev fleet?") far more than raw arithmetic.

---

## 1. Prerequisite: CapEx vs OpEx and the Cloud Cost Mindset

Before any tool, internalize *why* cloud cost works differently. Traditional data centers
are **CapEx** — capital expenditure: you buy servers up front, depreciate them over years,
and pay whether or not they're used. The cloud is **OpEx** — operational expenditure: you
pay per use, monthly, with no up-front asset.

| | CapEx (on-prem) | OpEx (cloud) |
|---|---|---|
| Spend timing | Large up-front | Pay-as-you-go, monthly |
| Asset ownership | You own and depreciate hardware | You rent capacity |
| Capacity decision | Guess peak years in advance | Scale to actual demand |
| Risk | Over-provision (waste) or under-provision (outage) | Match supply to demand |

> **Key insight**: The cloud's core financial promise is *replacing fixed CapEx guesses with
> variable OpEx that tracks demand*. Every cost-optimization lever below is really "stop
> paying for capacity you aren't using."

⚠️ OpEx cuts both ways. Because spend is variable and self-service, costs can sprawl
silently — forgotten test instances, un-deleted snapshots, chatty cross-region traffic.
Cost *governance* (the tools in §3) exists to make that variable spend visible.

---

## 2. The Four Cloud Cost Drivers

Almost every line on an AWS bill reduces to four things. If you can name the driver, you can
name the lever.

```
                   AWS BILL
                      │
      ┌───────────┬───┴────────┬──────────────┐
      ▼           ▼            ▼               ▼
  COMPUTE     STORAGE     DATA TRANSFER     REQUESTS
  EC2 hrs     GB-month    GB egress         API calls,
  Lambda      EBS/EFS/S3  (cross-AZ,        S3 GET/PUT,
  GB-sec      snapshots   cross-region,     Lambda invokes,
              backups     internet OUT)     DynamoDB RCU/WCU
```

| Driver | Billed by | Cheap | Expensive | The trap |
|--------|-----------|-------|-----------|----------|
| **Compute** | Instance-hours, Lambda GB-seconds | Spot, right-sized, off-when-idle | Over-provisioned 24/7 On-Demand | Idle but running instances |
| **Storage** | GB-month + IOPS/throughput | S3 IA/Glacier, gp3 over io2 | Old snapshots, over-provisioned io2 | Snapshots/volumes nobody deletes |
| **Data transfer** | GB moved | Within same AZ, into AWS | Egress to internet, cross-region | Cross-AZ and NAT Gateway traffic |
| **Requests** | Per million calls / per RCU-WCU | Batching, caching | Chatty fine-grained calls | Hot-loop API calls |

> **Rule — the data-transfer hierarchy (memorize this)**: **Data *into* AWS is free.**
> Data *out to the internet* (egress) costs the most. Cross-region costs. Cross-AZ costs (a
> small per-GB charge in *both* directions). Same-AZ via private IP is **free**. This single
> ranking answers a surprising number of exam questions.

💡 **Egress is the silent budget killer.** Serving large media to users? Put **CloudFront** in
front so most bytes leave from the cache (cheaper egress + offloaded origin) instead of from
S3/EC2 directly.

---

## 3. Billing & Cost Management Tools

This is the single most testable table in the whole topic. The exam loves "which tool…"
questions where two tools sound plausible — the discriminator is almost always
**alert/forecast (Budgets)** vs **analyze/visualize (Cost Explorer)** vs **raw line-item
data (CUR)**.

| Tool | What it does | Use when the question says… |
|------|--------------|------------------------------|
| **AWS Budgets** | Set a cost/usage/RI/SP budget; **alert (and optionally act)** when actual or *forecasted* spend crosses a threshold | "notify when spend will exceed \$X", "alert at 80% of budget", "take action when over budget" |
| **Cost Explorer** | Interactive UI to **visualize and analyze** past spend (up to 12 mo back) and **forecast** (up to 12 mo ahead); right-sizing & RI/SP recommendations | "visualize cost trends", "which service grew", "forecast next quarter", "RI purchase recommendation" |
| **Cost and Usage Report (CUR)** | The **most granular** billing data — hourly/by-resource line items delivered to **S3**, queryable with Athena/Redshift/Amazon Quick Sight | "most detailed/granular cost data", "analyze billing with SQL/Athena", "custom BI dashboards" |
| **Cost Allocation Tags** | Tag resources (`CostCenter`, `Project`, `Env`); activate tags so cost reports **break down by tag** | "attribute cost per team/project/environment", "show spend by department" |
| **Billing alarm (CloudWatch)** | A CloudWatch alarm on the `EstimatedCharges` metric → SNS notification | "simplest alarm on total estimated charges", legacy/CloudWatch-based alerting |
| **AWS Pricing Calculator** | Estimate cost of a planned architecture **before** deploying | "estimate monthly cost before building", "compare two designs' cost upfront" |
| **AWS Compute Optimizer** | ML-based **right-sizing** recommendations for EC2, ASGs, EBS, Lambda (over/under-provisioned) | "recommend a better instance type", "is this instance over-provisioned?" |
| **Trusted Advisor (Cost checks)** | Automated checks for **idle/underutilized** resources, unattached EBS, idle RDS/ELB, low-util RIs | "identify idle resources", "automated cost best-practice checks" |

> **Key discriminator — Budgets vs Cost Explorer vs CUR**:
> - **Budgets = act on the future** (alert/forecast against a threshold).
> - **Cost Explorer = understand the past** (charts, trends, recommendations).
> - **CUR = the raw data** (every line item, in S3, for your own queries).

```
 PLAN          BUILD                       OPERATE
  │              │                            │
Pricing      (deploy)                   ┌─────┴─────────────────────┐
Calculator                              │                           │
(estimate                          Cost Explorer               AWS Budgets
 before                            (analyze trends,            (threshold alerts
 spending)                          forecast, RI recs)          + actions)
                                         │                           │
                                  Compute Optimizer            Trusted Advisor
                                  (right-size recs)            (idle-resource checks)
                                         │
                                    CUR → S3 → Athena
                                  (granular line items)
```

⚠️ **Billing data lives only in the management (payer) account** by default. The billing
CloudWatch metric `EstimatedCharges` is published **only in us-east-1** — exam-classic gotcha:
your billing alarm must be created in **N. Virginia** regardless of where your resources run.

💡 **Budgets can take action**, not just notify. A budget action can apply an IAM
policy / SCP or stop EC2/RDS instances when a threshold is hit — useful for hard-capping a
sandbox account.

---

## 4. Pricing Levers — How to Actually Cut the Bill

Tools *find* waste; these levers *remove* it. Map each to its cost driver from §2.

### Compute: right-size, commit, or go Spot

| Model | Discount vs On-Demand | Commitment | Best for |
|-------|----------------------|------------|----------|
| **On-Demand** | baseline | none | spiky/unknown, short-lived |
| **Spot** | up to ~90% | none (can be reclaimed w/ 2-min notice) | fault-tolerant, stateless, batch |
| **Reserved Instances (RI)** | up to ~72% | 1 or 3 yr, specific family/region | steady-state, predictable instances |
| **Savings Plans** | up to ~72% | 1 or 3 yr, a \$/hr commitment | steady spend with **flexibility** across family/region/even Lambda & Fargate |

> **Rule**: **Savings Plans = RI-level discount with more flexibility.** Compute Savings
> Plans cover EC2 *and* Fargate *and* Lambda and let you change instance family/size/region.
> Prefer them over standard RIs unless you specifically need RI capacity reservation or
> Convertible-RI marketplace resale. Use **Spot** for anything interruption-tolerant.

✅ Right-size first, *then* commit. Buying a 3-year RI for an oversized instance locks in
waste. Run **Compute Optimizer** → right-size → buy Savings Plans on the corrected baseline.

### Storage: classes, lifecycle, and cleanup

- **S3 storage classes** — move infrequently accessed data to **S3 Standard-IA / One Zone-IA**;
  archive to **Glacier Instant/Flexible/Deep Archive**. Use **S3 Intelligent-Tiering** when
  access patterns are unknown — it auto-moves objects between tiers (small monitoring fee, no
  retrieval cost).
- **Lifecycle policies** — automate transition (e.g., → IA after 30d, → Glacier after 90d) and
  **expiration** (delete after N days). Also expire **old object versions** and **incomplete
  multipart uploads** (a sneaky recurring charge).
- **EBS** — prefer **gp3** over gp2 (cheaper, IOPS/throughput decoupled); delete **unattached
  volumes** and **stale snapshots**; use **Data Lifecycle Manager** to age out snapshots.

### Database: pick the right one, then size it

- Don't run a self-managed DB on EC2 if a managed service fits — managed removes admin cost.
- **DynamoDB on-demand** for spiky/unpredictable; **provisioned + auto scaling** for steady,
  predictable load (cheaper at scale).
- **Aurora Serverless v2** for intermittent/variable workloads instead of an always-on instance.
- Add **ElastiCache / DAX** to cut read load (and thus required DB size / request cost).

### Turn off what's idle

✅ Dev/test fleets don't need to run nights and weekends — use the **Instance Scheduler** (or
EventBridge + Lambda) to stop/start on a calendar. **Trusted Advisor** flags idle ELBs, RDS,
and unattached EBS; **Compute Optimizer** flags over-provisioning.

### Minimize data transfer

- Keep chatty components in the **same AZ** (free via private IP) where availability allows.
- Use **VPC Gateway Endpoints** for S3/DynamoDB — traffic stays on the AWS network and the
  endpoint itself is **free**, *and* you avoid **NAT Gateway** data-processing charges.
- Put **CloudFront** in front of S3/ALB to cut origin egress and offload requests.
- Keep traffic **in-region**; cross-region replication and reads cost per-GB.

⚠️ **NAT Gateway** has both an hourly charge *and* a per-GB data-processing charge — a common
hidden cost. Routing S3/DynamoDB through a NAT instead of a (free) Gateway Endpoint is a
classic waste pattern.

---

## 5. Consolidated Billing & Organizations

With **AWS Organizations**, all member accounts roll up to one **management (payer) account** —
**consolidated billing**. Beyond a single invoice, it unlocks real savings:

- ✅ **Volume / tiered discounts** — usage is *aggregated across all accounts*, so the whole org
  reaches higher-volume price tiers faster (e.g., S3 tiered pricing, data-transfer tiers).
- ✅ **Reserved Instance & Savings Plan sharing** — an unused RI/SP in one account
  automatically applies to matching usage in **another** account, maximizing utilization.
- ✅ One free tier shared, one place to apply **Cost Allocation Tags** and **SCPs**.

> **Key insight**: Consolidated billing's exam-relevant benefit is **shared volume discounts +
> RI/SP sharing across accounts** — not merely "one bill."

---

## 6. "How Do I Reduce Cost for X?" — Decision Points

| Scenario | Cheapest correct answer |
|----------|--------------------------|
| Steady 24/7 production EC2 | **Compute Savings Plan** (or RI), 1–3 yr |
| Fault-tolerant batch / CI workers | **Spot Instances** |
| Dev/test boxes idle nights & weekends | **Stop on a schedule** (Instance Scheduler) |
| Unknown S3 access pattern | **S3 Intelligent-Tiering** |
| Logs/backups rarely read, must keep for years | **Lifecycle → Glacier Deep Archive** |
| App makes heavy S3/DynamoDB calls from private subnets | **VPC Gateway Endpoint** (free, avoids NAT charges) |
| Serving large static/media files globally | **CloudFront** in front of origin |
| Need to alert before overspending | **AWS Budgets** (with forecasted threshold) |
| Need most granular billing data for BI | **CUR → S3 → Athena/Amazon Quick Sight** |
| Attribute spend per team/project | **Cost Allocation Tags** |
| "Is this instance the right size?" | **Compute Optimizer** |
| "Find idle/unused resources automatically" | **Trusted Advisor** cost checks |
| Estimate cost of a design before building | **Pricing Calculator** |
| Multiple accounts, want max RI/discount benefit | **Organizations consolidated billing** |

---

## 7. Enterprise Cost Allocation and Visibility

At organization scale, the goal is not merely a smaller invoice. Every material
cost needs an owner, a business purpose, and enough detail to explain variance.

### Build a queryable cost-data pipeline

Use **AWS Data Exports for Cost and Usage Reports (CUR 2.0)**, or a legacy CUR
where already deployed, to deliver detailed billing data to a central S3 bucket.
Catalog/query it with Glue and Athena and publish governed dashboards through
Amazon QuickSight or the organization's BI platform. Partition exports and
control retention because repeated wide Athena scans and long-lived detailed
reports also cost money.

Keep the raw export immutable and restrict it: billing line items can expose
account names, resource identifiers, tags, discounts, and commercial rates.
Reconcile internal dashboards to the payer invoice and version allocation SQL so
a finance result can be reproduced.

### Define allocation in layers

| Mechanism | What it solves | Limitation to design around |
|-----------|----------------|-----------------------------|
| **Accounts/OUs** | Strong ownership and environment/product boundary | Shared platform accounts still need allocation rules |
| Activated **cost allocation tags** | Resource-level project, owner, environment, or application attribution | Tags are not retroactive, some costs are untaggable, and misspellings create new values |
| **Tag policies** and IaC validation | Standardize allowed tag keys/values | A tag policy reports/enforces syntax; it does not allocate untaggable shared cost |
| **Cost Categories** | Rule-based business groupings across accounts, tags, services, and charges | Rule order and inherited values need testing and change control |
| **AWS Billing Conductor** | Pro forma rates and billing groups for showback/chargeback where custom pricing is needed | It changes the pro forma view, not what AWS invoices the payer |

Define how support, shared networking, observability, commitments, marketplace,
credits, refunds, and untagged cost are allocated. Publish both **showback** (who
caused cost) and, if required, **chargeback** (what internal price is assigned).
Do not hide shared-platform economics by forcing every line item to one team.

### Delegate without exposing the payer account

Use IAM Identity Center permission sets and fine-grained Billing and Cost
Management permissions for finance, FinOps, and workload owners. Keep root and
management-account administration separate from routine report access. Scope
views/exports where possible, log changes with CloudTrail, and require approval
for purchases, sharing preferences, budgets, and allocation-rule changes.

Configure **Cost Anomaly Detection** monitors by service, linked account, Cost
Category, or tag boundary and route alerts to an owner who can investigate. An
anomaly alert complements Budgets: Budgets compare against planned thresholds;
anomaly detection finds unusual changes relative to historical behavior.

For RIs and Savings Plans, choose organization/account-group sharing preferences
deliberately. Central sharing raises utilization, but the account receiving the
discount may not own the commitment. Report gross usage, amortized commitment
cost, effective savings, and the internal allocation method so teams are not
rewarded or penalized accidentally.

---

## 8. Run an Optimization Investigation

Treat each optimization as a measured change with an owner and rollback, not a
list of generic savings tips.

1. **Establish the business unit.** Use cost per order, tenant, build, GB
   processed, or another outcome alongside availability and latency SLOs.
2. **Find the driver.** Group CUR/Cost Explorer by account, service, usage type,
   Region, AZ, purchase option, and tag/Cost Category. Explain the change in
   units, rate, or both.
3. **Correlate utilization.** Use CloudWatch and Compute Optimizer for CPU,
   memory where the agent exposes it, network, storage IOPS/throughput, Lambda
   duration/memory, database load, cache hit rate, and scaling history. Billing
   data alone cannot prove a resource is oversized.
4. **Model options and risk.** Include implementation cost, migration window,
   contractual commitment, performance headroom, failure capacity, and exit
   path. Estimate monthly and unit-cost change under baseline, growth, and
   failure scenarios.
5. **Change safely and verify.** Canary or schedule the change, measure the same
   SLOs and cost unit, then keep, tune, or roll back it.

### Investigations the exam expects

- **Idle/overprovisioned**: find unattached volumes/IPs, old snapshots, empty load
  balancers, stopped-resource storage, low-use instances/databases, and min/max
  fleets that never scale. Preserve deliberate DR headroom and compliance
  retention rather than deleting everything with low utilization.
- **Data transfer**: separate internet egress, inter-Region, cross-AZ, NAT Gateway
  processing, Transit Gateway, PrivateLink, and service request bytes. A chatty
  cross-AZ path can charge on multiple legs. Consider S3/DynamoDB gateway
  endpoints, distributed per-AZ NAT, caching/compression, topology changes, and
  fewer payload bytes, then check whether the availability trade-off is valid.
- **Commitments**: measure coverage and utilization after rightsizing. Purchase
  only the durable baseline and compare Compute Savings Plans flexibility with
  narrower plans/RIs. A larger discount can lose money if a migration changes
  Region, family, architecture, service, or account during the term.

Cost is one constraint. Reducing Multi-AZ capacity, backup retention, log detail,
or security inspection can lower spend while violating reliability, recovery,
audit, or performance requirements. Record the accepted cross-pillar trade-off
and the metric/guardrail that keeps it valid.

---

## 9. Key Exam Points

- ✅ **Cloud = OpEx, pay-as-you-go**; the goal is matching supply to demand, not over-provisioning.
- ✅ **Data-transfer ranking**: in = free, same-AZ = free, cross-AZ = small charge both ways,
  cross-region = more, **internet egress = most**.
- ✅ **Budgets** = alert/act on a threshold (incl. forecast). **Cost Explorer** = visualize &
  forecast trends + recommendations. **CUR** = granular line items in S3 for custom queries.
- ✅ **Compute Optimizer** = right-sizing recommendations. **Trusted Advisor** = idle/underused
  resource checks. **Pricing Calculator** = estimate *before* deploying.
- ✅ **Savings Plans** > RIs for flexibility (cover EC2/Fargate/Lambda). **Spot** for
  interruption-tolerant work.
- ✅ **Cost Allocation Tags** attribute spend per team/project/env.
- ✅ **Consolidated billing** → aggregated volume discounts + RI/SP sharing across accounts.
- ✅ The **billing CloudWatch metric is in us-east-1 only** — create billing alarms there.
- ✅ **S3 Intelligent-Tiering** for unknown access; **lifecycle policies** to transition/expire.
- ✅ **Gateway Endpoints** are free and avoid NAT data-processing charges.
- ✅ CUR/Data Exports create the queryable billing source; Cost Categories, tags, accounts/OUs, and optional Billing Conductor pro forma rates create the allocation model.
- ✅ Cost Anomaly Detection finds unusual spend; Budgets track planned actual/forecast thresholds.
- ✅ Optimize from usage and business-unit metrics, then verify cost, performance, and reliability after the change.

---

## 10. Common Mistakes

- ❌ Confusing **Budgets** (forecast/alert/act) with **Cost Explorer** (analyze/visualize). If
  the question wants a *notification at a threshold*, it's Budgets.
- ❌ Reaching for **CUR** when the question just wants a chart — CUR is the *raw data export*,
  Cost Explorer is the dashboard.
- ❌ Buying **Reserved Instances before right-sizing** — you lock in the wrong size.
- ❌ Creating a billing alarm in the wrong region — the metric only exists in **us-east-1**.
- ❌ Routing S3/DynamoDB traffic through a **NAT Gateway** instead of a free **Gateway Endpoint**.
- ❌ Forgetting that **cross-AZ** traffic is charged in *both* directions, and that **snapshots,
  old object versions, and incomplete multipart uploads** keep billing after you "delete" things.
- ❌ Assuming consolidated billing is "just one invoice" — its real value is **shared discounts
  and RI/SP coverage** across accounts.
- ❌ Treating tags as a complete allocation strategy without governance or rules for shared, untaggable, credit, support, and commitment costs.
- ❌ Buying commitments from last month's bill before rightsizing or checking the migration/modernization roadmap.
- ❌ Removing failure headroom, backups, logs, or inspection because utilization looks low, without revalidating SLOs and compliance.

---

**Next**: [02_well_architected_framework.md — The Well-Architected Framework & Its Six Pillars](02_well_architected_framework.md)
