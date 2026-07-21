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
| **Amazon Data Firehose** | Archive messages to S3/Redshift/OpenSearch |

A single topic can have subscribers of *mixed* protocols at once (e.g., one SQS queue + one Lambda
+ one email). Topic, subscription, and publish quotas vary by Region and some are adjustable; check
Service Quotas instead of designing from a memorized global number.

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
| Delivery | At-least-once, best-effort order | Ordered and deduplicated within a message group under documented conditions |
| Throughput | High; account/Regional publish quotas apply | 300 msg/s per group; topic/account quota depends on throughput scope and Region |
| Subscribers | All supported protocols | Amazon SQS **FIFO or standard** queues |
| Ordering | None | Per `MessageGroupId` |
| Name | any | ends in **`.fifo`** |

A FIFO topic can fan out to SQS FIFO and standard queues. Use FIFO end-to-end (FIFO topic → FIFO
queue) when order matters; a standard queue subscription deliberately gives up ordering and may
deliver duplicates. `MessageGroupId` defines the ordering lane and `MessageDeduplicationId` (or a
content hash) suppresses a repeated publish during the five-minute deduplication window.

With `FifoThroughputScope=Topic`, deduplication is topic-wide and the default per-topic ceiling is
3,000 messages/s or 20 MB/s, whichever is reached first. With `MessageGroup` scope, deduplication
and throughput are per group and the topic can scale toward its Regional account quota. Every
single group remains limited to 300 messages/s, so partition by the smallest business entity that
needs order.

AWS describes exactly-once FIFO delivery only under specific end-to-end conditions: an authorized
SQS FIFO subscription, no subscription filtering, successful acknowledgement, and no preventing
network disruption. Filtering makes delivery at-most-once for filtered messages. Consumers should
still make business side effects idempotent because a crash after committing work but before
deleting the SQS message can cause reprocessing.

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
- **In transit** — publish through TLS endpoints and prefer HTTPS subscriptions over HTTP.
- **At rest** — **SSE with KMS** (`KmsMasterKeyId` on the topic) encrypts stored messages.
- **Access control** — IAM policies and SNS **topic policies** (resource policies) govern who can
  publish/subscribe; restrict cross-account access here.

---

## 7. Production Fan-Out Design

### Cross-account SNS to SQS

Cross-account fan-out needs authorization on both resources:

1. The topic policy allows the queue-owning account to call `sns:Subscribe` if that account creates
   the subscription. Having the queue owner subscribe is preferred because confirmation is
   automatic; a subscription created by a different account must be confirmed.
2. The SQS queue policy allows the `sns.amazonaws.com` service principal to call
   `sqs:SendMessage`, restricted with `aws:SourceArn` to the exact topic and, where appropriate,
   `aws:SourceAccount`.
3. The deployment identities need their own least-privilege IAM permissions to create or manage
   the subscription. Resource policies do not grant an operator every management action.

Apply the same pattern for each queue rather than using a wildcard topic principal. Keep queue and
topic ownership explicit so that deleting a subscription, changing a queue policy, or rotating a
key has a named team and an alarm.

### Retry, DLQ, and delivery status by protocol

SNS retry behavior belongs to the **subscription protocol**:

| Endpoint | Retry control | Failure capture |
|----------|---------------|-----------------|
| **SQS / Lambda** | AWS-managed policy; server-side failures are retried for an extended period | Attach an SQS DLQ to each subscription and alarm on it. |
| **HTTP/S** | A delivery policy can tune backoff and average receive throttling; total retry policy time cannot exceed one hour | Use a subscription DLQ because the endpoint itself is not a durable buffer. |
| **Email, SMS, mobile push** | AWS-managed retry policy; the application cannot customize it | Use delivery/status features supported by that protocol and design an alternate notification path where loss matters. |
| **Data Firehose** | Protocol-defined behavior; throttling uses customer-managed-endpoint retry behavior | Configure a subscription DLQ and monitor delivery failures. |

SNS does not retry permanent client-side failures such as a deleted endpoint or a policy that
denies delivery. A subscription DLQ catches messages that cannot be delivered; it does not catch a
subscriber that accepted a message and then failed its own business processing. Put SQS between
SNS and application workers when that processing needs durable retry.

Enable **delivery status logging** to CloudWatch Logs for supported endpoints (Data Firehose, SQS,
Lambda, HTTPS, and platform application endpoints). Log failures and sample successes; use dwell
time and provider responses to distinguish permission errors, throttling, and target outages.
Grant SNS a narrowly scoped log-writing role and alarm on failure metrics as well as DLQ depth.

### KMS and sensitive data

For an encrypted topic using a customer-managed KMS key:

- A publisher needs `kms:GenerateDataKey*` and `kms:Decrypt` for that key in addition to
  `sns:Publish`.
