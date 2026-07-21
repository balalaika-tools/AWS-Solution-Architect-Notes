# Amazon SQS — Simple Queue Service

> **Who this is for**: Engineers who understand the **queue model** and **at-least-once delivery**
> from [01_messaging_concepts.md](01_messaging_concepts.md) and now need the AWS implementation.
> SQS is the most heavily tested messaging service on SAA-C03 — the **Standard vs FIFO** table and
> the **visibility-timeout duplicate trap** appear on nearly every exam.

---

## 1. What SQS Is

**Amazon SQS (Simple Queue Service)** is a fully managed, serverless **message queue**. Producers
send messages to a queue; a pool of consumers polls the queue, processes each message, and deletes
it. SQS implements the **point-to-point queue model**: one consumer in the competing pool claims a
message at a time. Delivery can still be repeated, so the consumer must be safe to run again.

```
   producers                  SQS queue                 consumers (poll)
   ┌──────┐   SendMessage   ┌───────────────┐  ReceiveMessage  ┌──────┐
   │ Web  │ ───────────────►│ m1 m2 m3 m4 …  │ ────────────────►│ EC2  │
   └──────┘                 └───────────────┘                  └──────┘
                                                  DeleteMessage  (after success)
```

> **Key insight**: SQS is **pull-based** — consumers *poll* for messages; SQS never pushes to them.
> (Lambda *appears* push-based but is really the Lambda service polling on your behalf.) A message
> is only removed when a consumer explicitly calls `DeleteMessage` after successful processing.

It is fully managed: no servers, near-unlimited scale, and pay-per-request. It is the default
answer for "decouple two components" and "buffer/load-level work."

---

## 2. Standard vs FIFO Queues

SQS has two queue types. Choosing between them is a near-certain exam question.

```
STANDARD                                FIFO  (name MUST end in .fifo)
 ├─ nearly unlimited throughput          ├─ default: 300 API calls/s (3,000 msg/s batched)
 ├─ at-least-once (may DUPLICATE)        ├─ 5-minute send deduplication
 └─ best-effort ordering (may reorder)   └─ strict ordering per MessageGroupId
```

| Feature | **Standard** | **FIFO** |
|---------|-------------|----------|
| Throughput | **Nearly unlimited** | Default: 300 API calls/s per action, **3,000 messages/s with batching**; high-throughput FIFO can scale much higher to regional quotas |
| Delivery | **At-least-once** (duplicates possible) | Deduplicated sends within 5 minutes; consumers still need idempotency |
| Ordering | Best-effort (can arrive out of order) | **Strict FIFO** within a Message Group |
| Dedup | None (consumers must be idempotent) | 5-minute dedup window (`MessageDeduplicationId`) |
| Queue name | any | must end in **`.fifo`** |
| Use when | Max throughput, order doesn't matter | Order and no-duplicates matter (e.g., financial txns) |

Two FIFO-specific attributes:
- **`MessageGroupId`** — ordering is guaranteed *within a group*. Different groups are processed in
  parallel, so you get ordering *and* parallelism (e.g., one group per user ID).
- **`MessageDeduplicationId`** — within a 5-minute window, a duplicate `SendMessage` with the same
  dedup ID is silently discarded (or content-based dedup hashes the body).

⚠️ Don't reflexively pick FIFO for "ordering." Most workloads don't need strict order. Default to
**Standard**; choose FIFO when the business workflow needs ordering within a well-defined entity.
FIFO's send deduplication reduces producer duplicates, but it does not make an arbitrary consumer
side effect exactly once.

💡 **High-throughput FIFO**: enable high-throughput mode by using message-group deduplication scope
and the per-message-group throughput limit, then spread work across many `MessageGroupId` values.
A single hot group is still sequential. Default FIFO supports 300 API calls/s per action, or 3,000
messages/s when every batch contains ten messages; high-throughput regional quotas are much larger
and vary by Region, so confirm them in Service Quotas during capacity planning.

---

## 3. Visibility Timeout — and the Duplicate-Processing Trap

When a consumer receives a message, SQS does **not** delete it. Instead it hides the message from
other consumers for the **visibility timeout** (default **30s**, max **12 hours**). The consumer
must call `DeleteMessage` *before* the timeout expires. If it does, the message is gone. If it
doesn't, the message becomes visible again and **another consumer will receive it**.

