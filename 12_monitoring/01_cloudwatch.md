# Amazon CloudWatch — Metrics, Logs, Alarms & Dashboards

> **Who this is for**: Engineers prepping for SAA-C03 who can launch EC2/Lambda/RDS but
> haven't formalized *how* to observe them. Assumes you understand [Auto Scaling](../07_ha_scaling/README.md)
> and [SNS](../11_messaging/README.md), since alarms drive both.

---

## 1. Prerequisite: What Observability Means

Before any AWS service, get the vocabulary straight. **Observability** is your ability to
answer "what is the system doing right now, and why?" from the data it emits. There are
three signal types — the "three pillars":

| Pillar | What it is | Question it answers | AWS home |
|--------|-----------|---------------------|----------|
| **Metrics** | Numeric time-series sampled at intervals (CPU %, request count) | *Is it healthy? Trending up or down?* | CloudWatch Metrics |
| **Logs** | Timestamped text/JSON records of discrete events | *What exactly happened at 14:03?* | CloudWatch Logs |
| **Traces** | The path of one request across services, with timing per hop | *Where is the latency / failure?* | AWS X-Ray |

> **Key insight**: Metrics tell you *something is wrong*; logs and traces tell you *why*.
> CloudWatch owns metrics and logs; **X-Ray** owns traces. Don't confuse this with
> **CloudTrail** (an *audit* log of API calls — covered in the next file) — CloudWatch is
> about *performance and health*, not *who did what*.

CloudWatch is AWS's native monitoring service. Nearly every AWS service publishes metrics
to it automatically and for free; you add logs, alarms, and dashboards on top.

---

## 2. Metrics: Namespaces, Dimensions, Resolution

A **metric** is a time-ordered set of data points (e.g., `CPUUtilization`). Metrics are
organized into **namespaces** — a container that isolates one service's metrics from
another's. AWS namespaces look like `AWS/EC2`, `AWS/Lambda`, `AWS/RDS`; your own use any
string (e.g., `MyApp/Orders`).

A **dimension** is a name/value pair that scopes a metric to a specific resource. The same
`CPUUtilization` metric exists per instance, distinguished by the `InstanceId` dimension.

```
Namespace: AWS/EC2
  Metric: CPUUtilization
    Dimension: InstanceId = i-0abc...   →  [data points over time]
    Dimension: InstanceId = i-0def...   →  [data points over time]
    Dimension: AutoScalingGroupName = web-asg  →  [aggregated]
```

> **Rule**: A metric is uniquely identified by **namespace + name + the full set of
> dimensions**. Querying `CPUUtilization` with no dimension returns nothing useful —
> dimensions are not optional filters layered on a single series; each combination is its
> own series.

### Standard vs custom metrics

| | Standard metrics | Custom metrics |
|---|---|---|
| Source | Published by AWS automatically | You push via `PutMetricData` / the agent |
| Examples | EC2 CPU, ELB request count, S3 bucket size | App-level: queue depth, logged-in users, **EC2 memory/disk** |
| Cost | Free | Billed per metric |

⚠️ **Exam classic — EC2 memory and disk usage are NOT standard metrics.** The hypervisor
can see CPU, network, and disk *I/O*, but it cannot see *inside* the guest OS. To get
**memory utilization** or **disk space used**, you must install the **CloudWatch Agent**
(see §8), which publishes them as custom metrics.

### Resolution: basic vs detailed monitoring

| Mode | Granularity | Cost | Default for |
|------|------------|------|-------------|
| **Basic monitoring** | 1 data point per **5 minutes** | Free | EC2 (default) |
| **Detailed monitoring** | 1 data point per **1 minute** | Paid (enable per-instance) | Opt-in |
| **High-resolution custom** | Down to **1 second** | Paid | Custom `PutMetricData` only |

💡 Faster scaling reactions (e.g., responsive Auto Scaling) need **detailed monitoring** so
alarms see 1-minute data instead of waiting 5 minutes.

---

## 3. Alarms: States and Actions

A **CloudWatch alarm** watches a single metric (or a metric-math expression) and changes
state when the value breaches a threshold for a configured number of periods.

```
   datapoints crossing threshold
   ─────────────────────────────►
        OK  ──breach──►  ALARM  ──recovers──►  OK
         │                 │
         └── INSUFFICIENT_DATA (not enough data yet / metric stopped reporting)
```

| Alarm state | Meaning |
|-------------|---------|
| `OK` | Metric is within threshold |
| `ALARM` | Threshold breached for the required periods |
| `INSUFFICIENT_DATA` | Alarm just started, metric stopped, or data is missing |

