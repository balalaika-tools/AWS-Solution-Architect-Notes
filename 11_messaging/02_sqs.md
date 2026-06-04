# Amazon SQS — Simple Queue Service

> **Who this is for**: Engineers who understand the **queue model** and **at-least-once delivery**
> from [01_messaging_concepts.md](01_messaging_concepts.md) and now need the AWS implementation.
> SQS is the most heavily tested messaging service on SAA-C03 — the **Standard vs FIFO** table and
> the **visibility-timeout duplicate trap** appear on nearly every exam.

---

## 1. What SQS Is

**Amazon SQS (Simple Queue Service)** is a fully managed, serverless **message queue**. Producers
send messages to a queue; a pool of consumers polls the queue, processes each message, and deletes
it. SQS implements the **point-to-point queue model**: each message is processed by exactly one
consumer from the competing pool.

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
 ├─ at-least-once (may DUPLICATE)        ├─ exactly-once processing
 └─ best-effort ordering (may reorder)   └─ strict ordering per MessageGroupId
```

| Feature | **Standard** | **FIFO** |
|---------|-------------|----------|
| Throughput | **Nearly unlimited** | Default: 300 API calls/s per action, **3,000 messages/s with batching**; high-throughput FIFO can scale much higher to regional quotas |
| Delivery | **At-least-once** (duplicates possible) | **Exactly-once** processing |
| Ordering | Best-effort (can arrive out of order) | **Strict FIFO** within a Message Group |
| Dedup | None (consumers must be idempotent) | 5-minute dedup window (`MessageDeduplicationId`) |
| Queue name | any | must end in **`.fifo`** |
| Use when | Max throughput, order doesn't matter | Order and no-duplicates matter (e.g., financial txns) |

Two FIFO-specific attributes:
- **`MessageGroupId`** — ordering is guaranteed *within a group*. Different groups are processed in
  parallel, so you get ordering *and* parallelism (e.g., one group per user ID).
- **`MessageDeduplicationId`** — within a 5-minute window, a duplicate `SendMessage` with the same
  dedup ID is silently discarded (or content-based dedup hashes the body).

⚠️ Don't reflexively pick FIFO for "ordering." Most workloads don't need strict order and would be
throttled by FIFO's throughput cap. Default to **Standard**; choose FIFO only when the question
*explicitly* requires no duplicates or strict order.

💡 **High-throughput FIFO**: if you truly need FIFO semantics at higher rates, enable high-throughput
mode and spread work across many `MessageGroupId` values. A single hot message group is still
sequential; parallelism comes from many groups.

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
- ✅ Set the visibility timeout **longer than your worst-case processing time**.
- ✅ For variable durations, the consumer can extend the timeout mid-flight with
  **`ChangeMessageVisibility`** (a "heartbeat").
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
- **Redrive to source** lets you replay DLQ messages back after fixing the bug.

⚠️ A common mistake is no DLQ at all — a poison-pill message then cycles forever, wasting compute and
blocking nothing but generating cost and noise.

---

## 6. Message Size, Retention, and Delay

| Setting | Default | Range | Notes |
|---------|---------|-------|-------|
| **Max message size** | 256 KB | up to 256 KB | For larger payloads, see extended client below |
| **Message retention** | 4 days | 60s – **14 days** | How long an undeleted message survives |
| **Delivery delay** (delay queue) | 0s | 0 – **15 min** | Hides new messages for N seconds |
| **Visibility timeout** | 30s | 0s – **12 hours** | See §3 |
| **Long poll wait** | 0s | 0 – 20s | See §4 |

**Large payloads (> 256 KB)** — use the **SQS Extended Client Library**: the actual payload is
stored in **S3**, and only a *pointer* (S3 object reference) is sent through SQS. The library
transparently puts to / gets from S3.

```
  producer ─► [ put 2 MB payload ] ─► S3
           └─► [ send S3 pointer  ] ─► SQS ─► consumer ─► [ get payload ] ◄─ S3
```

**Delay queues** delay *every* new message in the queue by up to 15 minutes (queue-level). For a
*single* message, use the **per-message `DelaySeconds`** attribute instead.

> **Limits to memorize**: 256 KB max size, 14 days max retention, 12 hours max visibility timeout,
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

## 8. Key Exam Points

- **SQS = queue model**, pull-based, one message → one consumer, message deleted after processing.
- **Standard**: unlimited throughput, at-least-once (**duplicates possible**), best-effort order.
- **FIFO**: exactly-once, strict order per `MessageGroupId`, default **300 API calls/s**
  (**3,000 messages/s batched**), higher with high-throughput FIFO + many message groups; name ends
  in `.fifo`. Choose only when order/no-dupes is explicitly required.
- **Visibility timeout**: message hidden after receive; if not deleted in time it reappears →
  **duplicate processing**. Fix: timeout > processing time, `ChangeMessageVisibility`, idempotency.
- **Long polling** (`WaitTimeSeconds = 20`) reduces cost and empty receives — use it.
- **DLQ + `maxReceiveCount`** isolates poison-pill messages; alarm on the DLQ.
- **256 KB** max message → use the **Extended Client + S3** for larger payloads.
- Retention up to **14 days**; delay up to **15 min**; visibility up to **12 hours**.
- Scale consumers on **queue depth / backlog-per-instance**, not CPU.
- SQS does **not** fan-out by itself — pair **SNS → SQS** for one-to-many (file 03).

---

## 9. Common Mistakes

- ❌ **Visibility timeout shorter than processing time** → the message reappears and is processed
  twice. The most common SQS bug.
- ❌ Assuming Standard SQS never duplicates — it's **at-least-once**; build idempotent consumers.
- ❌ Choosing FIFO "to be safe" and then hitting the default throughput cap or a single hot
  `MessageGroupId`. Use Standard when order is unnecessary; use high-throughput FIFO + many groups
  when order is required at scale.
- ❌ Using **short polling** and paying for thousands of empty `ReceiveMessage` calls.
- ❌ Forgetting a **DLQ**, letting a poison-pill message loop forever.
- ❌ Sending payloads > 256 KB directly — must use the **S3 extended client**.
- ❌ Auto-scaling consumers on CPU instead of **queue depth**.

---

**Next**: [03_sns.md — Amazon SNS](03_sns.md)
