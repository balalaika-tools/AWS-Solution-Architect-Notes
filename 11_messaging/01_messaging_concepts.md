# Messaging & Decoupling Concepts

> **Who this is for**: Engineers who know how to call one service from another over HTTP but
> haven't formally studied *why* you'd put a queue or a topic in between. This is the
> prerequisite file for the whole messaging section — SQS, SNS, EventBridge, and Kinesis are all
> just AWS implementations of the patterns described here. No AWS knowledge is assumed.

---

## 1️⃣ Synchronous vs Asynchronous Communication

Two services can talk in one of two timing models.

**Synchronous** — the caller sends a request and *waits* (blocks) for the response. The two
services are temporally bound: both must be up, healthy, and fast at the same instant.

```
   Order Service                 Payment Service
       │                              │
       │──── POST /charge ───────────►│
       │                              │ (processing… caller blocks)
       │◄──── 200 OK ─────────────────│
       │
   continues
```

**Asynchronous** — the caller hands off a message and immediately moves on. Some intermediary
(a queue, a topic, a stream) holds the message until the receiver is ready to process it.

```
   Order Service        Queue          Payment Service
       │                  │                  │
       │── enqueue ──────►│                  │
       │◄─ ack (instant)  │                  │
   continues              │── deliver later ►│
                          │                  │ (processes when ready)
```

| Aspect | Synchronous | Asynchronous |
|--------|-------------|--------------|
| Caller waits? | Yes, blocks for response | No, fires and forgets |
| Both services up at once? | Required | Not required |
| Failure of receiver | Caller fails too | Message waits / retries |
| Latency seen by caller | Full round-trip | Just the enqueue |
| Examples | REST call, RPC, DB query | SQS, SNS, Kinesis, email |

> **Key insight**: Asynchronous messaging is the mechanism that lets you *decouple* services in
> time. The producer doesn't need the consumer to exist, be healthy, or be fast right now.

💡 Async isn't free — you trade immediate consistency for resilience. The caller gets an "accepted"
ack, not a "done" result, so the system becomes **eventually consistent**.

---

## 2️⃣ Tight vs Loose Coupling

**Coupling** measures how much one component depends on the specifics of another.

```
TIGHT COUPLING (direct, synchronous calls)
                                                                  if any box is
   ┌─────────┐    ┌─────────┐    ┌─────────┐                      down or slow,
   │   Web   │───►│  Order  │───►│ Payment │                      the whole
   └─────────┘    └─────────┘    └─────────┘                      chain breaks
       knows the address, port, and availability of the next hop

LOOSE COUPLING (mediated by a messaging layer)
   ┌─────────┐    ┌───────┐    ┌─────────┐    ┌───────┐    ┌─────────┐
   │   Web   │───►│ Queue │───►│  Order  │───►│ Queue │───►│ Payment │
   └─────────┘    └───────┘    └─────────┘    └───────┘    └─────────┘
       producers and consumers only know the queue, not each other
```

In tight coupling, a consumer being down, slow, or rescaled directly affects the producer. In
loose coupling, the messaging layer absorbs those differences — neither side knows the other's
address, count, or health.

| Dimension | Tight Coupling | Loose Coupling |
|-----------|----------------|----------------|
| Knowledge of peer | IP/DNS, port, API | Only the queue/topic name |
| Consumer outage | Breaks the producer | Messages wait, then retry |
| Scale independently | Hard | Easy (add consumers freely) |
| Deploy independently | Risky (lockstep) | Safe |
| Failure blast radius | Cascades upstream | Contained at the boundary |

---

## 3️⃣ Why Decouple

Three concrete payoffs the exam expects you to recognize.

**Resilience** — a consumer can crash, get patched, or scale to zero, and messages simply wait in
the queue instead of being lost. The producer never sees the failure.

✅ Use a queue when the downstream service has variable availability and you must not drop work.

**Independent scaling** — producers and consumers scale on their own metrics. A web tier can
spike to 10,000 req/s while a fixed pool of workers drains the backlog at its own steady rate.

```
   bursty producers                steady consumers
   ████████████████  ──► [ QUEUE ] ──►  ▓▓▓▓
   (10k req/s spike)   buffer absorbs   (drains at 500/s)
```

**Buffering & backpressure** — the queue is a shock absorber. When producers outpace consumers,
the backlog grows instead of overwhelming (and crashing) the consumer. The queue *depth* becomes
a signal: a growing backlog tells you to add consumers (the basis of SQS-driven Auto Scaling).

⚠️ Buffering hides a slow consumer, it doesn't fix it. If consumers are *permanently* slower than
producers, the backlog grows without bound until messages expire (data loss) — buffering buys you
time to scale, not a free lunch.

---

## 4️⃣ The Three Messaging Models

Almost every messaging system is one of three models. Knowing which model a service implements is
the single most useful mental shortcut for this exam.

### Message Queue (point-to-point)

One message is claimed by **one consumer at a time** out of a competing pool. Consumers pull work;
the message is removed once acknowledged, although a retry can deliver it again. Used for **work
distribution** — load-leveling tasks across a fleet.

