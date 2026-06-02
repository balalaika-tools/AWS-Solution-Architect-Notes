# Amazon EventBridge — The Serverless Event Bus

> **Who this is for**: Engineers who know **pub/sub** ([01](01_messaging_concepts.md)) and **SNS**
> ([03_sns.md](03_sns.md)) and now need the AWS service for **routing events** — especially events
> *emitted by AWS services themselves*. The **EventBridge vs SNS vs SQS** decision table is a
> recurring SAA-C03 question.

---

## 1. What EventBridge Is

**Amazon EventBridge** (formerly **CloudWatch Events** — same underlying service, expanded) is a
serverless **event bus**. Events flow onto a bus; **rules** match events against **event patterns**
and route matches to one or more **targets**. It's the backbone of event-driven architectures on
AWS.

```
   event sources                    EventBridge bus                 targets
   ┌──────────────┐               ┌──────────────────┐         ┌──────────────┐
   │ AWS services │──┐            │  RULE 1 (pattern) │────────►│ Lambda       │
   │ (S3, EC2, …) │  │   events   │──────────────────┤────────►│ SQS / SNS    │
   ├──────────────┤  ├──────────►│  RULE 2 (pattern) │────────►│ Step Funcs   │
   │ Your app     │  │            │──────────────────┤────────►│ Kinesis      │
   ├──────────────┤  │            │  RULE 3 (cron)    │────────►│ API dest.    │
   │ SaaS partners│──┘            └──────────────────┘         └──────────────┘
   └──────────────┘
```

> **Key insight**: SNS broadcasts a message to *subscribers*; EventBridge **routes** an event to
> *targets that match a content pattern*, and it natively understands events emitted by **90+ AWS
> services**. It's "pub/sub with a rules engine and AWS-service awareness."

---

## 2. Event Buses, Rules, Patterns, Targets

**Event** — a JSON document describing something that happened (e.g., an EC2 instance changed
state, an object was created in S3, your app emitted `OrderPlaced`).

**Event bus** — the pipe events flow onto. Three kinds:
- **Default bus** — receives events from AWS services in your account automatically.
- **Custom bus** — you create one for your own application events (isolation per domain/app).
- **Partner bus** — receives events from SaaS partners (§5).

**Rule** — matches events via an **event pattern** (or runs on a **schedule**, §4) and sends
matches to targets.

**Event pattern** — JSON that matches on event fields (content-based routing — far richer than
SNS attribute filtering):

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": { "state": ["stopped", "terminated"] }
}
```

**Targets** — where matched events go: **Lambda, SQS, SNS, Step Functions, Kinesis Data
Streams/Firehose, ECS tasks, API destinations** (any external HTTP API), other event buses, and
more — **20+ AWS service targets**. A single rule can have **up to 5 targets**.

✅ EventBridge can match on *any field in the event body*, so it routes on rich content — not just
the attribute-only filtering SNS offers.

---

## 3. Scheduled Rules (Cron / Rate)

A rule can fire on a **schedule** instead of an event pattern — replacing cron servers for
serverless scheduling.

```
   rate(5 minutes)          ──► every 5 minutes
   cron(0 8 * * ? *)        ──► every day at 08:00 UTC
                                 └─ targets: Lambda, Step Functions, ECS task, …
```

Common use: trigger a Lambda nightly, run a periodic ECS batch task, snapshot resources on a
schedule.

💡 For richer scheduling (one-time schedules, time zones, flexible time windows, millions of
schedules) AWS now offers **EventBridge Scheduler**, a dedicated scheduler decoupled from the bus.
For the exam, "EventBridge cron rule" is the standard answer for serverless scheduled jobs.

---

## 4. Schema Registry, Archive & Replay

**Schema registry** — EventBridge can **discover and store the schema** (structure) of events on a
bus, and generate **code bindings** so developers get typed objects for events. Useful for
governing event contracts across teams.

**Archive & replay** — you can **archive** events that match a pattern (with a retention period)
and later **replay** them to a bus/target. This is unique among the AWS messaging services for
*after-the-fact* reprocessing.

```
   events ──► [ archive (retain N days) ] ──replay──► bus ──► targets reprocess