An alarm can trigger one or more **actions** on any state transition:

| Action target | Use case |
|----------------|----------|
| **SNS topic** | Notify (email/SMS/Lambda/SQS) — most common |
| **Auto Scaling policy** | Scale an ASG out/in based on load — see cross-link below |
| **EC2 action** | Stop / terminate / reboot / recover a *single* instance |
| **Systems Manager** | OpsItem / incident creation |

```
CPUUtilization > 70% for 3 periods
        │
     ALARM
        ├──► SNS topic  ──► email the on-call
        └──► ASG scale-out policy  ──► add 2 instances
```

> **Hands-on**: The full alarm-driven scaling walkthrough lives in
> [Practical Examples — CloudWatch Alarm Scaling](../18_practical_examples/16_cloudwatch_alarm_scaling.md).

⚠️ **EC2 recover vs reboot**: the *recover* action only works for **system-status-check**
failures (failed underlying hardware) and migrates the instance to new hardware keeping the
same IP/EBS. It does *not* fix application crashes — that's *reboot* or an ASG replacement.

### Composite alarms

A **composite alarm** combines other alarms with boolean logic (`AND`/`OR`/`NOT`):

```
ALARM(HighCPU) AND ALARM(HighLatency)  →  composite "ServiceDegraded" fires
```

💡 Composite alarms reduce **alarm noise / notification storms**. Instead of paging on every
single CPU blip, you page only when several conditions are true together. They also let one
SNS notification represent many underlying alarms.

---

## 4. CloudWatch Logs

Logs are organized hierarchically:

```
Log Group  (e.g., /aws/lambda/order-processor — one per app/service, where retention is set)
  └── Log Stream  (one sequence of events from a single source, e.g., one Lambda instance)
        └── Log Events  (timestamp + raw message)
```

- **Retention** is configured on the **log group**. Default is *Never expire* — ⚠️ a common
  cost trap. Set retention (e.g., 30/90 days) deliberately.
- Sources: the **CloudWatch Agent** (EC2/on-prem), **Lambda** (automatic), ECS, API Gateway,
  Route 53, VPC **Flow Logs**, etc.

### Metric filters

A **metric filter** scans incoming log events for a pattern and converts matches into a
**CloudWatch metric** you can alarm on.

```
Log group: /var/log/app
  Filter pattern: "ERROR"
        │  (matches → increment metric)
        ▼
  Custom metric: AppErrors  ──►  Alarm: AppErrors > 10 / 5min  ──► SNS
```

> **Exam pattern**: "Alert when the application logs more than N errors." → metric filter
> on the log group → alarm on the resulting metric. Metric filters are **not retroactive** —
> they only act on events ingested *after* the filter is created.

### Subscription filters

A **subscription filter** streams matching log events in **near real time** to a
destination for further processing:

| Destination | Why |
|-------------|-----|
| **Kinesis Data Streams / Firehose** | Pipe to S3, OpenSearch, Splunk, analytics |
| **Lambda** | Custom processing / reformatting |
| **Cross-account / cross-Region** | Centralized logging account |

> **Metric filter vs subscription filter**: a metric filter produces a *number to alarm on*;
> a subscription filter produces a *stream of log events to send somewhere else*.

### Logs Insights

**CloudWatch Logs Insights** is an interactive query language for ad-hoc log analysis —
no pre-defined filters needed.

```sql
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as errors by bin(5m)
| sort errors desc
```

💡 Logs Insights is for *exploration after the fact* ("why did latency spike at 2pm?").
Metric filters are for *standing alerts*. Both read the same log groups.

---

## 5. Dashboards

**CloudWatch dashboards** are customizable visual home-pages of metrics and alarms.

- **Global / cross-Region and cross-account** — a single dashboard can show metrics from
  multiple Regions and multiple accounts (a key exam differentiator).
- Widgets: line/stacked graphs, numbers, gauges, alarm status, text, and **Logs Insights**
  query results.
- Billed per dashboard beyond the free tier.

---

## 6. CloudWatch and EventBridge

**EventBridge** (formerly *CloudWatch Events*) is the event-routing service. Historically
part of CloudWatch, it's now branded separately but still tightly related.

```
Event source (AWS service state change, schedule, or custom app event)
        │
   EventBridge rule (event pattern OR cron/rate schedule)
        │
   Target: Lambda, SNS, SQS, Step Functions, ECS task, ...
```

- **Scheduled rules** (cron/rate) replace the old "CloudWatch scheduled events" — use these
  for "run this Lambda every hour."
