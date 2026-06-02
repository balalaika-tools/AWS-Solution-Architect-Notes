# Auto Scaling Groups

> **Who this is for**: Engineers prepping for SAA-C03 who can stand up a load balancer and now want capacity to adjust itself. Networking-light readers welcome. Assumes you've read [02_elb_alb_nlb_gwlb.md](02_elb_alb_nlb_gwlb.md) (target groups, health checks) — an ASG registers its instances into those target groups.

---

## 1. What an ASG Is and Why

A **load balancer spreads traffic** across instances. But it doesn't create or destroy them — if all your instances are overloaded, the LB just spreads the pain. An **Auto Scaling Group (ASG)** is the piece that **adds and removes EC2 instances automatically** to match demand and replace failures.

```
   CloudWatch metric (CPU, requests, queue depth)
                 │ crosses threshold
                 ▼
   ┌──────────── Auto Scaling Group ─────────────┐
   │   desired = 4   (min = 2, max = 8)          │
   │                                             │
   │   ┌──┐  ┌──┐  ┌──┐  ┌──┐   ← launches/      │
   │   │EC2│ │EC2│ │EC2│ │EC2│     terminates    │
   │   └──┘  └──┘  └──┘  └──┘     to hit desired │
   └───────────────┬─────────────────────────────┘
                   │ registers instances into
                   ▼
              ELB target group  ──▶ traffic
```

An ASG gives you three things:

| Benefit | How |
|---------|-----|
| **Elasticity** | Scale out under load, scale in when idle — pay only for what you need |
| **Self-healing** | If an instance fails a health check, the ASG terminates it and launches a replacement to maintain `desired` |
| **High availability** | Spreads instances across **multiple AZs**; if an AZ fails, it relaunches capacity in the survivors |

> **Key insight**: An ASG maintains a **target count of healthy instances** across AZs. Everything else — scaling policies, health checks, lifecycle hooks — is in service of keeping the actual count at `desired` and adjusting `desired` to match demand.

---

## 2. Launch Templates vs Launch Configurations

An ASG needs a recipe for "what does a new instance look like?" — AMI, instance type, security groups, key pair, user data, IAM role. That recipe is a **launch template** (modern) or a **launch configuration** (legacy).

```
   Launch Template  ──▶  ASG  ──▶  launches identical instances
   (AMI, type, SG,        │
    user data, role)      └─ "give me 4 of these across 3 AZs"
```

| | Launch Configuration (legacy) | Launch Template (modern) |
|---|---|---|
| **Versioning** | ❌ none — immutable, recreate to change | ✅ versioned, edit and roll forward/back |
| **Mixed instance types / Spot+On-Demand** | ❌ | ✅ |
| **Newer features (T2/T3 unlimited, placement, etc.)** | limited | ✅ full |
| **AWS recommendation** | deprecated | **use this** |

> **Rule**: Always choose **launch templates** for new ASGs. Launch configurations are legacy and don't support mixed instances or Spot+On-Demand blends. If a question mentions combining Spot and On-Demand or versioning the config, the answer is a launch template.

---

## 3. Desired, Minimum, Maximum Capacity

Three numbers define an ASG's size:

```
   max     = 8   ──────────────────────────  ceiling (never exceed)
                          ▲ scale out
   desired = 4   ──────── ● current target ──
                          ▼ scale in
   min     = 2   ──────────────────────────  floor (never go below)
```

| Setting | Meaning |
|---------|---------|
| **Minimum** | The ASG never has fewer running instances than this (your availability floor) |
| **Desired** | The number the ASG actively tries to maintain right now |
| **Maximum** | The ceiling scaling can grow to (your cost guardrail) |

Scaling policies adjust **desired** up or down between **min** and **max**. The ASG then launches or terminates instances to match.

💡 Set `min ≥ 2` across **≥2 AZs** for production so a single instance or AZ failure never takes the service to zero.

---

## 4. Scaling Policies

This is the most exam-heavy part. There are several ways to change `desired`.

### Dynamic scaling — reacts to metrics

| Policy | How it works | When to use |
|--------|--------------|-------------|
| **Target tracking** | Pick a metric + target value (e.g., "keep avg CPU at 50%"); AWS adds/removes instances to hold it | **Default choice** — simplest; the exam's go-to |
| **Step scaling** | Define steps: e.g., +1 if CPU 50–70%, +3 if CPU >70% | When you need graduated, larger responses to bigger spikes |
| **Simple scaling** | One adjustment per alarm, then wait out a cooldown | Legacy; superseded by step scaling |

```
   TARGET TRACKING:  "keep CPU at 50%"
   CPU climbs to 80% ──▶ ASG adds instances until CPU settles near 50%
   CPU drops to 20%  ──▶ ASG removes instances until CPU rises toward 50%
```

