# Amazon SNS — Simple Notification Service

> **Who this is for**: Engineers who understand the **pub/sub model** and the **fan-out pattern**
> from [01_messaging_concepts.md](01_messaging_concepts.md) and SQS from
> [02_sqs.md](02_sqs.md). SNS is the AWS pub/sub service, and the **SNS→SQS fan-out** pattern plus
> **message filtering** are recurring SAA-C03 topics.

---

## 1. What SNS Is

**Amazon SNS (Simple Notification Service)** is a fully managed **publish/subscribe** service.
Publishers send a message to a **topic**; SNS pushes a **copy** of that message to **every
subscriber** of the topic. The publisher knows nothing about how many subscribers exist or what
they are.

```
                              ┌──────────────► Lambda function
   ┌──────────┐   Publish   ┌─┴─────┐
   │ Producer │ ───────────►│ TOPIC │──────────► SQS queue
   └──────────┘             └─┬─────┘
                              └──────────────► HTTP/S endpoint, email, SMS…
        one message  →  SNS pushes a COPY to each subscriber
```

> **Key insight**: SNS is **push-based** (it delivers *to* subscribers), the mirror image of SQS,
> which is **pull-based**. SNS doesn't store messages for later polling — if a subscriber can't
> receive, SNS retries per its delivery policy and then gives up (or sends to a DLQ).

---

## 2. Topics and Subscriptions

A **topic** is the named channel publishers send to. A **subscription** binds an endpoint to a
topic. SNS supports many subscription **protocols**:

| Subscriber type | Use case |
|-----------------|----------|
| **SQS** | Fan-out into durable queues (the key pattern, §4) |
| **Lambda** | Trigger serverless processing on each message |
| **HTTP / HTTPS** | Call a webhook / external service |
| **Email / Email-JSON** | Human notifications |
| **SMS** | Text-message alerts |
| **Mobile push** (SNS Mobile Push) | APNs (iOS), FCM (Android) app notifications |
| **Kinesis Data Firehose** | Archive messages to S3/Redshift/OpenSearch |

A single topic can have subscribers of *mixed* protocols at once (e.g., one SQS queue + one Lambda
+ one email). Up to **12.5 million subscriptions per topic**, and **100,000 topics** per account
(soft limits).

⚠️ HTTP/S and email subscriptions require **confirmation** — SNS sends a confirmation request and
the endpoint must opt in before it receives messages.

---

## 3. Message Filtering

By default every subscriber gets every message. A **filter policy** (JSON, attached to a
subscription) lets a subscriber receive only the messages it cares about, based on **message
attributes** (or, optionally, the message **body**). SNS evaluates the policy and only delivers
matching messages — filtering happens *at the topic*, so non-matching messages never reach the
subscriber.

```
   Publish: {attributes: {eventType:"order_placed", region:"us-east-1"}}
                              │
                      ┌───────┴────────┐
                      │   SNS TOPIC    │
                      └───────┬────────┘
       filter {eventType:["order_placed"]} ──► matches  ──► Order queue ✅
       filter {eventType:["order_refund"]} ──► no match ──► Refund queue ❌ (not delivered)
```

✅ Filtering replaces the anti-pattern of "subscribe everyone, then have each consumer drop
irrelevant messages." It reduces consumer load and cost by routing at the source.

💡 This is the simplest form of content-based routing. If you need *richer* routing (matching on
many fields, AWS-service events, scheduling), reach for **EventBridge** (file 04).

---

## 4. The Fan-Out Pattern (SNS → multiple SQS)

The flagship SNS architecture: publish once to a topic that fans out to **multiple SQS queues**,
each feeding an independent consumer. This is *the* exam pattern for "process one event in several
ways simultaneously."

```
                                ┌──► SQS Q1 ──► Email service     (own buffer + DLQ)
   ┌────────┐   ┌─────────┐     │
   │Producer│──►│SNS Topic│─────┼──► SQS Q2 ──► Analytics         (own buffer + DLQ)
   └────────┘   └─────────┘     │
                                └──► SQS Q3 ──► Inventory update  (own buffer + DLQ)
```

Why insert SQS between SNS and each worker rather than subscribing workers directly?

✅ Each queue gives its consumer **durable buffering, independent retry, and its own DLQ**. If the
analytics worker is down for an hour, its messages wait safely in Q2 while email and inventory keep
flowing.

❌ Subscribing a worker *directly* to SNS (e.g., an HTTP endpoint) means a down worker simply
**misses** the message after SNS exhausts retries — no buffering, possible data loss.