```
                    ┌──────────────┐
  producer ───────► │ msg msg msg  │ ───┬──► consumer A   (each message goes
  producer ───────► │  (the queue) │    ├──► consumer B    to ONE consumer —
                    └──────────────┘    └──► consumer C    they compete)
```
**AWS service: SQS.**

### Publish / Subscribe (pub-sub)

One message is **copied to every subscriber** of a topic. The publisher doesn't know or care how
many subscribers exist. Used for **broadcasting / notifying** multiple interested parties.

```
                    ┌─────────┐ ──copy──► subscriber A
  publisher ──────► │  TOPIC  │ ──copy──► subscriber B   (every subscriber gets
                    └─────────┘ ──copy──► subscriber C    its OWN copy)
```
**AWS service: SNS** (and EventBridge, which adds routing rules on top).

### Event Streaming (log)

Messages are appended to an ordered, durable **log** that is **retained** for a period. Many
independent consumers read the log at their own pace, each tracking its own position, and can
**replay** from any point. Used for **real-time analytics, ordered processing, and replay**.

```
  producers ─► │0│1│2│3│4│5│6│7│… ◄─ append-only ordered log (retained 24h–365d)
                  ▲           ▲
            consumer X   consumer Y    (each reads independently, can rewind/replay)
```
**AWS service: Kinesis Data Streams** (and Kafka/MSK).

---

## 5️⃣ Three-Model Comparison

| | Queue (SQS) | Pub/Sub (SNS) | Stream (Kinesis) |
|---|---|---|---|
| Delivery | One message → one consumer | One message → all subscribers | Append to log, many readers |
| After consume | Message **deleted** | Copy delivered, then gone | Message **stays** until retention expires |
| Replay old messages | ❌ No | ❌ No | ✅ Yes (re-read the log) |
| Ordering | Standard: no / FIFO: yes | FIFO topics only | Per-shard ordering |
| # of consumers per message | 1 (competing pool) | N (all subscribers) | N (independent positions) |
| Primary purpose | Work distribution, buffering | Broadcast / notify / fan-out | Analytics, ordered events, replay |
| Mental model | To-do list | Mailing list | Tape recording / DVR |

> **Key insight**: The deciding question is *"after a message is read, what happens to it?"* —
> deleted (queue), copied-and-gone (pub/sub), or retained for replay (stream). Pick the model
> first, then the service follows.

---

## 6️⃣ Producers, Consumers, and Delivery Semantics

A **producer** (publisher) creates and sends messages. A **consumer** (subscriber) receives and
processes them. The same service is often both — a consumer of one queue and a producer to the
next.

**Delivery semantics** describe what guarantees the system makes about how many times and in what
order a message arrives. The exam tests these by name.

| Guarantee | Meaning | Trade-off | Example |
|-----------|---------|-----------|---------|
| **At-most-once** | Delivered 0 or 1 times | May lose messages, never duplicates | Fire-and-forget metrics |
| **At-least-once** | Delivered 1+ times | Never lost, but **may duplicate** | SQS Standard, SNS |
| **Exactly-once effect** | The business outcome happens once even if transport retries | Requires deduplication, idempotency, and usually state | Idempotent payment keyed by transaction ID |

⚠️ **At-least-once is the default in most distributed messaging** (including SQS Standard and SNS).
The retry that improves delivery reliability is the same mechanism that can produce a duplicate.
Your consumers must therefore be **idempotent** — processing the same message twice has the same
effect as processing it once (e.g., key writes by a unique message/transaction ID).

FIFO services deduplicate sends within a bounded window and preserve order within a group, but
that is not a blanket guarantee that the *business transaction* executes exactly once. A consumer
can finish its side effect and fail before acknowledging the message. Keep consumer idempotency
even when the transport is FIFO.

**Ordering** is a separate guarantee:
- **No ordering** — messages may arrive in any order (SQS Standard, for max throughput).
- **Ordered** — messages arrive in the sequence sent, usually *per group/partition*, not globally
  (SQS FIFO message groups, Kinesis per-shard).

💡 Strict ordering and transport deduplication reduce available parallelism. Standard/best-effort
modes scale much further; ordered modes depend on the number and distribution of groups or
partitions. This trade-off recurs in SQS (Standard vs FIFO) and Kinesis (shard limits).

---

## 7️⃣ The Fan-Out Pattern

**Fan-out** = one event is delivered to *multiple* downstream consumers, each doing different work,
in parallel and independently. It is the canonical SNS+SQS architecture and a guaranteed exam
topic.

```
                                    ┌──────────┐     ┌─────────────┐
                                ┌──►│  SQS  Q1 │────►│ Email worker│
                                │   └──────────┘     └─────────────┘
   ┌────────┐    ┌─────────┐    │   ┌──────────┐     ┌─────────────┐
   │ Producer│──►│   SNS   │────┼──►│  SQS  Q2 │────►│ Analytics   │
   └────────┘    │  Topic  │    │   └──────────┘     └─────────────┘
                 └─────────┘    │   ┌──────────┐     ┌─────────────┐
                                └──►│  SQS  Q3 │────►│ Inventory   │
                                    └──────────┘     └─────────────┘
       one "OrderPlaced" event   →  three queues  →  three independent consumers
```

