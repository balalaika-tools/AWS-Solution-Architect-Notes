# Decoupled Workers: SQS-Driven Auto Scaling

> **Who this is for**: Engineers prepping for SAA-C03 who need a fleet of worker
> EC2 instances to process a queue and scale on **how much work is waiting**, not
> on CPU. Assumes you've read [SQS](../11_messaging/02_sqs.md) and
> [Auto Scaling Groups](../07_ha_scaling/03_auto_scaling_groups.md).

> **Key insight**: When work arrives via a queue, the right scaling signal is the
> **backlog** (messages waiting), not CPU. Scale the ASG on
> `ApproximateNumberOfMessagesVisible` or, better, a custom **backlog-per-instance**
> metric so each worker has a bounded amount of work.

---

## 1. The Pattern

```
  producers              decoupling buffer            worker fleet (ASG)
 ┌──────────┐            ┌──────────────────┐         ┌──────────────────┐
 │ Web tier │ ──send──▶  │  SQS queue        │ ◀──poll │ EC2 worker #1     │
 │ Lambda   │ ──send──▶  │  (Standard)       │ ──msg──▶│ EC2 worker #2     │
 │ API      │            │                   │         │ EC2 worker #N     │
 └──────────┘            └────────┬─────────┘         └──────────────────┘
                                  │ depth = ApproximateNumberOfMessagesVisible
                                  ▼
                         CloudWatch metric  ──▶  Target Tracking policy
                                                 scales ASG DesiredCapacity
                                  │
                                  ▼ (poison messages after maxReceiveCount)
                         ┌──────────────────┐
                         │  Dead-Letter Queue│
                         └──────────────────┘
```

The queue absorbs bursts: producers never block on slow workers, and a worker
crash just means its in-flight message reappears for another worker after the
**visibility timeout**.

---

## 2. Why Not CPU-Based Scaling?

❌ **CPU is a lagging, weak proxy for queue work.** Workers may be I/O-bound
(low CPU even with a huge backlog), or CPU may spike on a single heavy message
while the queue is nearly empty. Either way the fleet size doesn't track the
actual amount of pending work.

✅ **Backlog scales the fleet to the work.** Driving the ASG off
`ApproximateNumberOfMessagesVisible` (or backlog-per-instance) means more waiting
messages ⇒ more instances, and an empty queue ⇒ scale to the minimum. It reacts
to demand directly.

| Scaling signal                         | Tracks queue work? | Notes                          |
|----------------------------------------|:------------------:|--------------------------------|
| `CPUUtilization`                        | ❌ indirect        | misleads for I/O-bound workers |
| `ApproximateNumberOfMessagesVisible`    | ✅ yes             | total backlog                  |
| **Backlog per instance** (custom)       | ✅ best            | bounds work per worker         |

---

## 3. Backlog-Per-Instance — the Recommended Metric

AWS's recommended approach: decide an **acceptable backlog per instance** (e.g.
each worker should have at most 100 queued messages), then track:

```
backlogPerInstance = ApproximateNumberOfMessagesVisible / runningInstanceCount
```

Publish it as a custom metric and target-track it at 100. The ASG keeps roughly
`messages / 100` instances running.

```bash
# Compute and publish the custom metric (run on a schedule, e.g. EventBridge → Lambda)
VISIBLE=$(aws sqs get-queue-attributes --queue-url "$Q_URL" \
  --attribute-names ApproximateNumberOfMessagesVisible \
  --query 'Attributes.ApproximateNumberOfMessagesVisible' --output text)
INSTANCES=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names worker-asg \
  --query 'AutoScalingGroups[0].Instances | length(@)' --output text)
BPI=$(( INSTANCES > 0 ? VISIBLE / INSTANCES : VISIBLE ))

aws cloudwatch put-metric-data \
  --namespace "Worker/SQS" \
  --metric-name BacklogPerInstance \
  --value "$BPI"
```

---