- **Event-pattern rules** react to things like *EC2 instance entered `stopped`* or
  *S3 object created*.

> **CloudWatch Alarm vs EventBridge rule**: an **alarm** reacts to a *metric crossing a
> threshold*; an **EventBridge rule** reacts to a *discrete event or a schedule*. "Run a
> Lambda nightly" = EventBridge schedule, not an alarm.

See [Messaging & Events](../11_messaging/README.md) for EventBridge in depth.

---

## 7. The CloudWatch Agent

The **CloudWatch Agent** is software you install on EC2 (and on-prem servers) to push data
the hypervisor can't see:

- **Custom metrics**: memory, swap, disk space, per-process stats.
- **Logs**: application and system log files → CloudWatch Logs.

```
EC2 instance
  ┌──────────────────────────────┐
  │  Application                 │
  │  /var/log/app.log ────────┐  │
  │  OS: memory, disk used     │ │
  └────────────────────────────┼─┘
                CloudWatch Agent│  (needs an IAM role: CloudWatchAgentServerPolicy)
                                ▼
              CloudWatch Metrics + CloudWatch Logs
```

⚠️ The agent needs an **IAM role/instance profile** granting `cloudwatch:PutMetricData` and
logs permissions (managed policy `CloudWatchAgentServerPolicy`). The legacy "CloudWatch Logs
agent" is deprecated — the **unified CloudWatch Agent** is the current answer.

💡 Mnemonic for the exam: **"Memory metric? → install the agent."**

---

## 8. Synthetics Canaries & the X-Ray Pointer

**CloudWatch Synthetics canaries** are configurable scripts (Node.js/Python on a Lambda
runtime) that run on a schedule to probe your endpoints from the *outside* — like a synthetic
user. They measure availability and latency and can validate page content or a full API
workflow *before real users hit a problem*.

> **Real-user vs synthetic**: canaries generate *synthetic* traffic on a schedule. They
> complement, not replace, real-user metrics. "Continuously verify the login flow works
> even when no one is using it" → Synthetics canary.

**ServiceLens** stitches together metrics, logs, and **X-Ray traces** into a service map so
you can see end-to-end request flow and pinpoint the failing/slow component. X-Ray itself is
covered with serverless/microservices tracing; for SAA-C03 just remember:

> **X-Ray = distributed tracing** (the third pillar) — *where* in a microservice/serverless
> chain the latency or error is. CloudWatch metrics say *something* is slow; X-Ray says
> *which hop*.

---

## Key Exam Points

- ✅ **EC2 memory & disk usage require the CloudWatch Agent** — they are custom metrics, not
  standard. The #1 CloudWatch trap.
- ✅ **Basic = 5-minute**, **Detailed = 1-minute**, high-resolution custom down to 1 second.
  Faster scaling → enable detailed monitoring.
- ✅ Alarm states: `OK`, `ALARM`, `INSUFFICIENT_DATA`. Actions target **SNS**, **Auto
  Scaling**, **EC2** (stop/terminate/reboot/recover), and SSM.
- ✅ **Composite alarms** reduce notification noise by combining alarms with AND/OR/NOT.
- ✅ **Metric filter** → turn a log pattern into a metric to alarm on. **Subscription
  filter** → stream logs in real time to Kinesis/Lambda. **Logs Insights** → ad-hoc queries.
- ✅ Log **retention is set on the log group** and defaults to *never expire* (cost trap).
- ✅ Dashboards can be **cross-Region and cross-account**.
- ✅ **EventBridge schedule** for "run on a timer"; **alarm** for "metric crossed a threshold."
- ✅ **Synthetics canaries** = scheduled outside-in probes; **X-Ray/ServiceLens** = tracing.

---

## Common Mistakes

- ❌ Assuming EC2 memory utilization is available out of the box. It is not — install the agent.
- ❌ Picking a CloudWatch alarm to "run a Lambda every night." That's an **EventBridge
  scheduled rule**.
- ❌ Confusing CloudWatch with CloudTrail. CloudWatch = *performance/health*; CloudTrail =
  *who made which API call* (next file).
- ❌ Expecting a metric filter to backfill old logs. Filters only act on events ingested
  *after* creation.
- ❌ Leaving log groups on default *never expire* retention and being surprised by the bill.
- ❌ Expecting the EC2 *recover* action to fix an app crash — it only handles system (hardware)
  status-check failures.

---

**Next**: [02_cloudtrail.md — AWS CloudTrail: Auditing Every API Call](02_cloudtrail.md)