```
  t=0   Consumer A: ReceiveMessage  ──► msg hidden (visibility timeout starts, 30s)
  t=5   Consumer A: processing…
  t=30  TIMEOUT expires — A hasn't deleted yet
  t=30  msg becomes VISIBLE again
  t=31  Consumer B: ReceiveMessage  ──► PROCESSES THE SAME MESSAGE  ⚠️ duplicate!
  t=40  Consumer A finally finishes, DeleteMessage  (too late — B already ran it)
```

> **The trap**: If processing takes *longer* than the visibility timeout, the message reappears and
> gets processed **twice**. This is the #1 source of SQS duplicate processing and a classic exam
> scenario.

How to handle it:
- ✅ Set the initial visibility timeout above normal processing time plus delete/acknowledgement
  latency. Avoid one extreme timeout that makes genuine failures take hours to retry.
- ✅ For variable-duration work, extend the timeout incrementally with
  **`ChangeMessageVisibility`** (a heartbeat). Stop extending if the worker loses ownership or
  becomes unhealthy, and remember that a message cannot remain invisible for more than 12 hours
  from the original receive.
- ✅ Make consumers **idempotent** — because SQS Standard is at-least-once, duplicates can happen
  even with a correct timeout. Idempotency is the real fix.

---

## 4. Polling — Long vs Short

Consumers retrieve messages by polling. Two modes:

| | **Short polling** (default) | **Long polling** (recommended) |
|---|---|---|
| Behavior | Returns immediately, even if empty | Waits up to `WaitTimeSeconds` for a message |
| `WaitTimeSeconds` | 0 | 1–20 (set to **20**) |
| Empty responses | Many (you poll constantly) | Few |
| Cost | Higher (you pay per API request) | **Lower** (fewer empty receives) |
| Latency to first msg | Lowest | Slightly higher |

✅ **Use long polling** (`WaitTimeSeconds = 20`) in almost all cases — it cuts cost and reduces
empty receives without meaningfully hurting latency. Set it on the queue (`ReceiveMessageWaitTimeSeconds`)
or per request.

💡 Short polling samples only a *subset* of SQS servers, so it can return "empty" even when messages
exist. Long polling queries all servers, so it won't falsely report empty.

---

## 5. Dead-Letter Queues (DLQ)

A **dead-letter queue** is a separate SQS queue that captures messages a consumer repeatedly fails
to process, so they don't loop forever (a **poison pill**) and can be inspected later.

You configure a **redrive policy** on the source queue pointing at the DLQ, with a
**`maxReceiveCount`**. Each failed receive (visibility timeout expires without delete) increments
the message's receive count. When it exceeds `maxReceiveCount`, SQS moves the message to the DLQ.

```
  source queue ── receive #1 fails ─┐
                ── receive #2 fails ─┤  maxReceiveCount = 3
                ── receive #3 fails ─┘
                ── receive #4  ──────────► moved to DLQ  (give up, alarm on it)
```

- The DLQ type must match the source (FIFO source → FIFO DLQ).
- ✅ Put a **CloudWatch alarm** on the DLQ's `ApproximateNumberOfMessagesVisible` — a non-zero DLQ
  means broken messages need attention.
- A **redrive allow policy** on the DLQ controls which source queues may use it. Restrict it to the
  intended queues rather than accepting every source in the account.
- **Redrive to source** (or to another queue) starts a managed message-move task after the defect is
  fixed. Set a conservative redrive velocity, release a canary first, and watch failure and age
  metrics; moving the full DLQ at line rate can reproduce the outage.
- The redrive operator needs receive/delete/read permissions on the DLQ and send permission on the
  destination. Encrypted queues also require decrypt access to source keys and data-key/decrypt
  access to the destination key.

⚠️ A common mistake is no DLQ at all — a poison-pill message then cycles forever, wasting compute and
blocking nothing but generating cost and noise.

---

## 6. Message Size, Retention, and Delay

| Setting | Default | Range | Notes |
|---------|---------|-------|-------|
| **Max message size** | 1 MiB | 1 byte – **1 MiB** | For larger payloads, see extended client below |
| **Message retention** | 4 days | 60s – **14 days** | How long an undeleted message survives |
| **Delivery delay** (delay queue) | 0s | 0 – **15 min** | Hides new messages for N seconds |
| **Visibility timeout** | 30s | 0s – **12 hours** | See §3 |
| **Long poll wait** | 0s | 0 – 20s | See §4 |

