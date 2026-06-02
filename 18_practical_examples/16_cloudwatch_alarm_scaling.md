# CloudWatch Alarm → Auto Scaling: A CPU-Driven Scaling Walkthrough

> **Who this is for**: Engineers prepping for SAA-C03 who know what an
> [Auto Scaling Group](../07_ha_scaling/03_auto_scaling_groups.md) and a
> [CloudWatch metric](../12_monitoring/01_cloudwatch.md) are individually, but haven't wired
> a metric → alarm → scaling policy end-to-end and seen exactly which knobs control when a
> scale-out fires. We build one concrete CPU-based example and contrast the three policy types.

---

## 1. The Scenario

A stateless web tier runs behind an ALB in an Auto Scaling Group named `web-asg`
(min 2, desired 2, max 10). Traffic is spiky. We want the fleet to grow when average CPU
across the group climbs and shrink when it falls — without anyone touching the console.

The chain has three distinct objects, and the exam loves to test where one ends and the
next begins:

```
   ┌──────────────┐   publishes    ┌────────────────┐   state=ALARM   ┌──────────────────┐
   │ EC2 instances │──CPUUtilization──▶│ CloudWatch     │───triggers────▶│ Scaling policy   │
   │ in web-asg    │   (per-instance) │ metric + ALARM │                │ (on the ASG)     │
   └──────────────┘   aggregated by  └────────┬───────┘                └─────────┬────────┘
                      AutoScalingGroupName     │                                  │
                                               │ optional                         │ adjusts
                                               ▼                                  ▼
                                         ┌──────────┐                      ┌──────────────┐
                                         │   SNS    │ ◀── alarm action ──  │  DesiredCap  │
                                         │  topic   │                      │  2 → 4 …     │
                                         └──────────┘                      └──────────────┘
```

> **Key insight**: The **alarm** watches a metric and flips state. The **scaling policy**
> lives on the ASG and decides *how many* instances to add or remove. They are separate
> resources joined by `AlarmActions`. With **target tracking**, CloudWatch creates and manages
> the alarms *for you* — you never author them by hand.

---

## 2. The Metric Being Watched

`CPUUtilization` in the `AWS/EC2` namespace, aggregated across the group via the
`AutoScalingGroupName` dimension. That single series is the average CPU of all healthy
instances — the right thing to scale on (a per-instance dimension would let one hot box
trigger the whole group).

```
Namespace:   AWS/EC2
Metric:      CPUUtilization
Dimension:   AutoScalingGroupName = web-asg   ← group-wide average, not per instance
Statistic:   Average
Period:      60 s
```

⚠️ Default EC2 metrics are **5-minute** (basic monitoring). To react in 1-minute windows you
must enable **detailed monitoring** on the instances (extra cost). An alarm with a 60 s period
on a basic-monitoring instance sits in `INSUFFICIENT_DATA` because no datapoint arrives each
minute.

---

## 3. The Alarm Definition (Step / Simple Scaling)

A CloudWatch alarm is defined by **metric + threshold + period + evaluation periods +
datapoints-to-alarm**. Those last three control sensitivity and noise immunity.

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name web-asg-cpu-high \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=web-asg \
  --statistic Average \
  --period 60 \                       # each datapoint covers 60 seconds
  --evaluation-periods 3 \            # look at the last 3 datapoints (3 min window)
  --datapoints-to-alarm 2 \           # 2 of those 3 must breach → "M of N"
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions \
     arn:aws:autoscaling:us-east-1:111122223333:scalingPolicy:abcd...:policyName/scale-out \
     arn:aws:sns:us-east-1:111122223333:ops-alerts
```

| Field | Value | Meaning |
|-------|-------|---------|
| `threshold` + operator | `> 70` | Breaching condition |
| `period` | `60 s` | Width of one datapoint |
| `evaluation-periods` | `3` | How many recent datapoints to inspect |
| `datapoints-to-alarm` | `2` | How many of those must breach to alarm ("**2 of 3**") |
| `treat-missing-data` | `notBreaching` | Gaps don't push it to ALARM |

> **Rule**: `datapoints-to-alarm ≤ evaluation-periods`. "2 of 3" rides out a single noisy
> spike but still reacts within ~3 minutes. "3 of 3" is slower but calmer; "1 of 1" is
> twitchy and prone to flapping.

The alarm has three states: `OK` (below threshold), `ALARM` (breaching condition met),
`INSUFFICIENT_DATA` (not enough datapoints yet).

---

## 4. The Scaling Policies It Triggers

The alarm's `--alarm-actions` points at a **scaling policy ARN**. Here's a **step scaling**
policy that adds more instances the worse the breach gets:

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name scale-out \
  --policy-type StepScaling \
  --adjustment-type ChangeInCapacity \
  --estimated-instance-warmup 120 \
  --step-adjustments \
     MetricIntervalLowerBound=0,MetricIntervalUpperBound=20,ScalingAdjustment=1 \
     MetricIntervalLowerBound=20,ScalingAdjustment=3
```

Read the step bounds **relative to the alarm threshold (70)**:

| Measured CPU | Interval above threshold | Action |
|--------------|--------------------------|--------|
| 70–90% | 0 to +20 | add **1** instance |
| > 90% | +20 and up | add **3** instances |

`estimated-instance-warmup` (120 s) tells the ASG to ignore the new instances' metrics
until they've booted and started absorbing load — the step-scaling equivalent of a cooldown.

A matching `scale-in` policy is driven by a separate `web-asg-cpu-low` alarm
(`< 30` for 3 of 3 datapoints, `ChangeInCapacity = -1`).

---

## 5. Scale-Out and Scale-In in Motion

