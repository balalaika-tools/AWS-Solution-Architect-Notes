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
> *targets that match a content pattern*, and it natively understands events emitted by AWS
> services. It's "pub/sub with a rules engine and AWS-service awareness."

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

**Targets** — where matched events go: **Lambda, SQS, SNS, Step Functions, Kinesis Data Streams,
Amazon Data Firehose, ECS tasks, API destinations** (any external HTTP API), other event buses, and
more. Target and rule quotas change over time and some are adjustable; check Service Quotas for the
deployment Region rather than treating a memorized target count as architecture capacity.

✅ EventBridge can match on *any field in the event body*, so it routes on rich content — not just
the attribute-only filtering SNS offers.

---

## 3. EventBridge Scheduler and Scheduled Rules

A rule can fire on a **schedule** instead of an event pattern — replacing cron servers for
serverless scheduling.

```
   rate(5 minutes)          ──► every 5 minutes
   cron(0 8 * * ? *)        ──► every day at 08:00 UTC
                                 └─ targets: Lambda, Step Functions, ECS task, …
```

Scheduled rules are the older, simple recurring option and use UTC with one-minute precision.
For new scheduled workloads, prefer **EventBridge Scheduler**, a dedicated service decoupled from
event buses. It supports one-time, rate, and cron schedules; named time zones and daylight-saving
handling; flexible delivery windows; many AWS API targets; configurable retry; and an SQS DLQ.

Scheduler provides **at-least-once** delivery, so the target must be idempotent. A schedule can retry
for up to 24 hours and 185 attempts when configured; a standard SQS DLQ captures an invocation
after retries are exhausted. Delete completed one-time schedules automatically when they are no
longer needed—invoking once does not otherwise remove the schedule resource.

---

## 4. Schema Registry, Archive & Replay

**Schema registry** — EventBridge can **discover and store the schema** (structure) of events on a
bus, accept custom OpenAPI 3 or JSON Schema Draft 4 contracts, and generate **code bindings** so
developers get typed objects for events. Discovery creates a new schema version when observed
content changes; it does not decide whether the change is backward compatible.

Treat the registry as a catalog, not the governance process. Give every event type an owner, keep a
stable envelope and explicit application schema version, prefer additive changes, and validate
producer/consumer compatibility in CI. Consumers should ignore unknown optional fields, while
producers should not remove or reinterpret fields until all consumers have migrated. Discovery can
also expose sensitive field shapes, so minimize event payloads and control registry access.

**Archive & replay** — you can **archive** events that match a pattern (with a retention period)
and later **replay** a selected time range to the **same source bus**, optionally through selected
rules. This is useful for after-the-fact reprocessing.

```
   events ──► [ archive (retain N days) ] ──replay──► source bus ──► rules/targets
```

✅ Use archive/replay to reprocess events after fixing a bug, or to seed a new consumer with
historical events. Replayed events include `replay-name`, are not guaranteed to arrive in their
original order, and can repeat side effects. Make targets idempotent and use a rule or maintenance
mode to prevent replay from hitting consumers that should not run. An EventBridge archive is not a
substitute for a high-throughput ordered event log such as Kinesis or MSK.

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

## 6. Enterprise Event Architecture

### Cross-account and organization buses

A central or domain event bus can accept events from other AWS accounts without sharing
credentials:

1. The receiving account attaches a **resource policy** to the bus allowing `events:PutEvents` from
   explicit accounts or an AWS Organizations ID. Do not use an unrestricted principal when the
   organization boundary is known.
2. The sending account creates a rule whose target is the destination bus. New cross-account bus
   targets require an IAM role; scope it to the exact destination ARN so SCPs and role policy both
   participate in authorization.
3. Receiving rules match the `account` field as well as `source` and `detail-type`. Without an
   account constraint, any trusted sender whose event matches the rest of the pattern can trigger
   the rule.

Cross-account and supported cross-Region bus targets solve routing, not contract ownership or
encryption permissions at downstream targets. Include the source account and application event ID
in logs, cost allocation, incident response, and replay authorization.