**Large payloads (> 1 MiB)** — use the **SQS Extended Client Library**: the actual payload is
stored in **S3**, and only a *pointer* (S3 object reference) is sent through SQS. The library
transparently puts to / gets from S3 for payloads up to 2 GB. The extended libraries work with
synchronous clients; define S3 lifecycle, encryption, access, and orphan cleanup because deleting
the SQS message and deleting the S3 object are separate operations.

```
  producer ─► [ put 2 MB payload ] ─► S3
           └─► [ send S3 pointer  ] ─► SQS ─► consumer ─► [ get payload ] ◄─ S3
```

**Delay queues** delay *every* new message in the queue by up to 15 minutes (queue-level). For a
*single* message, use the **per-message `DelaySeconds`** attribute instead.

> **Limits to memorize**: 1 MiB max size, 14 days max retention, 12 hours max visibility timeout,
> 20s max long-poll wait, 15 min max delay.

---

## 7. SQS with Auto Scaling — Queue-Depth Scaling

The headline SQS architecture pattern: scale a consumer **Auto Scaling Group** based on **queue
depth**, not CPU. The buffering property of the queue (file 01, §3) becomes the scaling signal.

```
  producers ─► [ SQS queue ] ─► ┌─ ASG of EC2 consumers ─┐
                    │            │ instance instance …    │
                    │            └────────────────────────┘
                    ▼                       ▲
        CloudWatch metric:                  │ scale out when backlog grows
        ApproximateNumberOfMessagesVisible ─┘ scale in when it drains
```

✅ The recommended metric is **backlog per instance** =
`ApproximateNumberOfMessagesVisible / RunningInstanceCount`, compared against an "acceptable
backlog per instance" target. Scaling on raw queue length alone ignores how many workers you
already have.

⚠️ Don't scale consumers on **CPU** — a queue backlog can grow while CPU is low (workers idle waiting
on I/O), so CPU is the wrong signal. Queue depth directly reflects pending work.

See the full walkthrough: [SQS with Auto Scaling Group](../18_practical_examples/15_sqs_with_asg.md).

---

## 8. Production Consumer Design

### Lambda event source mappings and partial batch failure

Lambda polls SQS and invokes the function with a batch. By default, one uncaught error makes the
**entire batch** visible again after the visibility timeout, including messages already processed.
Enable `ReportBatchItemFailures`, catch per-record errors, and return only failed message IDs in
`batchItemFailures` to avoid unnecessary reprocessing.

For a FIFO queue, stop processing after the first failure in the ordered sequence and report the
failed and all unprocessed records. Continuing after the failure can violate the workflow's order.
Lambda concurrency for FIFO is also bounded by active message groups, so one group cannot exploit a
large function concurrency setting.

Partial batch reporting reduces duplicate work; it does not remove the need for idempotency. If the
function itself crashes or times out, Lambda treats the batch as failed. Set queue visibility to
cover the function timeout, batching window, and retry/throttling margin, and choose
`maxReceiveCount` high enough that Lambda has a realistic chance to process a transient failure.

### Long-running tasks and in-flight quotas

A robust worker follows this lifecycle:

1. Receive a message and record its message ID, receive count, and start time.
2. Acquire an idempotency/ownership record before causing an irreversible side effect.
3. Heartbeat with `ChangeMessageVisibility` before the current timeout expires.
4. Persist the business result, then delete using the latest receipt handle.
5. On a retryable failure, stop heartbeats and let the message reappear; on a permanent data error,
   let the bounded retry policy move it to the DLQ.

Standard queues support approximately 120,000 in-flight messages; FIFO queues have a 120,000
in-flight quota. At the limit, long polling can look like an empty queue rather than returning an
error. Alarm on `ApproximateNumberOfMessagesNotVisible`, keep processing times bounded, delete
promptly, and request a quota increase or shard work across queues before the peak.

If legitimate work can exceed SQS's 12-hour visibility boundary, put durable orchestration in Step
Functions or store job state outside the queue. A receipt handle is not a long-term lease.

### Cross-account queues and KMS

Cross-account access is a two-sided trust relationship:

- The consuming or producing principal needs an IAM identity policy for the required SQS actions.
- The queue owner must add a queue resource policy that trusts that principal. For an AWS service
  publisher such as SNS or EventBridge, grant the service principal `sqs:SendMessage` and restrict
  it with the exact `aws:SourceArn` and, where supported, `aws:SourceAccount`/organization boundary
  to prevent a confused deputy.
- Separate publish, consume, purge, and redrive permissions. A worker rarely needs
  `sqs:PurgeQueue`, `sqs:SetQueueAttributes`, or message-move APIs.