Why SNS *into* SQS rather than SNS straight to the workers?

✅ Each SQS queue gives its consumer **its own buffer, retry, and DLQ** — one slow or broken worker
can't affect the others, and no message is lost if a worker is down.

❌ Subscribing workers *directly* to SNS (e.g., HTTP endpoints) couples recovery to the SNS retry
policy. After retries are exhausted the message is lost unless the subscription has a DLQ; there
is no durable, consumer-controlled backlog to drain.

This combination is so common it has a name on the exam: the **"SNS fan-out to SQS"** pattern.
EventBridge offers a similar fan-out with content-based routing rules (covered in file 04).

---

## 8️⃣ Production Architecture Decision Framework

Do not choose a messaging service from the happy-path diagram. Write down the contract below
first; it exposes the operational work the architecture must perform during an outage.

| Requirement | Design question | Production consequence |
|-------------|-----------------|------------------------|
| **Delivery** | Is loss acceptable? Are duplicates acceptable? What proves completion? | Prefer at-least-once plus an idempotency key for important work. A transport acknowledgement means "accepted," not "business transaction committed." |
| **Ordering scope** | Must all events be ordered, or only events for one order/account/device? | Global order destroys parallelism. Partition by the smallest entity that needs order and reject designs that rely on wall-clock timestamps alone. |
| **Replay** | Must a new consumer or repaired system process history? For how long? | Use a retained stream or event archive. A queue DLQ is a failure quarantine, not a general event history. |
| **Back-pressure** | What happens when arrival rate exceeds service rate, and for how long? | Size retention for the outage window, cap consumer concurrency to protect dependencies, and alarm on message/event age—not just count. |
| **Poison isolation** | How many transient retries occur before quarantine? Who repairs and replays? | Use bounded retries, a DLQ/quarantine store, an alert, a runbook, and controlled replay. Blind redrive recreates the incident. |
| **Schema evolution** | Who owns the contract? How are incompatible changes detected? | Use a standard envelope, explicit schema version, additive changes, tolerant readers, and compatibility tests in CI. Keep large mutable payloads out of the event. |
| **Disaster recovery** | What RTO/RPO applies to transport, consumers, state, and keys? | Deploy the whole path in the recovery Region. Define producer failover, replication/dual-publish behavior, duplicate reconciliation, and failback—not only a second queue or bus. |
| **Trust boundary** | Which accounts may publish, route, consume, or administer replay? | Combine identity and resource policies, restrict service principals with source conditions, and include KMS key policies. Separate publish, consume, and redrive roles. |

### Back-pressure and capacity math

Let peak arrival rate be `λ` messages/s, one worker's sustainable service rate be `μ`, and worker
count be `N`. A stable system needs `N × μ > λ`; otherwise the backlog grows by
`λ - (N × μ)` each second. If a backlog already exists, approximate drain time as:

```
drain_time_seconds = backlog / ((N × μ) - λ)
```

That formula is only a starting point. Also check transport quotas, batch size, downstream API and
database limits, ordering partitions, in-flight work, and retry amplification. Scaling consumers
faster than the dependency they call simply moves the outage downstream.

### Schema and payload contract

A useful event envelope includes at least:

```json
{
  "event_id": "01J...",
  "event_type": "OrderPlaced",
  "schema_version": 2,
  "occurred_at": "2026-07-21T12:34:56Z",
  "producer": "orders",
  "correlation_id": "checkout-123",
  "causation_id": "command-456",
  "subject": "order/789",
  "data": {}
}
```

`event_id` is the deduplication key; `correlation_id` ties the end-to-end business workflow
together; `causation_id` identifies the command or event that caused this event. Never put secrets
or unrestricted personal data into events merely because encryption is enabled—every fan-out copy,
archive, DLQ, and log expands the data boundary.

### Observability for asynchronous workflows

An asynchronous request does not have one request latency. Instrument each hop and the business
outcome:

- **Transport health** — publish failures/throttles, visible backlog, oldest-message age, in-flight
  work, consumer lag, retry count, and DLQ depth.
- **Consumer health** — processing latency, success/failure by event type and schema version,
  idempotency hits, dependency throttles, and last successful checkpoint.
- **Workflow health** — time from the original command to the final business state, correlated by
  `correlation_id`. Propagate trace context where supported, but keep durable IDs because a trace
  may expire before a delayed message is processed.
- **Recovery health** — redrive/replay rate, repeated failures, regional replication lag, and the
  age of the oldest unreconciled business event.

Logs alone are insufficient: create alarms tied to an operator action and dashboard the service
level objective the queue or stream protects.

---

**Next**: [02_sqs.md — Amazon SQS](02_sqs.md)