```
CPU%
 95 ┤                  ╭─────╮   web-asg-cpu-high → ALARM (2 of 3 > 70)
 70 ┤- - - - - - -╭────╯     ╰──╮ - - - - - - - - - - - - - - - threshold (high)
 50 ┤        ╭────╯             ╰────╮
 30 ┤- - ────╯                        ╰────────── threshold (low) → scale-in
    └────┬───────┬──────┬───────┬──────┬──────────▶ time
 desired 2       2      4       7      7→6→5→...

  t1: CPU climbs, 2 of last 3 datapoints > 70  → ALARM → step policy adds instances
  t2: CPU > 90 briefly → higher step → +3 in one go
  t3: warmup window: new instances' metrics ignored 120 s
  t4: CPU drops < 30 for 3 of 3 → scale-in alarm → remove 1 at a time
```

💡 Scale **out fast, in slow**. Aggressive scale-out (lower `datapoints-to-alarm`, bigger
steps) protects availability; conservative scale-in (3-of-3, `-1` at a time) avoids removing
capacity you're about to need again.

---

## 6. The SNS Notification Option

An alarm can fan out to **multiple actions**. Above, `web-asg-cpu-high` lists *both* the
scaling-policy ARN *and* an SNS topic ARN, so the same breach scales the fleet **and** emails
on-call. Separately, the ASG itself can publish **lifecycle notifications** (instance launch/
terminate) to SNS:

```bash
aws autoscaling put-notification-configuration \
  --auto-scaling-group-name web-asg \
  --topic-arn arn:aws:sns:us-east-1:111122223333:ops-alerts \
  --notification-types \
     autoscaling:EC2_INSTANCE_LAUNCH \
     autoscaling:EC2_INSTANCE_TERMINATE
```

> **Exam distinction**: the **alarm action → SNS** tells you *the metric breached*; the
> **ASG notification → SNS** tells you *an instance actually launched/terminated*. Different
> publishers, different meaning.

---

## 7. Target Tracking vs Step vs Simple

This is the most-tested comparison in this topic.

| | **Target tracking** | **Step scaling** | **Simple scaling** |
|---|---|---|---|
| You specify | A **target value** (e.g. CPU = 50%) | Alarm + tiered steps | Alarm + one adjustment |
| Alarms | CloudWatch **auto-creates & manages** them | You create them | You create them |
| Reaction | Adds/removes to hold the target | Bigger breach → bigger step | One fixed change, then cooldown |
| Multiple steps | No (one target) | Yes (graduated response) | No |
| Cooldown / warmup | Instance warmup | Instance warmup | Classic **cooldown** (default 300 s) |
| Use when | "Keep utilization steady" (default choice) | Response must scale with severity | Legacy only |

> **Rule**: On the exam, **"maintain ~X% CPU"** or **"keep the metric at a target"** → **target
> tracking** (simplest correct answer). **"Add more the worse it gets"** → step scaling.
> **Simple scaling** is the legacy single-action-then-cooldown model — almost never the right
> answer for new designs.

A target-tracking policy is a one-liner — no alarm authoring at all:

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name cpu-target-50 \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
      "PredefinedMetricSpecification": {"PredefinedMetricType": "ASGAverageCPUUtilization"},
      "TargetValue": 50.0
  }'
```

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Instances flap in/out every few minutes | High and low thresholds too close; `datapoints-to-alarm` too low; cooldown too short | Widen the gap (e.g. >70 / <30), use "2 of 3", lengthen cooldown/warmup |
| Scale-out fires too late | `period` 300 s (basic monitoring) or large `evaluation-periods` | Enable **detailed monitoring** (1-min), reduce evaluation periods |
| Alarm stuck in `INSUFFICIENT_DATA` | No datapoints (basic monitoring + 60 s period), or wrong dimension/namespace | Match period to publishing rate; verify `AutoScalingGroupName` dimension |
| Scaled out but new instances do nothing for 2 min | `estimated-instance-warmup` excluding them from the metric | Expected — tune warmup to real boot time |
| ASG immediately removes new instances | Aggressive scale-in alarm; no warmup | Add warmup; make scale-in 3-of-3 and `-1` at a time |
| Capacity won't grow past N | Hit `MaxSize` | Raise `--max-size` |

⚠️ **Flapping** is the classic distractor. Root cause is almost always *thresholds too close
together* or *cooldown/warmup too short*. Under **simple scaling**, the fix the exam wants is
"increase the cooldown."

✅ For a brand-new "keep CPU around 50%" requirement, reach for **target tracking** and stop
hand-writing alarms.

❌ Don't scale on a **per-instance** `InstanceId` dimension — one hot instance shouldn't drive
the whole group. Aggregate on `AutoScalingGroupName`.

---

## Key Exam Points

- An alarm = **metric + threshold + period + evaluation-periods + datapoints-to-alarm**;
  `datapoints-to-alarm ≤ evaluation-periods` gives "M of N" noise tolerance.
- Alarm states: `OK`, `ALARM`, `INSUFFICIENT_DATA`. `treat-missing-data` controls how gaps
  are scored.
- **Target tracking** auto-manages its own alarms; you only set a `TargetValue`. It's the
  default correct answer for "hold a metric steady."
- **Step scaling** = graduated response (bigger breach → bigger adjustment). **Simple
  scaling** = one action + cooldown (legacy).
- Cooldown (default **300 s**) is mainly a **simple-scaling** concept; target/step use
  **instance warmup**.
- A single alarm can drive **both** a scaling policy and an SNS notification via
  `AlarmActions`. ASG lifecycle notifications to SNS are a separate mechanism.
- Basic monitoring = 5-min datapoints; **detailed monitoring** = 1-min (needed for fast
  reaction).

---

**Next**: [17_cloudtrail_vs_cloudwatch_vs_config.md — Which Tool Answers Which Question](17_cloudtrail_vs_cloudwatch_vs_config.md)