- With a customer-managed KMS key, producers need `kms:GenerateDataKey` and `kms:Decrypt`;
  consumers need `kms:Decrypt`. The KMS key policy must trust the cross-account principals or
  services as well as the IAM policy. Use a customer-managed key for cross-account control rather
  than relying on the AWS managed SQS key.

Test the policy with the actual role and encryption setting. A queue policy can allow
`SendMessage` while a missing KMS grant still makes every encrypted publish fail.

### Backlog alarms and operating thresholds

Monitor the queue as a waiting-time system, not just a message counter:

| Signal | What it reveals | Typical action |
|--------|-----------------|----------------|
| `ApproximateAgeOfOldestMessage` | User-visible delay and risk of exceeding retention/SLO | Page when age breaches the processing SLO; scale or shed non-critical work. |
| `ApproximateNumberOfMessagesVisible` | Pending backlog | Scale on backlog per worker and compare arrival versus delete rate. |
| `ApproximateNumberOfMessagesNotVisible` | In-flight saturation, slow/stuck consumers | Inspect processing latency and heartbeat/delete behavior; check the in-flight quota. |
| DLQ `ApproximateNumberOfMessagesVisible` | Poison messages or exhausted retries | Alert an owner, sample safely, fix first, then controlled redrive. |
| Sent vs deleted rate | Whether the backlog will grow or drain | Capacity-plan sustained rate, not only a short burst benchmark. |

For an arrival rate `λ`, per-worker service rate `μ`, and `N` workers, sustainable capacity requires
`N × μ > λ`. Backlog drain time is approximately `backlog / ((N × μ) - λ)`. Cap concurrency at the
downstream database/API capacity; otherwise an SQS scale-out turns backlog pressure into a
dependency outage.

---

## 9. Key Exam Points

- **SQS = queue model**, pull-based, one message → one consumer, message deleted after processing.
- **Standard**: unlimited throughput, at-least-once (**duplicates possible**), best-effort order.
- **FIFO**: send deduplication + strict order per `MessageGroupId`, default **300 API calls/s**
  (**3,000 messages/s batched**), higher with high-throughput FIFO + many message groups; name ends
  in `.fifo`. Consumers remain idempotent.
- **Visibility timeout**: message hidden after receive; if not deleted in time it reappears →
  **duplicate processing**. Fix: timeout > processing time, `ChangeMessageVisibility`, idempotency.
- **Long polling** (`WaitTimeSeconds = 20`) reduces cost and empty receives — use it.
- **DLQ + `maxReceiveCount`** isolates poison-pill messages; alarm on the DLQ.
- **1 MiB** max message → use the **Extended Client + S3** for larger payloads.
- Lambda: enable `ReportBatchItemFailures`; with FIFO, stop after the first failure and report
  failed plus unprocessed messages.
- Cross-account encrypted queues require queue policy + identity policy + KMS key permissions.
- Alarm on **oldest message age**, backlog, in-flight messages, and DLQ depth; redrive only after
  fixing the cause.
- Retention up to **14 days**; delay up to **15 min**; visibility up to **12 hours**.
- Scale consumers on **queue depth / backlog-per-instance**, not CPU.
- SQS does **not** fan-out by itself — pair **SNS → SQS** for one-to-many (file 03).

---

## 10. Common Mistakes

- ❌ **Visibility timeout shorter than processing time** → the message reappears and is processed
  twice. The most common SQS bug.
- ❌ Assuming Standard SQS never duplicates — it's **at-least-once**; build idempotent consumers.
- ❌ Choosing FIFO "to be safe" and then hitting the default throughput cap or a single hot
  `MessageGroupId`. Use Standard when order is unnecessary; use high-throughput FIFO + many groups
  when order is required at scale.
- ❌ Using **short polling** and paying for thousands of empty `ReceiveMessage` calls.
- ❌ Forgetting a **DLQ**, letting a poison-pill message loop forever.
- ❌ Sending payloads > 1 MiB directly — use the **S3 extended client** and manage S3 cleanup.
- ❌ Auto-scaling consumers on CPU instead of **queue depth**.
- ❌ Returning a Lambda error after successfully processing part of a batch without partial batch
  reporting — every successful record is retried too.
- ❌ Granting `sqs:SendMessage` cross-account but forgetting the queue policy or the KMS key policy.
- ❌ Redriving an entire DLQ before validating the fix and downstream capacity.

---

**Next**: [03_sns.md — Amazon SNS](03_sns.md)