```

✅ Use archive/replay to reprocess events after fixing a bug, or to seed a new consumer with
historical events. SNS and SQS cannot replay past messages.

---

## 5. SaaS Partner Event Sources & Custom Buses

**Partner event sources** — SaaS providers (e.g., Datadog, PagerDuty, Zendesk, Shopify) can push
events directly into EventBridge via a **partner event bus**, with no polling or webhook
infrastructure on your side. You match and route those events like any other.

**Custom buses** — create separate buses per application or domain to isolate event traffic, apply
distinct permissions (resource policies on the bus, including **cross-account** event delivery),
and keep blast radius small.

```
   SaaS partner ──► partner event bus ──► rules ──► your targets
   App A events ──► custom bus "app-a" ──► rules (isolated from app B)
```

⚠️ AWS service events land on the **default** bus. Your own and partner events use **custom** /
**partner** buses — don't expect to filter AWS-service events off a custom bus.

---

## 6. EventBridge vs SNS vs SQS — When to Use Which

The core decision table. All three are "messaging," but they answer different questions.

| | **SQS** | **SNS** | **EventBridge** |
|---|---|---|---|
| Model | Queue (point-to-point) | Pub/sub (broadcast) | Event bus + rules (routing) |
| Push or pull | **Pull** (poll) | **Push** | **Push** |
| Consumers per message | 1 (competing pool) | N (all subscribers) | N (matching rules/targets) |
| Routing logic | None | Attribute filter policy | Rich **content** event patterns |
| AWS service events | No | No | ✅ Native (90+ services) |
| SaaS sources | No | No | ✅ Partner buses |
| Scheduling (cron) | No | No | ✅ Scheduled rules |
| Archive / replay | No (retain ≤14d, not replay) | No | ✅ Archive & replay |
| Throughput / latency | Very high | **Highest, lowest latency** | High, slightly higher latency |
| Targets / protocols | Consumers poll | SQS, Lambda, HTTP, email, SMS, push | 20+ AWS targets, API destinations |
| Best for | Decouple + buffer + retry work | High-throughput fan-out, SMS/mobile push | Event-driven routing of AWS/SaaS events, scheduling |

> **Rule of thumb**:
> - Need to **buffer work** and process once per message → **SQS**.
> - Need **high-throughput fan-out** to many subscribers, or **SMS / mobile push** → **SNS** (often
>   SNS → SQS).
> - Need to **route AWS-service or SaaS events** by content, **schedule** jobs, or **replay**
>   events → **EventBridge**.

---

## 7. Use Cases

- **React to AWS service events** — e.g., on `EC2 stopped` → Lambda cleanup; on S3 object created →
  start a workflow; on a Config rule non-compliance → notify. EventBridge is the natural router.
- **Decouple microservices** — services emit domain events to a custom bus; other services
  subscribe via rules, never calling each other directly.
- **Serverless scheduling** — nightly Lambda / periodic ECS task via a cron rule.
- **SaaS integration** — ingest partner events without building webhook receivers.
- **Audit / replay** — archive events and replay after incidents.

---

## 8. Key Exam Points

- **EventBridge = serverless event bus + rules engine**; formerly **CloudWatch Events**.
- **Rules** match **event patterns** (rich content matching) or run on a **schedule (cron/rate)**.
- Natively ingests events from **90+ AWS services** (default bus) and **SaaS partners** (partner
  bus); use **custom buses** for your own app events and cross-account delivery.
- A rule has **up to 5 targets**; 20+ AWS service targets plus **API destinations** (external HTTP).
- **Schema registry** generates typed bindings; **archive & replay** reprocesses past events
  (unique vs SNS/SQS).
- Decision: **SQS** = buffer/queue; **SNS** = high-throughput fan-out + SMS/push; **EventBridge** =
  AWS/SaaS event routing, scheduling, replay.

---

## 9. Common Mistakes

- ❌ Building a **cron EC2 server** for scheduled jobs instead of an **EventBridge scheduled rule**.
- ❌ Using **SNS** when you need to route on **AWS service events** by content — that's EventBridge.
- ❌ Expecting AWS-service events on a **custom bus** — they arrive on the **default** bus.
- ❌ Confusing EventBridge with **Kinesis** for streaming/replay of high-volume data — EventBridge
  routes discrete events, not high-throughput record streams (see [05_kinesis.md](05_kinesis.md)).
- ❌ Treating SNS attribute filtering and EventBridge event patterns as equivalent — EventBridge
  matches on **full event content**, far richer than SNS.

---

**Next**: [05_kinesis.md — Amazon Kinesis](05_kinesis.md)
