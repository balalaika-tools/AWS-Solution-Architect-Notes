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

### Treating missing data

When a metric stops reporting (a stopped instance, an idle metric that only emits on
activity), the alarm has to decide what to do with the gaps. You configure this per alarm:

| Setting | Missing data points are treated as… | Effect |
|---------|-------------------------------------|--------|
| `missing` (**default**) | Unknown | If the evaluation window has too few real data points, the alarm moves to `INSUFFICIENT_DATA` |
| `notBreaching` | Within threshold | Pushes toward `OK` |
| `breaching` | Out of threshold | Pushes toward `ALARM` |
| `ignore` | Not evaluated | The current alarm state is retained |

> **Exam pattern**: an alarm flips to `INSUFFICIENT_DATA` (or won't fire) when a low-traffic
> metric simply has no data points. The fix is usually **`treatMissingData`** — e.g., set
> `notBreaching` so an idle queue/endpoint reads as healthy instead of unknown.

⚠️ Watch the interaction with the EC2 *recover/stop* actions: if you stop an instance, its
metrics go missing and the alarm can react in ways you didn't intend unless missing data is
handled deliberately.

### Anomaly detection alarms

Instead of a fixed number, an **anomaly-detection alarm** has CloudWatch learn a metric's
normal **band** (from historical patterns, including daily/weekly seasonality) and alarms
when a value falls outside the band.

```
   value
     │        ┌─ expected band (model) ─┐
     │   ●●●●●│●●●●        ●●●●●●●●●●●●●●│●●●●
     │        │                  ● ←────│── outside band → ALARM
     └────────┴───────────────────────┴──────► time
```

💡 Use anomaly detection when a **static threshold doesn't fit** — e.g., traffic that is
naturally high during the day and low at night, where a single "CPU > 70%" line would either
false-alarm at night or miss daytime problems.

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
| **Kinesis Data Streams / Amazon Data Firehose** | Pipe to S3, OpenSearch, Splunk, analytics |
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
  for simple "run this Lambda every hour" patterns. **EventBridge Scheduler** is the newer,
  dedicated scheduler for one-time schedules, time zones, flexible windows, and very large numbers
  of schedules.
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

## 9. Organization-Wide Observability Architecture

A monitoring account needs two different capabilities, and they should not be confused:

- **Shared visibility** lets operators query telemetry that remains in workload accounts.
- **Centralized copies or streams** move selected telemetry into an account or external system with
  its own retention, security, and processing controls.

Use a dedicated observability account rather than the Organizations management account:

```
Workload accounts, each Region
  metrics / logs / traces / SLOs
          │
          ├── OAM links ────────────────────────────────────────────┐
          │                                                        │
          ├── Logs centralization / subscriptions ───► central log groups │
          │                                                        │
          └── Metric streams ──► Data Firehose ──► S3 / vendor      │
                                                                   ▼
                                                    Observability account
                                                    dashboards / alarms / investigation
```

### Share telemetry with OAM links and sinks

**CloudWatch Observability Access Manager (OAM)** implements CloudWatch cross-account
observability. In each selected Region:

1. Create a **sink** in the monitoring account and attach a sink policy that permits links from the
   intended organization, OUs, or account IDs.
2. Create a **link** in each source account and select what it shares. CloudFormation can roll links
   out across an organization or OU and cover new accounts consistently.
3. Query the linked metrics, log groups, traces, Application Signals services/SLOs, Application
   Insights applications, and internet monitors from the monitoring account.

OAM is **regional**: repeat the sink/link design in every Region in scope. It shares access rather
than copying telemetry, so the source account still owns the data, retention, and ingestion bill.
A link is not a backup and does not preserve logs after their source retention expires.

### Centralize or subscribe logs only when a copy/stream is required

| Need | Pattern | Important boundary |
|------|---------|--------------------|
| Search source log groups centrally without duplicating them | **OAM** | Retention and encryption remain in each source account |
| Keep an organization-controlled CloudWatch Logs copy | **CloudWatch Logs organization centralization rule** | Copies only new data after the rule is created; destination ingestion/storage and optional backup-Region copies add cost |
| Feed a data lake, SIEM, or custom processor in near real time | **Account-level subscription filter** → cross-account Logs destination → Kinesis Data Streams, Data Firehose, or Lambda | Size downstream capacity, monitor delivery errors, and retain a replay/archive path where loss is unacceptable |

For a subscription design, create the logical Logs destination and destination resource policy in
the receiving account. Restrict senders with the Organizations ID, then deploy an account-level
subscription policy in source accounts so new log groups are included automatically. The log group
and CloudWatch Logs destination are in the same Region, although the resource behind the destination
can deliver elsewhere.

Exclude the central pipeline's own delivery log groups from account-level filters. Otherwise it can
subscribe to its own errors and create an ingestion and billing loop. For Kinesis, use `Random`
distribution when one busy log stream could overload a shard, and alarm on throttling or forwarding
errors. Subscription filters are forward-looking; export or separately process historical logs.

### Stream metrics and build cross-account dashboards

A **metric stream** continuously sends selected CloudWatch metric updates through Data Firehose to
S3 or a supported observability platform. A stream in an OAM monitoring account can include metrics
from linked source accounts. Filter namespaces and individual metrics instead of exporting every
series, and request extra statistics only when the consumer uses them: pricing is based on metric
updates, with additional Firehose and destination charges.

Build service dashboards in the monitoring account with explicit account and Region labels. Start
with business SLIs, then latency/error/saturation and deployment markers; a wall of host CPU graphs
is not a service view. Keep source-account alarms for local safety actions that must continue if the
monitoring account or link is unavailable, and use central composite alarms for cross-service
operator notification.

### Retention, encryption, and access separation

Set log-group retention by data class rather than leaving the default **never expire** everywhere:

| Data class | Typical treatment |
|------------|-------------------|
| Active operational logs | Searchable in CloudWatch Logs for the incident-response window |
| Long-term audit/security evidence | Stream or export to a security-owned S3 archive with lifecycle and, if required, Object Lock |
| Debug/high-volume logs | Short retention, sampling or filtering, with explicit temporary extensions during an investigation |

CloudWatch Logs is encrypted at rest by default; associate a customer managed KMS key when key
ownership, revocation, or audit separation requires it. The key policy must allow the regional
CloudWatch Logs service and the operators that must decrypt/query the data. Test centralization after
changing a key: an inaccessible or disabled key can stop delivery. Apply the intended encryption and
retention again in the destination because a centralized copy has its own lifecycle. For an
organization centralization rule that uses a customer managed key for destination log groups, apply
the required `LogsManaged=true` key tag and permissions; an existing destination log group with a
different encryption configuration can cause delivery to be skipped.

Separate roles so no single everyday operator owns the evidence and its controls:

- **Workload roles** publish telemetry and inspect only their application.
- **Observability platform administrators** manage OAM sinks, centralization, streams, dashboards,
  and KMS integration, but do not administer workloads.
- **On-call/read roles** query approved logs, traces, dashboards, and SLOs without changing retention
  or deleting log groups.
- **Security/audit roles** read protected archives and CloudTrail evidence; deletion and key
  administration use separate break-glass paths.

Protect central destinations, sink policies, retention, and KMS keys with least privilege and
organization guardrails. Record administrative changes in CloudTrail.

### Control cardinality before it becomes a bill

Every unique custom metric name plus dimension-value set is a separate time series. Never use
request IDs, user IDs, timestamps, or other unbounded values as metric dimensions. Put those values
in logs or traces and keep metric dimensions bounded, such as `Service`, `Operation`, `Region`, and a
small set of result classes.

Review these cost drivers per account and team:

- custom/high-resolution metrics and high-resolution alarms;
- detailed resource monitoring, dashboards, canaries, and Application Signals;
- log ingestion, duplicate central copies, archive storage, and Logs Insights bytes scanned;
- metric-stream updates, extra statistics, Firehose, and downstream vendor ingestion;
- cross-Region transfer and KMS requests.

Use log classes, retention, metric/log filters, sampling, and budgets deliberately. Dropping all
telemetry to save cost is not an optimization; keep the signals required by the SLO and incident
runbook, then remove data that has no consumer.

---

## 10. Turn Business Requirements into SLOs and Proven Improvements

A **KPI** states what the business cares about. A **service-level indicator (SLI)** is the measured
signal, and a **service-level objective (SLO)** is the target over an interval. The **error budget**
is the allowed amount of failure while still meeting that objective.

| Business statement | Actionable observability definition |
|--------------------|-------------------------------------|
| "Checkout must work" | Request-based availability SLI = successful checkout requests / valid checkout attempts; SLO = 99.9% over a 28-day rolling interval |
| "Checkout must feel fast" | Latency SLI = proportion of valid checkout requests completed under 400 ms, plus p95/p99 latency for diagnosis |
| "Orders must not sit in the queue" | End-to-end order-age SLI from accepted to processed; SLO = 99% completed within two minutes |

Do not silently substitute an easy infrastructure metric for the requirement. CPU can help diagnose
a slow checkout, but `CPUUtilization < 70%` is not proof that customers checked out successfully.
Likewise, Application Signals' standard availability counts non-`5xx` responses as successful; use a
custom metric or metric expression if a business failure can return `2xx`/`4xx`.

### Scenario A: protect the checkout SLO

1. Instrument total and successful valid checkouts plus the end-to-end latency distribution. Use
   Application Signals' latency/availability metrics where their semantics match; otherwise define a
   request-based SLO from a bounded-dimension custom metric or metric-math expression.
2. Create short- and long-window **burn-rate alarms**. A fast burn pages before a short outage spends
   the interval's error budget; a slower burn creates a ticket for a persistent regression.
3. Add a **Synthetics canary** that performs a safe test checkout through the public path, including
   DNS, TLS, authentication, and a reversible test transaction. Store artifacts with short retention
   and protect any secret in Secrets Manager, not in the script.
4. Use an **anomaly-detection alarm** for a seasonal supporting signal such as request count or
   latency. It detects an unusual traffic collapse or shape change; the fixed SLO still defines what
   is acceptable.
5. Page through a **composite alarm** only when customer impact is credible—for example, fast error-
   budget burn and a failed canary. Keep the component alarms visible so a canary bug does not hide a
   real server-side failure.

For a safe, known failure mode, route the alarm state change through EventBridge to a versioned,
idempotent Systems Manager Automation runbook—for example, roll back the latest deployment or shift
traffic to a healthy target group. Limit concurrency and errors, verify preconditions, and require
approval for destructive or ambiguous actions. Automation failure must page a human rather than
loop indefinitely.

### Scenario B: remove an order-processing bottleneck

Suppose the order-age SLO is being missed while queue depth grows. Correlate arrival rate, age of the
oldest valid work item, consumer throughput, errors, throttles, and downstream latency. A composite
alarm such as `OrderAgeSLOBreach AND ConsumerErrors` distinguishes a broken consumer from an
expected short burst; anomaly detection on arrival rate adds context without redefining the SLO.

After increasing consumer concurrency or removing a downstream limit, prove the change worked:

1. Record a baseline across comparable traffic periods: SLO attainment/error-budget burn, p50/p95/
   p99 processing time, throughput, errors, queue age, and resource saturation.
2. Mark the deployment on the dashboard and run the same representative load or canary workflow.
3. Compare distributions and error-budget consumption, not only averages. Check that throughput
   improved without increasing failures, throttling, or cost beyond the agreed boundary.
4. Observe for at least the longest important seasonality window and confirm the alarm/canary still
   detects an injected or staged failure.
5. Keep the before/after dashboard, Logs Insights query, change ID, and load-test parameters as
   evidence. Roll back if the predefined SLO, error, saturation, or cost threshold is breached.

An alarm returning to `OK` proves only that its expression is currently below threshold. It does not
by itself prove causality, sustained improvement, or that a different customer segment was not harmed.

---

## Key Exam Points

- ✅ **EC2 memory & disk usage require the CloudWatch Agent** — they are custom metrics, not
  standard. The #1 CloudWatch trap.
- ✅ **Basic = 5-minute**, **Detailed = 1-minute**, high-resolution custom down to 1 second.
  Faster scaling → enable detailed monitoring.
- ✅ Alarm states: `OK`, `ALARM`, `INSUFFICIENT_DATA`. Actions target **SNS**, **Auto
  Scaling**, **EC2** (stop/terminate/reboot/recover), and SSM.
- ✅ **Composite alarms** reduce notification noise by combining alarms with AND/OR/NOT.
- ✅ **`treatMissingData`** controls gap behavior: `missing` (default; can become
  `INSUFFICIENT_DATA`), `notBreaching`, `breaching`, or `ignore` (retain current state).
- ✅ **Anomaly-detection alarms** learn a seasonal band — use when a static threshold can't
  fit day/night traffic patterns.
- ✅ **Metric filter** → turn a log pattern into a metric to alarm on. **Subscription
  filter** → stream logs in real time to Kinesis/Lambda. **Logs Insights** → ad-hoc queries.
- ✅ Log **retention is set on the log group** and defaults to *never expire* (cost trap).
- ✅ Dashboards can be **cross-Region and cross-account**.
- ✅ **EventBridge schedule** for "run on a timer"; **alarm** for "metric crossed a threshold."
- ✅ **Synthetics canaries** = scheduled outside-in probes; **X-Ray/ServiceLens** = tracing.
- ✅ **OAM sink + source links** = regional cross-account observability without copying data.
  Logs centralization/subscriptions and metric streams are for copied or streamed data.
- ✅ Start from a business **KPI → SLI → SLO + error budget**. Burn-rate alarms expose
  customer-impact risk; infrastructure metrics diagnose it.
- ✅ Bound metric dimensions. A request/user ID as a dimension creates high-cardinality custom
  metrics and uncontrolled cost.

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
- ❌ Leaving `treatMissingData` on default and being surprised an idle metric moves to
  `INSUFFICIENT_DATA` instead of `OK`. Choose missing-data semantics from what silence means.
- ❌ Forcing a static threshold onto seasonal traffic — that's what **anomaly detection** is for.
- ❌ Treating OAM as a central archive. It shares access; source retention still deletes the data.
- ❌ Applying an account-level subscription to the pipeline's own log groups and creating a
  recursive ingestion loop.
- ❌ Calling a lower CPU average proof of a customer-facing improvement. Compare SLO attainment,
  latency distributions, errors, throughput, saturation, and cost under comparable load.

---

**Next**: [02_cloudtrail.md — AWS CloudTrail: Auditing Every API Call](02_cloudtrail.md)
