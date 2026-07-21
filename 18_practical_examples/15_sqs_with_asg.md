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
  --query 'AutoScalingGroups[0].Instances[?LifecycleState==`InService`] | length(@)' \
  --output text)
BPI=$(awk -v visible="$VISIBLE" -v instances="$INSTANCES" \
  'BEGIN { print instances > 0 ? visible / instances : visible }')

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

## 6. Production Worker Lifecycle

Queue depth gets instances running; it does not make message processing correct. The worker must
also survive duplicate delivery, a slow job, scale-in, poison input, and a Regional recovery.

### Make the side effect idempotent

SQS Standard queues provide **at-least-once delivery**. A duplicate can arrive even when the
visibility timeout was long enough, and a worker can finish the side effect but crash before
`DeleteMessage`. Treat every receive as a retry.

A practical handler uses a stable producer-supplied `job_id` as an idempotency key:

```text
ReceiveMessage (long poll)
  → validate schema/version and job_id
  → conditional PutItem job_id, state=PROCESSING, lease_owner=<worker>
       condition: attribute_not_exists(job_id)
  → perform or resume the business operation using job_id as its idempotency token
  → transactionally record COMPLETED/result where possible
  → DeleteMessage only after the durable commit succeeds
```

If the idempotency record already says `COMPLETED`, return the stored result and delete the
duplicate. If it says `PROCESSING`, use a bounded lease/attempt policy rather than allowing two
workers to perform the same external side effect. DynamoDB conditional writes can protect an AWS
state transition; a payment, email, or third-party call also needs the same idempotency key at the
downstream system. FIFO deduplication reduces duplicates in its deduplication window but does not
replace idempotent consumers.

### Treat visibility as a renewable processing lease

Set the initial visibility timeout above normal processing time, then heartbeat with
`ChangeMessageVisibility` before it expires. Record the receive time and impose a maximum job
duration: endlessly extending a stuck job hides it forever. If shutdown or the maximum duration
arrives, stop extending the message and let it be retried. Delete only with the current receipt
handle; an old handle from an earlier receive cannot safely complete the current attempt.

Monitor `ApproximateNumberOfMessagesNotVisible` as well as visible backlog. A large in-flight
count with no completion rate usually means hung workers or an overly long visibility timeout.

### Drain before ASG scale-in or Spot interruption

Use an ASG **termination lifecycle hook** for normal scale-in:

1. The hook places the instance in `Terminating:Wait` and emits an EventBridge notification.
2. A drain agent marks the worker unavailable and stops calling `ReceiveMessage`.
3. It completes its current job, continuing visibility heartbeats, then calls
   `CompleteLifecycleAction`.
4. If it cannot finish before the bounded drain deadline, it abandons the job so SQS can
   redeliver it and completes termination.

While a worker owns a long job, instance scale-in protection can keep normal ASG scale-in from
selecting it; clear the protection immediately after the job or the group may become unable to
scale in. Lifecycle hooks and protection do not guarantee unlimited time and do not prevent an
EC2 failure.

For Spot capacity, watch the instance interruption/rebalance signal, stop receiving immediately,
and abandon or checkpoint work that cannot finish in the remaining notice window. Enable
**Capacity Rebalancing** so the ASG can launch replacement Spot capacity when risk rises. Use a
mixed-instances policy with several compatible instance types, capacity-optimized allocation,
and an On-Demand base/percentage large enough to meet the workload's minimum drain rate. Verify
that every instance type has the same architecture, AMI support, storage, and per-worker
throughput assumptions.

### Isolate poison messages and operate the DLQ

Reject malformed schema/version failures quickly instead of spending the entire visibility
timeout on every attempt. Configure `maxReceiveCount` high enough to tolerate transient failures
but low enough to isolate deterministic poison input. Alarm on both DLQ message count and
`ApproximateAgeOfOldestMessage`; include source queue, message type, producer version, failure
class, and trace/job ID in logs so operators can determine whether redrive is safe.

Redrive is an operational change, not a “retry all” button:

1. Preserve/sample the failed payload and identify the cause.
2. Deploy and verify the worker or producer fix.
3. Redrive a small rate-limited batch and watch success, age, and downstream saturation.
4. Pause on repeated failure; never create a DLQ-to-source loop.

Set source and DLQ retention to cover detection plus repair time. If failed work has a business
expiry, validate that before replay so an old command cannot perform a now-invalid action.

### Scale to a backlog-age SLO, not just a pretty utilization graph

`ApproximateAgeOfOldestMessage` answers whether the oldest eligible work is approaching its
latency objective. Use it with visible backlog, in-flight count, completion/error rate, DLQ depth,
and ASG `InService`/pending/failed-launch metrics. Derive the backlog-per-instance target from a
load test:

```text
acceptable backlog per worker
  = acceptable queue wait seconds × sustained messages per worker per second
```

Measure a slow/large message mix, not only the average job. Target tracking handles the normal
case; add a high-age alarm and an emergency step policy or operator action for an SLO breach.

CloudWatch missing data is **unknown, not zero**. If the custom backlog metric stops publishing,
its target-tracking alarms can enter `INSUFFICIENT_DATA` and scaling may stall. Publish a real zero
when the queue is empty, alarm on metric absence, guard division by zero, and keep a bootstrap
minimum (or a separate wake-up mechanism) if the fleet is allowed to reach zero.

### Regional recovery

An SQS queue, its URL, DLQ, KMS key, policies, alarms, and scaling policy are Regional. Deploy the
same stack from IaC in the recovery Region and pre-validate IAM, AMIs, instance capacity, and
downstream connectivity. SQS does not turn a queue into a cross-Region replicated log. Choose one
of these data strategies explicitly:

- Route new producer traffic to the recovery queue and accept loss of unprocessed primary-Region
  messages within the declared RPO.
- Archive commands/events durably and replay them to the recovery queue.
- Publish to both Regions through an application-owned outbox/replicator, accepting duplicate and
  reordering handling through the same idempotency keys.

During failover, fence the old producers/workers before enabling recovery consumers, restore the
minimum On-Demand worker floor, verify completion and DLQ rates, and then raise traffic. Failback
requires draining or reconciling both queues; changing DNS alone does not merge them.

---

## 7. Troubleshooting

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

**Instances terminate while jobs are still running**

- Add a termination lifecycle hook, stop polling before termination, heartbeat message visibility,
  and use short-lived scale-in protection only while a job is active. Handle Spot interruption
  separately; it is not delayed by a normal lifecycle hook indefinitely.

**Backlog and age grow but desired capacity does not**

- Check for a missing custom metric/`INSUFFICIENT_DATA`, ASG `MaxSize`, EC2 or Spot capacity
  errors, launch-template/AMI failures, and account quotas. A scaling policy cannot create
  capacity that the ASG cannot launch.

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
- Use conditional idempotency state, graceful lifecycle-hook drains, Capacity Rebalancing, and a
  mixed On-Demand/Spot capacity floor for production workers.
- Alarm on **oldest-message age** and DLQ state. Missing custom metrics are not evidence of an
  empty queue.
- SQS is Regional; recovery requires a prebuilt queue/worker stack and an explicit replay,
  dual-publish, or accepted-data-loss strategy.

---

**Next**: [16_cloudwatch_alarm_scaling.md — CloudWatch Alarms Driving Scaling Actions](16_cloudwatch_alarm_scaling.md)