### Scheduled scaling — reacts to time

Change `min`/`max`/`desired` on a schedule when load is **predictable**. Example: scale to 10 instances every weekday at 08:00, back to 2 at 20:00.

### Predictive scaling — reacts to forecasts

Uses **machine learning** on historical CloudWatch data to **forecast** load and scale **ahead of** demand (provisioning before the spike, not during it). Best for cyclical, recurring patterns. Often combined with dynamic scaling as a safety net.

| Policy | Trigger | Best for |
|--------|---------|----------|
| **Target tracking** | Metric vs target | Most workloads (simplest) |
| **Step scaling** | Alarm thresholds, graduated | Spiky load needing sized responses |
| **Simple scaling** | Single alarm + cooldown | Legacy only |
| **Scheduled** | Time of day/week | Known, recurring schedules |
| **Predictive** | ML forecast | Cyclical patterns; pre-warming for spikes |

> **Key insight**: **Target tracking** is the default exam answer ("keep CPU at X%"). **Scheduled** is for *known* time-based patterns. **Predictive** is for *forecastable* recurring patterns where you must provision *before* the spike. Match the keyword in the question.

💡 **"Auto Scaling Group" ≠ all AWS auto scaling.** An ASG scales **EC2 instances**. The same three policy types (target tracking, step, scheduled) are also offered by **Application Auto Scaling**, a *separate* service that scales **ECS task count**, DynamoDB throughput, Aurora replicas, and more. Same vocabulary, different target — don't answer "ASG" when a question is about scaling ECS tasks. See [ECS Service Auto Scaling](../09_containers/02_ecs_ecr_fargate_eks.md).

---

## 5. Cooldowns

After a scaling action, a **cooldown period** (default **300 s** for simple scaling) blocks further *simple* scaling actions so the new instances have time to boot, register, and start absorbing load — preventing wild oscillation (thrashing).

```
   scale out ──▶ [ cooldown 300s: ignore new simple-scaling triggers ] ──▶ ready again
```

⚠️ Cooldowns apply mainly to **simple scaling**. **Target tracking** and **step scaling** use their own internal logic (and an *instance warmup* setting) and generally don't need a classic cooldown. The exam contrasts these: if instances are flapping under simple scaling, increase the cooldown.

---

## 6. Health Checks: EC2 vs ELB

The ASG decides an instance is unhealthy (and replaces it) based on one or both of:

```
   EC2 health check (default):  Is the instance running &
                                passing EC2 status checks?

   ELB health check (opt-in):   Is the instance passing the
                                LOAD BALANCER's health check?
                                (e.g., HTTP 200 on /health)
```

| | EC2 health check | ELB health check |
|---|---|---|
| **Checks** | Instance status (hardware/OS reachable) | Application responds correctly (target-group probe) |
| **Catches** | Crashed/stopped/impaired instances | A *running* instance whose **app is broken** |
| **Default** | ✅ on | ❌ must enable |

> **Rule**: If an ASG sits behind a load balancer, **enable ELB health checks**. With only EC2 checks, an instance whose app has crashed but whose VM is still "running" stays in service and keeps failing requests. The ELB health check catches the broken app and lets the ASG replace it. This is a very common exam scenario.

💡 The ASG uses the **same health check** configured on the target group (file 01, §5). EC2 + ELB health checks are complementary — turn on both.

---

## 7. Instance Refresh

**Instance refresh** rolls out a new launch-template version (e.g., a patched AMI) across the ASG by **gradually replacing** instances, respecting a **minimum healthy percentage** so the service stays up.

```
   New AMI in launch template ──▶ start instance refresh
   replace a batch ──▶ wait for healthy ──▶ replace next batch ──▶ … done
   (keeps ≥ min healthy % in service throughout)
```

✅ Use instance refresh for zero-downtime AMI updates / config changes across the whole group.

⚠️ Set the **minimum healthy percentage** and **instance warmup** sensibly, or a refresh can over-replace and dip capacity below what the load needs.

---

## 8. Lifecycle Hooks

**Lifecycle hooks** pause an instance at launch or termination so you can run custom actions before it goes into service or is destroyed.

```
   Pending ──[ launch hook: pause ]──▶ run bootstrap (pull data, register
                                       with config mgmt) ──▶ InService

   Terminating ──[ terminate hook: pause ]──▶ drain, upload logs,
                                              deregister ──▶ Terminated
```

| Hook | Pauses at | Typical use |
|------|-----------|-------------|
| **Launch hook** | Before `InService` | Warm caches, install software, register the instance |
| **Terminate hook** | Before termination | Flush logs, drain connections, deregister cleanly |

💡 Lifecycle hooks are how you guarantee setup/cleanup work completes despite the ASG launching and terminating instances on its own schedule.

---

## 9. Termination Policies