### Central governance without a central bottleneck

A common multi-account model has workload accounts publish domain events to an ingress bus in an
integration account, then routes approved events to domain-owned buses or queues. Guard it with:

- an Organizations-scoped bus policy and least-privilege sender roles;
- standard `source`, `detail-type`, account, schema-version, correlation, and data-classification
  fields;
- infrastructure-as-code review for rules, targets, archives, DLQs, and bus policies;
- a schema catalog plus compatibility tests owned by producer and consumer teams;
- CloudTrail and CloudWatch visibility for policy changes, failed invocations, throttling, DLQs,
  archive/replay operations, and unexpected publishers;
- separate roles for publishing, rule administration, and replay, because replay can re-run
  destructive business actions.

Do not force all traffic through one bus merely for visibility. Separate buses by bounded domain or
trust boundary, apply quotas and alarms per domain, and keep consumer failure isolation with SQS or
another durable target where needed.

### Global endpoints and Regional failover

**EventBridge global endpoints** provide managed failover for **custom events** sent with
`PutEvents`. A Route 53 health check chooses the primary or secondary Region; optional event
replication copies custom events to both Regional buses. Custom buses must have the same name in
both Regions and be in the same account, and publishers must use the global endpoint ID.

Failover occurs on the order of minutes, not instantaneously. Replication is asynchronous, so an
event can be delayed, delivered in both Regions, or processed in the recovered primary later. Make
targets idempotent and deploy equivalent rules, targets, IAM roles, KMS keys, DLQs, and downstream
state in both Regions. Test failover **and failback**, and monitor health-check state plus
replication/processing lag. Global endpoints do not make AWS-service events, partner buses, or all
downstream state globally resilient.

### Ordering, retries, and DLQs

EventBridge event buses provide **at-least-once delivery with no ordering guarantee**. Concurrent
rules, retries, cross-account hops, and replay can all change arrival order or create duplicates.
If a business entity needs order, route to an SQS FIFO queue or Kinesis partition and use the
entity ID as the group/partition key; still make the side effect idempotent.

Target retry and failure handling are configured **per target**. By default, EventBridge retries
retriable delivery failures for up to 24 hours and 185 attempts with exponential backoff and
jitter. After the target's retry policy is exhausted, EventBridge drops the event unless an SQS
DLQ is configured. Alarm on failed invocations, throttles, and DLQ depth, and grant the EventBridge
service permission to send to the DLQ.

A target DLQ catches failure to hand the event to the target. It does not catch a Lambda or
workflow that accepted the invocation and later failed its business logic. For critical work, use
the target service's own failure destination/retry controls or route through SQS.

### EventBridge Pipes

**EventBridge Pipes** creates a point-to-point integration from one supported source to one target,
with optional filtering, transformation, and enrichment. Use a pipe when the requirement is
"consume this queue/stream, enrich matching records, and invoke that target"; use a bus when many
publishers and rules need many-to-many routing. A pipe can also target a bus to combine both.

If the source enforces order, Pipes maintains that order through synchronous target invocation.
An unordered source remains unordered. Give the pipe execution role only the read/checkpoint
permissions for its source and invoke permissions for its enrichment and target, then configure
source-appropriate batching, retries, partial failures, DLQ/on-failure behavior, and logs. Filtering
out a stream record advances the source iterator—it is not retained by Pipes for later delivery.

---

## 7. EventBridge vs SNS vs SQS — When to Use Which

The core decision table. All three are "messaging," but they answer different questions.