## 4. The Scaling Policy (Target Tracking)

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name worker-asg \
  --policy-name backlog-per-instance-100 \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
      "TargetValue": 100.0,
      "CustomizedMetricSpecification": {
        "MetricName": "BacklogPerInstance",
        "Namespace": "Worker/SQS",
        "Statistic": "Average"
      },
      "ScaleInCooldown": 120,
      "ScaleOutCooldown": 60,
      "DisableScaleIn": false
  }'
```

💡 If you don't need per-instance bounding, you can target-track on a CloudWatch
**math expression** of `ApproximateNumberOfMessagesVisible` directly — but
backlog-per-instance is the cleaner answer because it keeps each worker's load
bounded as the fleet grows.

---

## 5. Visibility Timeout Must Match Processing Time

When a worker receives a message, SQS hides it for the **visibility timeout**.
If the worker finishes and deletes it within that window, all good. If the
timeout expires first, SQS assumes the worker died and **redelivers** the
message — causing **duplicate processing**.

```bash
# Set visibility timeout >= worst-case processing time (e.g. jobs take up to 4 min)
aws sqs set-queue-attributes --queue-url "$Q_URL" \
  --attributes VisibilityTimeout=300        # 5 min, with headroom

# Attach a dead-letter queue so poison messages don't loop forever
aws sqs set-queue-attributes --queue-url "$Q_URL" \
  --attributes '{
    "RedrivePolicy":"{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:111122223333:worker-dlq\",\"maxReceiveCount\":\"5\"}"
  }'
```

⚠️ For jobs whose duration varies, call **`ChangeMessageVisibility`** to extend
the timeout while still working (a heartbeat), rather than setting one huge fixed
timeout that delays redelivery of genuinely-failed messages.

---

## 6. Troubleshooting

**Scaling reacts too slowly to a spike**

- Long polling / metric publish interval is coarse. Publish the backlog metric
  more frequently and lower **ScaleOutCooldown**. Note SQS queue-depth metrics
  are emitted ~once a minute, so sub-minute reaction needs a custom high-frequency
  metric or step scaling.
- `ScaleOutCooldown` too long blocks consecutive scale-outs during a fast ramp.

**Messages get processed twice (duplicates)**

- **Visibility timeout shorter than processing time** — SQS redelivered while the
  first worker was still working. Raise `VisibilityTimeout` to exceed worst-case
  duration, or heartbeat with `ChangeMessageVisibility`. Also make processing
  **idempotent** (Standard queues are at-least-once by design).

**Fleet never scales in to zero / minimum**

- ASG `MinSize` floor, or the worker doesn't `DeleteMessage` after success so the
  backlog never clears. Ensure successful jobs are deleted, and confirm DisableScaleIn is false.

**Same message keeps failing and clogs the queue**

- A poison message with no DLQ loops forever. Configure a **dead-letter queue**
  with `maxReceiveCount` so it's parked after N failed receives.

**Backlog-per-instance metric divides by zero / spikes at startup**

- When `runningInstanceCount` is 0, guard the division (fall back to raw backlog)
  so the metric doesn't break the policy during cold start.

---

## Key Exam Points

- Decouple producers from workers with **SQS**; scale the **ASG on queue depth**,
  not CPU.
- Best metric = **backlog per instance** (`ApproximateNumberOfMessagesVisible /
  instances`) with **target tracking**; bounds work per worker.
- **CPU-based scaling for queue workers is the wrong answer** — it doesn't track
  pending work, especially for I/O-bound jobs.
- **Visibility timeout ≥ processing time**, or messages get reprocessed; use
  `ChangeMessageVisibility` to extend, and design **idempotent** consumers.
- Add a **dead-letter queue** (`maxReceiveCount`) for poison messages.
- Workers must **delete** messages after success, or the backlog never drains.

---

**Next**: [16_cloudwatch_alarm_scaling.md — CloudWatch Alarms Driving Scaling Actions](16_cloudwatch_alarm_scaling.md)