When scaling **in**, the ASG must choose *which* instance to kill. The **termination policy** decides.

| Policy | Picks the instance that… |
|--------|--------------------------|
| **Default** | Balances across AZs first, then oldest launch template/config, then closest to the next billing hour |
| **OldestInstance** | Has been running longest (good for rolling to newer config) |
| **NewestInstance** | Was launched most recently |
| **OldestLaunchTemplate** | Uses the oldest template version |
| **ClosestToNextInstanceHour** | Minimizes wasted partial-hour cost (legacy billing) |

> **Key insight**: The default policy first keeps **AZs balanced** — it won't drain one AZ while another is full. Only override when you have a specific need (e.g., always retire the oldest config first).

---

## 10. How It All Wires Together

```
  CloudWatch Alarm (e.g., avg CPU > 70%)
        │ triggers scaling policy
        ▼
  ┌──────────────── Auto Scaling Group ───────────────┐
  │  Launch Template → launches instances             │
  │  across AZ-a, AZ-b, AZ-c                          │
  │  health checks: EC2 + ELB                         │
  └───────────────────────┬───────────────────────────┘
                          │ auto-registers new instances
                          ▼
                  ELB Target Group
                          │
                          ▼
                  Application Load Balancer ──▶ clients
```

The exam loves this end-to-end chain: **CloudWatch alarm → scaling policy → ASG changes desired → launches from template → registers into the ALB target group → traffic flows.** See the worked example: [CloudWatch Alarm Triggering Scaling](../18_practical_examples/16_cloudwatch_alarm_scaling.md).

---

## 11. Scaling vs Multi-AZ Distribution

These are two **different** goals that an ASG happens to serve at once. Don't conflate them.

| | Scaling | Multi-AZ distribution |
|---|---|---|
| **Goal** | Handle changing **load** | Survive an **AZ failure** |
| **Mechanism** | Change `desired` up/down | Spread instances across ≥2 AZs |
| **Without it** | Over/under-provisioned | Outage when one AZ fails |
| **ASG setting** | Scaling policies | Multiple subnets (one per AZ) |

⚠️ An ASG with **max = 1 in a single AZ** scales nothing and survives nothing. Real HA needs **≥2 AZs** *and* room to grow (`max > min`).

💡 A non-LB pattern: an ASG can scale on **SQS queue depth** (a backlog of messages) rather than CPU — workers scale out when the queue grows. See [SQS with Auto Scaling Group](../18_practical_examples/15_sqs_with_asg.md).

---

## 12. Key Exam Points

- An ASG **maintains `desired` healthy instances across AZs**, self-heals failures, and scales with demand.
- Use **launch templates** (versioned, support Spot+On-Demand mix), not launch configurations.
- **Target tracking** is the default scaling policy ("keep CPU at X%"). **Scheduled** = known times. **Predictive** = ML forecast, scales *ahead* of demand.
- **Enable ELB health checks** when behind a load balancer — EC2 checks alone miss a crashed app on a running VM.
- **Cooldowns** prevent thrash (default 300 s, mainly for simple scaling).
- **Instance refresh** rolls out new AMIs with a minimum-healthy-percentage guardrail.
- **Lifecycle hooks** pause launch/terminate for custom setup/cleanup.
- Distribute across **≥2 AZs** for availability; scaling and multi-AZ are separate concerns.
- An ASG can scale on **SQS queue depth**, not just CPU.

---

## 13. Common Mistakes

- **❌ Using only EC2 health checks behind a load balancer.** A hung app on a "running" instance stays in service. Enable **ELB health checks**.
- **❌ Single-AZ ASG.** No AZ-failure protection — always span ≥2 AZs.
- **❌ Setting `min = max`.** The group can't scale; you've built a fixed fleet.
- **❌ Reaching for step/predictive when target tracking suffices.** Target tracking is the simplest correct answer for "keep utilization steady."
- **❌ Launch configuration for a Spot+On-Demand mix.** Only **launch templates** support mixed instances.
- **❌ Cooldown too short under simple scaling**, causing instances to flap in and out.
- **❌ Expecting an ASG to load-balance traffic.** It manages *count*; the **ELB** distributes requests. They work together.

---

## 14. Limits & Defaults

| Item | Value |
|------|-------|
| Default simple-scaling cooldown | 300 s |
| Health check (default) | EC2; enable ELB when behind an LB |
| Recommended AZ spread | ≥2 AZs |
| Launch templates | Versioned; support Spot + On-Demand |
| Default termination policy | Balance AZs → oldest template → closest to billing hour |
| Capacity model | min ≤ desired ≤ max |
| Instance refresh | Honors minimum healthy percentage + warmup |

---

**Next**: [01_dns_fundamentals.md — DNS Fundamentals](../08_dns_edge/01_dns_fundamentals.md)