| | **SQS** | **SNS** | **EventBridge** |
|---|---|---|---|
| Model | Queue (point-to-point) | Pub/sub (broadcast) | Event bus + rules (routing) |
| Push or pull | **Pull** (poll) | **Push** | **Push** |
| Consumers per message | 1 (competing pool) | N (all subscribers) | N (matching rules/targets) |
| Routing logic | None | Attribute filter policy | Rich **content** event patterns |
| AWS service events | No | No | ✅ Native |
| SaaS sources | No | No | ✅ Partner buses |
| Scheduling | No | No | ✅ EventBridge Scheduler; scheduled rules for legacy/simple recurring jobs |
| Archive / replay | No (retain ≤14d, not replay) | No | ✅ Archive & replay |
| Throughput / latency | Very high | **Highest, lowest latency** | High, slightly higher latency |
| Targets / protocols | Consumers poll | SQS, Lambda, HTTP, email, SMS, push | Many AWS targets, API destinations |
| Best for | Decouple + buffer + retry work | High-throughput fan-out, SMS/mobile push | Event-driven routing of AWS/SaaS events, scheduling |

> **Rule of thumb**:
> - Need to **buffer work** and process once per message → **SQS**.
> - Need **high-throughput fan-out** to many subscribers, or **SMS / mobile push** → **SNS** (often
>   SNS → SQS).
> - Need to **route AWS-service or SaaS events** by content, **schedule** jobs, or **replay**
>   events → **EventBridge**.

---

## 8. Use Cases

- **React to AWS service events** — e.g., on `EC2 stopped` → Lambda cleanup; on S3 object created →
  start a workflow; on a Config rule non-compliance → notify. EventBridge is the natural router.
- **Decouple microservices** — services emit domain events to a custom bus; other services
  subscribe via rules, never calling each other directly.
- **Serverless scheduling** — one-time or recurring work through EventBridge Scheduler.
- **SaaS integration** — ingest partner events without building webhook receivers.
- **Audit / replay** — archive events and replay after incidents.

---

## 9. Key Exam Points

- **EventBridge = serverless event bus + rules engine**; formerly **CloudWatch Events**.
- **Rules** match **event patterns** (rich content matching) or run on a **schedule (cron/rate)**.
- Natively ingests events from AWS services (default bus) and **SaaS partners** (partner
  bus); use **custom buses** for your own app events and cross-account delivery.
- **Schema registry** generates typed bindings; **archive & replay** reprocesses past events
  back to the source bus, without preserving order.
- **Scheduler** is the preferred dedicated scheduler; **Pipes** provides one-source-to-one-target
  filtering/enrichment; buses provide many-to-many routing.
- Event buses do not guarantee ordering. Configure retry and an SQS DLQ per target; make targets
  idempotent.
- Cross-account buses use resource policies + sender roles; global endpoints fail over custom
  `PutEvents` traffic but do not make the entire workload multi-Region.
- Decision: **SQS** = buffer/queue; **SNS** = high-throughput fan-out + SMS/push; **EventBridge** =
  AWS/SaaS event routing, scheduling, replay.

---

## 10. Common Mistakes

- ❌ Building a **cron EC2 server** for scheduled jobs instead of **EventBridge Scheduler**.
- ❌ Using **SNS** when you need to route on **AWS service events** by content — that's EventBridge.
- ❌ Expecting AWS-service events on a **custom bus** — they arrive on the **default** bus.
- ❌ Confusing EventBridge with **Kinesis** for streaming/replay of high-volume data — EventBridge
  routes discrete events, not high-throughput record streams (see [05_kinesis.md](05_kinesis.md)).
- ❌ Treating SNS attribute filtering and EventBridge event patterns as equivalent — EventBridge
  matches on **full event content**, far richer than SNS.
- ❌ Assuming archive replay preserves order or invokes only the repaired consumer; replay returns
  events to the source bus and rules match again.
- ❌ Omitting `account` from rules on a cross-account bus and unintentionally accepting matching
  events from every trusted publisher.
- ❌ Treating a target DLQ as business-processing error handling—it captures failed delivery to the
  target, not every failure after a target accepts an invocation.
- ❌ Enabling a global endpoint without deploying equivalent targets/state or making consumers
  duplicate-safe in the secondary Region.

---

**Next**: [05_kinesis.md — Amazon Kinesis](05_kinesis.md)