- An AWS service publishing to the topic must be allowed in the key policy. Where the integration
  supports them, use `aws:SourceArn`/`aws:SourceAccount` conditions to prevent confused-deputy use.
- If the destination SQS queue or the subscription DLQ is encrypted, its KMS key policy must allow
  the SNS service to use the key. Topic encryption does not configure destination encryption.
- KMS request cost and key-policy failure are part of the operational design; alarm on publish and
  delivery failures after key changes.

SNS **message data protection policies** can audit, mask/redact, or deny sensitive data on inbound
or outbound messages for existing eligible customers. This feature is **no longer available to new
customers** and applies to standard topics, so do not make it a universal architecture dependency.
Existing users should monitor findings and test deny/de-identify behavior; new designs should
classify and minimize payloads at the producer and enforce equivalent DLP controls at data
boundaries. Encryption protects storage, not inappropriate data propagation to every subscriber.

### Multi-Region publishing and failover

SNS topics, policies, subscriptions, and KMS keys are Regional. SNS supports cross-Region delivery
to SQS and Lambda, but that is delivery from one Regional topic—not topic replication or managed
publisher failover. A Region-resilient design normally deploys a topic and subscriptions in each
Region and routes publishers to the healthy Regional endpoint.

Choose and test one strategy:

- **Active/passive** — publish to the primary topic, fail over producers, and accept the documented
  RPO for messages that were not republished.
- **Active/active or dual publish** — publish to both Regional topics and deduplicate at consumers
  using a stable application event ID. This improves RPO but duplicates traffic, cost, and side
  effects unless consumers reconcile correctly.
- **Cross-Region subscriptions** — useful for selected SQS/Lambda targets, but the source topic is
  still a Regional dependency and opt-in Regions require the correct regionalized service
  principal in the destination policy.

Regional recovery must include subscriber compute, queues/DLQs, customer-managed keys, secrets,
quotas, and failback. A second topic alone is not a disaster-recovery plan.

---

## 8. SNS+SQS Fan-Out vs EventBridge

Both can deliver one event to many targets. Knowing when to pick which is a frequent exam call.

| | **SNS (+ SQS) fan-out** | **EventBridge** |
|---|---|---|
| Model | Pub/sub topic | Event bus with rules |
| Routing | Subscription filter policies (attributes) | Rich **event-pattern** matching on full content |
| Targets | SQS, Lambda, HTTP, email, SMS, mobile push | Many AWS services, API destinations, cross-account |
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

## 9. Key Exam Points

- **SNS = pub/sub**, **push-based**: one publish → a **copy to every subscriber**.
- Subscribers: **SQS, Lambda, HTTP/S, email, SMS, mobile push**, Data Firehose — mixed per topic.
- **SNS → multiple SQS = fan-out pattern**; SQS between SNS and workers adds buffering, retry, DLQ.
- **Filter policies** route only matching messages to a subscription (attribute-based).
- **FIFO topics** preserve order/dedup per message group; they can target FIFO or standard SQS
  queues, but only FIFO end-to-end preserves FIFO behavior. Consumers remain idempotent.
- **DLQ is per subscription**; attach an SQS DLQ to catch undeliverable messages.
- Cross-account fan-out needs topic + queue policies and, when encrypted, KMS key permissions.
- Delivery status logs expose handoff failures; retries differ by protocol.
- SNS is Regional and has no managed topic failover; duplicate-safe dual publish is one DR option.
- SNS+SQS = high-throughput app fan-out; **EventBridge** = AWS/SaaS event routing + scheduling +
  replay.

---

## 10. Common Mistakes

- ❌ Subscribing workers **directly** to SNS (HTTP) instead of via SQS — a down worker loses
  messages once retries are exhausted.
- ❌ Forgetting the **SQS queue policy** allowing the SNS topic to publish — fan-out silently fails.
- ❌ Expecting SNS to **store** messages for later polling — it pushes and retries, it is not a
  queue.
- ❌ Subscribing a standard SQS queue to a FIFO topic and expecting FIFO order—the subscription is
  valid, but the standard queue provides best-effort order and at-least-once delivery.
- ❌ Reaching for SNS when you need **content-based routing on AWS service events** — that's
  EventBridge.
- ❌ Assuming filtering uses the message body by default — it filters on **message attributes**
  unless you enable body-based filtering.
- ❌ Enabling topic SSE but omitting publisher/service permissions in the KMS key policy.
- ❌ Treating cross-Region delivery as multi-Region topic failover or assuming dual publish cannot
  duplicate a business action.
- ❌ Designing a new workload around SNS message data protection without checking availability; it
  is no longer offered to new customers.

---

**Next**: [04_eventbridge.md — Amazon EventBridge](04_eventbridge.md)