⚠️ For SNS to deliver into an SQS queue, the **queue's resource policy** must allow the SNS topic to
`SendMessage`. (The console wires this up; in IaC you set it explicitly.)

---

## 5. FIFO Topics and Ordering

Like SQS, SNS offers **Standard** and **FIFO** topics.

| | Standard topic | FIFO topic |
|---|---|---|
| Delivery | At-least-once, best-effort order | Exactly-once, strict order |
| Throughput | Very high | Limited (≈300 msg/s per topic) |
| Subscribers | All protocols | **SQS FIFO queues only** |
| Ordering | None | Per `MessageGroupId` |
| Name | any | ends in **`.fifo`** |

A **FIFO topic can only fan out to FIFO SQS queues**, and ordering/dedup work the same way as in
SQS FIFO (`MessageGroupId`, `MessageDeduplicationId`). Use FIFO end-to-end (FIFO topic → FIFO
queues) when ordered, deduplicated fan-out matters.

---

## 6. Reliability — DLQ and Encryption

**DLQ** — attach a **dead-letter queue (an SQS queue)** to an SNS *subscription*. If SNS exhausts
its delivery retries to that subscriber (e.g., the HTTP endpoint stays down), the undeliverable
message goes to the DLQ instead of being dropped.

```
   SNS Topic ──deliver──► subscriber (down) ──retries exhausted──► subscription DLQ (SQS)
```

⚠️ The DLQ is per **subscription**, not per topic — each subscriber can have its own.

**Encryption**:
- **In transit** — HTTPS by default.
- **At rest** — **SSE with KMS** (`KmsMasterKeyId` on the topic) encrypts stored messages.
- **Access control** — IAM policies and SNS **topic policies** (resource policies) govern who can
  publish/subscribe; restrict cross-account access here.

---

## 7. SNS+SQS Fan-Out vs EventBridge

Both can deliver one event to many targets. Knowing when to pick which is a frequent exam call.

| | **SNS (+ SQS) fan-out** | **EventBridge** |
|---|---|---|
| Model | Pub/sub topic | Event bus with rules |
| Routing | Subscription filter policies (attributes) | Rich **event-pattern** matching on full content |
| Targets | SQS, Lambda, HTTP, email, SMS, mobile push | 20+ AWS services, API destinations, cross-account |
| AWS service events | No (you publish manually) | ✅ Native source for AWS service events |
| Scheduling (cron) | No | ✅ Scheduled rules |
| Replay/archive | No | ✅ Archive & replay |
| Throughput / latency | Very high, lowest latency | Slightly higher latency |
| Best for | High-throughput app fan-out, mobile/SMS push | Event-driven routing of AWS/SaaS events |

> **Rule of thumb**: choose **SNS+SQS** for high-throughput, low-latency application fan-out (and
> the only option for SMS / mobile push). Choose **EventBridge** when you need content-based
> routing, AWS-service event sources, scheduling, or replay. Details in
> [04_eventbridge.md](04_eventbridge.md).

---

## 8. Key Exam Points

- **SNS = pub/sub**, **push-based**: one publish → a **copy to every subscriber**.
- Subscribers: **SQS, Lambda, HTTP/S, email, SMS, mobile push**, Firehose — mixed per topic.
- **SNS → multiple SQS = fan-out pattern**; SQS between SNS and workers adds buffering, retry, DLQ.
- **Filter policies** route only matching messages to a subscription (attribute-based).
- **FIFO topics** = ordered + exactly-once, fan out **only to FIFO SQS queues**, name ends `.fifo`.
- **DLQ is per subscription**; attach an SQS DLQ to catch undeliverable messages.
- Encrypt at rest with **KMS (SSE)**; control access with **topic (resource) policies**.
- SNS+SQS = high-throughput app fan-out; **EventBridge** = AWS/SaaS event routing + scheduling +
  replay.

---

## 9. Common Mistakes

- ❌ Subscribing workers **directly** to SNS (HTTP) instead of via SQS — a down worker loses
  messages once retries are exhausted.
- ❌ Forgetting the **SQS queue policy** allowing the SNS topic to publish — fan-out silently fails.
- ❌ Expecting SNS to **store** messages for later polling — it pushes and retries, it is not a
  queue.
- ❌ Subscribing a **non-FIFO** SQS queue to a **FIFO** topic (only FIFO queues are allowed).
- ❌ Reaching for SNS when you need **content-based routing on AWS service events** — that's
  EventBridge.
- ❌ Assuming filtering uses the message body by default — it filters on **message attributes**
  unless you enable body-based filtering.

---

**Next**: [04_eventbridge.md — Amazon EventBridge](04_eventbridge.md)
