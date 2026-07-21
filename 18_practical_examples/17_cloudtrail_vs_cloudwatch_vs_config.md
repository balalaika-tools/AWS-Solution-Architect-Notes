# CloudTrail vs CloudWatch vs Config — Which Tool Answers Which Question

> **Who this is for**: SAA-C03 candidates who keep mixing up the three "observability /
> governance" services because all three "watch your account." We pin each one to a concrete
> question it — and *only* it — answers, show a real example of each, then map how they feed
> one another. Builds on [CloudWatch](../12_monitoring/01_cloudwatch.md),
> [CloudTrail](../12_monitoring/02_cloudtrail.md), and [Config](../12_monitoring/03_config.md).

---

## 1. Three Questions, Three Services

The fastest way to pick the right service on the exam is to phrase the requirement as a
question and match it to a verb:

| The question | The verb | Service |
|--------------|----------|---------|
| **"Who deleted this S3 bucket?"** | *Who did what?* (audit) | **CloudTrail** |
| **"Is CPU > 80%? Alert me."** | *Is it healthy?* (performance/health) | **CloudWatch** |
| **"Which SGs allow 0.0.0.0/0 on 22, and when did that change?"** | *Is it compliant / what's its state history?* | **AWS Config** |

> **Key insight**: CloudTrail records the **action** (an API call happened). CloudWatch
> records **metrics & logs** (how the system is performing). Config records the **resulting
> resource state** and evaluates it against rules. CloudTrail = *who*, CloudWatch = *how
> healthy*, Config = *configured correctly and how it changed*.

---

## 2. "Who deleted this S3 bucket?" → CloudTrail

CloudTrail logs every API call (console, CLI, SDK) as an **event**: who, what, when, from
where. To answer the bucket-deletion question you look up the `DeleteBucket` event.

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteBucket \
  --start-time 2026-05-30T00:00:00Z
```

A trimmed `DeleteBucket` event record:

```json
{
  "eventTime": "2026-06-01T14:03:22Z",
  "eventSource": "s3.amazonaws.com",
  "eventName": "DeleteBucket",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "203.0.113.42",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "dev-alice",
    "arn": "arn:aws:iam::111122223333:user/dev-alice"
  },
  "requestParameters": { "bucketName": "prod-customer-uploads" },
  "responseElements": null,
  "readOnly": false,
  "managementEvent": true,
  "eventCategory": "Management"
}
```

There it is: `dev-alice` deleted `prod-customer-uploads` at 14:03 from `203.0.113.42`.

⚠️ CloudTrail logs **management events** (control-plane: create/delete/modify) by default.
**Data events** (S3 object-level `GetObject`/`PutObject`, Lambda `Invoke`) are high-volume,
**not logged by default**, and cost extra. "Who *read* this object?" needs data events enabled.
**Network activity events** are also opt-in and capture AWS API calls made through VPC endpoints,
including endpoint context; use VPC Flow Logs for packet/flow visibility.

💡 The default **Event history** keeps **90 days** of management events. For longer retention
or org-wide capture, create a **Trail** delivering to S3 (and optionally CloudWatch Logs).

---

## 3. "Is CPU > 80%? Alert me." → CloudWatch

This is a metric crossing a threshold — squarely CloudWatch. (Full alarm→scaling mechanics in
[16_cloudwatch_alarm_scaling.md](16_cloudwatch_alarm_scaling.md).)

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name ec2-cpu-over-80 \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0abc1234 \
  --statistic Average --period 300 \
  --evaluation-periods 2 --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:111122223333:ops-alerts
```

CloudWatch owns **metrics** (numeric health), **Logs** (application/system log lines), and
**alarms** (threshold → action). It answers *is the system performing acceptably* — never
*who did it*.

❌ Common trap: "alert when an unauthorized API call happens." That sounds like CloudWatch but
the *source* of API activity is **CloudTrail**. You send CloudTrail to a **CloudWatch Logs
metric filter**, then alarm on that. CloudWatch alone has no idea who called an API.

---

## 4. "Which SGs allow 0.0.0.0/0 on 22, and when did it change?" → Config

This is a **resource-state + history + compliance** question — AWS Config. Config records a
**configuration item** every time a resource changes and evaluates state against **rules**.

```bash
# Turn on the AWS-managed rule for SSH open to the world
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "restricted-ssh",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "INCOMING_SSH_DISABLED"
  },
  "Scope": { "ComplianceResourceTypes": ["AWS::EC2::SecurityGroup"] }
}'

# Which SGs are non-compliant (i.e., allow 0.0.0.0/0 on 22)?
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name restricted-ssh \
  --compliance-types NON_COMPLIANT

# When did THIS SG's configuration change?
aws configservice get-resource-config-history \
  --resource-type AWS::EC2::SecurityGroup \
  --resource-id sg-0abc1234
```

Config answers **both halves**: the *current* non-compliant set (the rule) **and** the
*timeline* of how `sg-0abc1234` got there (the configuration history, with before/after).

💡 Config can auto-fix via **remediation actions** (e.g., an SSM Automation document that
revokes the open rule) and can notify on every change/compliance flip via SNS.

---

## 5. The Big Three-Way Comparison

| | **CloudTrail** | **CloudWatch** | **AWS Config** |
|---|---|---|---|
| Core question | *Who did what, when, from where?* | *Is it healthy / performing?* | *Is it configured right? How did it change?* |
| Primary data | API call **events** | **Metrics** & **Logs** | **Configuration items** (resource state) |
| Records | The **action** (request) | Numeric series + log lines | The **resulting state** + relationships |
| Granularity | Per API call | Per metric datapoint / log event | Per resource change (snapshot) |
| Compliance rules | No | No | **Yes** (managed + custom rules) |
| History | 90 days (Event history); Trail→S3 uses the retention you configure | Metric retention up to 15 mo; logs as configured | Per-resource timeline while recorded/retained |
| Alerting | Via CloudWatch Logs metric filter | **Alarms** → SNS/Auto Scaling | Compliance change → SNS / EventBridge |
| Typical answer keyword | "audit", "who", "forensics", "API call" | "metric", "threshold", "CPU/latency", "alarm" | "compliant", "drift", "configuration change", "0.0.0.0/0" |
| Scope | Regional/org trail | Regional | Regional (aggregator for org) |

> **Rule of thumb**: see **"who / audit / API call"** → CloudTrail. See **"metric / threshold
> / utilization / latency / alarm"** → CloudWatch. See **"compliant / drift / configuration /
> security-group rule / over time"** → Config.

---

## 6. How They Overlap and Feed Each Other

They're complementary, not competitors. A single incident often touches all three:

```
                          ┌──────────────────────────────────────────┐
   API call (someone      │            AWS account activity           │
   opens SG to 0.0.0.0/0) │                                           │
            │             │                                           │
            ▼             │                                           │
   ┌─────────────────┐    │   records the ACTION (who/when)           │
   │   CloudTrail     │───┼──────────────┐                            │
   └────────┬────────┘    │              │ delivers events to         │
            │             │              ▼                            │
            │             │     ┌──────────────────┐  metric filter   │
            │             │     │ CloudWatch Logs  │──── + alarm ──▶ SNS
            │             │     └──────────────────┘                  │
            │             │                                           │
            │ triggers re-evaluation                                  │
            ▼             │                                           │
   ┌─────────────────┐    │   records the STATE + checks the RULE     │
   │   AWS Config     │───┼─── NON_COMPLIANT ──▶ EventBridge / SNS ──▶ remediation
   └─────────────────┘    │                                           │
                          └──────────────────────────────────────────┘
```

- **CloudTrail → CloudWatch Logs**: ship trail events into a log group, add a **metric
  filter** (e.g., count `AuthorizeSecurityGroupIngress` with cidr `0.0.0.0/0`), alarm on it.
- **Config → SNS / EventBridge**: every configuration/compliance change publishes a
  notification, enabling near-real-time alerts and auto-remediation.
- **CloudTrail and Config together**: CloudTrail says *"`dev-alice` called
  `AuthorizeSecurityGroupIngress` at 14:03"*; Config says *"as a result `sg-0abc1234` is now
  NON_COMPLIANT, and here's the before/after diff."* The action vs the resulting state.

---

## 7. Organization-Wide Build

The single-account examples explain the data, but an enterprise needs one governed place to
query it without making the Organizations management account the daily operations console.

```text
Organizations management account
  └─ registers delegated administrators and protects org-level configuration

Log archive account
  └─ organization CloudTrail → immutable S3 (+ optional CloudTrail Lake)

Observability account
  ├─ CloudWatch OAM sinks ← links from workload accounts
  ├─ central Logs destinations ← subscription filters / central collection
  └─ dashboards, alarms, investigations, incident routing

Security tooling account
  ├─ organization Config aggregator
  ├─ organization conformance packs
  └─ EventBridge → Systems Manager Automation remediation
```

These roles can be combined in a smaller organization, but keep log-retention administration
separate from principals being audited. Use the management account to enable trusted access and
register delegated administrators, then perform normal CloudTrail/Config/observability work from
member accounts with narrowly scoped roles.

### CloudTrail: one organization trail, protected evidence

Create a multi-Region **organization trail** so current and future member accounts contribute
management events. Add only the data/network activity events required by the threat model because
they can be high-volume and costly. CloudTrail supports a delegated administrator for managing
organization trails and event data stores; the management account must establish that delegation
and remains the ultimate Organizations control boundary.

Deliver to a dedicated S3 bucket in the log-archive account:

- bucket policy permits only the CloudTrail service delivery path with the expected organization
  trail/source conditions and denies insecure transport;
- SSE-KMS key policy permits CloudTrail delivery and tightly separates log readers from key/log
  administrators;
- enable log-file validation and monitor delivery failures;
- enable S3 Versioning and, where evidence requires write-once retention, create the bucket with
  **S3 Object Lock** and apply an approved governance/compliance retention mode;
- deny member-account deletion and lifecycle changes, while keeping a tested break-glass and legal
  retention procedure in the archive account.

CloudWatch Logs is useful for near-real-time filters, but it is not a substitute for the protected
S3 evidence copy. Event history is only a convenient 90-day Regional view. CloudTrail Lake can add
SQL-style organization queries and its own retention, but its retention/cost and access model must
be designed rather than assumed infinite.

### CloudWatch: share telemetry and centralize the logs that need operations

CloudWatch cross-account observability uses **OAM** (Observability Access Manager): create a sink
in the monitoring account, then create a link from each source account selecting the metrics,
logs, traces, Application Signals, or other supported telemetry to share. Deploy links from an
Organizations policy/StackSet and repeat the design in each observed Region. The monitoring
account can build dashboards and alarms without copying every metric into a new namespace.

OAM is access to linked telemetry, not an immutable log archive. For logs that must be processed
centrally, configure CloudWatch Logs cross-account subscription delivery to an authorized central
destination such as Kinesis Data Streams, Firehose, or Lambda. The destination resource policy,
source subscription filter, KMS permissions, and destination capacity all have to agree. Scope
filters to useful log groups and monitor delivery/throttling; duplicating every debug log can cost
more than the workload.

Keep alarms near the response boundary. A central monitoring account is good for composite alarms,
dashboards, and on-call routing; a workload account may still need a local alarm/action for an ASG
policy or a containment path that must work when the central link is impaired.

### Config: aggregate state, deploy policy

An organization **Config aggregator** collects resource configuration/compliance views from
accounts and Regions into the security tooling account. It is read-only aggregation: it does
**not** turn on configuration recorders, delivery channels, rules, or remediation in source
accounts. Deploy and continuously check those prerequisites separately in every governed Region.

Use organization **conformance packs** to deploy a versioned group of Config rules and remediation
definitions to accounts/OUs. Register a Config delegated administrator so this does not run from
the management account. Treat exclusions as code with an owner and expiry, and canary rule changes
on a non-production OU; a bad custom rule deployed organization-wide can create both noise and
remediation risk.

Config records supported resource state, not every application/database setting. Confirm that the
resource type and Region are recorded and keep CloudTrail for the actor/API evidence.

### EventBridge and Systems Manager remediation

There are two useful triggers:

- a CloudTrail event routed through EventBridge for an urgent high-risk API action, such as
  disabling a trail or changing a protected security group; and
- a Config `NON_COMPLIANT` result for state-based repair, such as a security group that currently
  exposes SSH.

Both can invoke a least-privilege **Systems Manager Automation** runbook in the affected account.
A production runbook must:

1. Re-read the resource and confirm it is still noncompliant; events can be delayed or duplicated.
2. Check approved exception tags/registry and acquire an idempotency lock.
3. Capture the before-state, CloudTrail event ID, Config evaluation, account, Region, and owner.
4. Apply the smallest reversible change under a dedicated remediation role.
5. Verify compliance and application health, then record after-state and ticket/finding status.

Set Automation concurrency and error thresholds for organization runs. Require approval for a
change that could cut production traffic; automatic remediation is safest for high-confidence,
reversible controls. EventBridge delivery and Automation invocation should have DLQ/failure alarms.

### Joined example: SSH opened to the world

```text
SCP / permissions boundary       limits who may authorize ingress             (prevent)
CloudTrail organization trail    identifies actor and API request             (audit)
Config restricted-ssh rule       proves current SG is noncompliant            (detect)
OAM / central Logs + alarm       routes one actionable incident               (observe)
EventBridge → SSM Automation     removes only the offending rule, then checks (respond)
S3 Object Lock archive           preserves event/config evidence              (prove)
```

If remediation fails, operators can still answer whether the API call occurred, what the current
state is, why the runbook refused/failed, and whether the audit record is intact. That is the
production value of keeping these services distinct.

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Can't find who **read** an S3 object in CloudTrail | Only management events logged | Enable **data events** for the bucket (extra cost) |
| Trail events older than 90 days missing | Relying on Event history, no Trail | Create a Trail → S3 for long-term retention |
| Config shows no history for a resource | Recorder not recording that resource type / not enabled in Region | Enable the **configuration recorder**; check resource-type scope |
| CloudWatch alarm never fires on "unauthorized API" | CloudWatch has no API data | Route CloudTrail → CloudWatch Logs, add a metric filter, alarm on it |
| Config rule shows everything compliant but you see open SG | Rule scope excludes SGs, or rule not the SSH one | Verify `SourceIdentifier`/scope; rules are evaluated only against in-scope types |
| Need org-wide view | Per-account/Region services | CloudTrail **org trail**; Config **aggregator**; CloudWatch **cross-account observability** |
| Aggregator shows no resources from a new account | Aggregation was created, but Config recorder/delivery or authorization is absent in source | Deploy/verify the recorder in every Region and reconcile source-account membership |
| Central dashboard works but remediation scaling action does not | Cross-account visibility does not move a local service action | Keep the required action/alarm in the workload account or invoke a deliberately authorized cross-account runbook |
| Organization trail exists but evidence can be altered | Archive bucket/key admins can delete or shorten retention | Separate duties; enforce bucket/key policy, versioning, Object Lock retention, and alerts on trail/bucket changes |
| Auto-remediation removed an approved exception | Runbook trusted a stale event and had no exception check | Re-read state, check time-bounded exceptions, add approval/concurrency/error controls, and keep rollback data |

---

## Key Exam Points

- **CloudTrail = who/what/when (audit)**, **CloudWatch = health/metrics/alarms**, **Config =
  resource state + compliance + change history**. Memorize the three verbs.
- CloudTrail default = **management events**, **90-day** Event history; **data events** off by
  default; Trail → S3 for long retention.
- "Alert on an API call / unauthorized activity" = **CloudTrail → CloudWatch Logs metric
  filter → alarm** (not CloudWatch alone).
- Config is the only one with **rules, compliance, and per-resource history** — anything with
  "drift", "compliant", "0.0.0.0/0", or "how did this resource change over time" is Config.
- They chain: **CloudTrail → CloudWatch Logs**, **Config → SNS/EventBridge → remediation**.
  CloudTrail logs the action; Config records the resulting state.
- At organization scale: use a delegated-admin organization trail to an immutable archive,
  CloudWatch OAM/log subscriptions for central operations, and Config aggregators plus conformance
  packs for governed state.
- An aggregator does not enable Config recording, and cross-account observability does not replace
  local service actions. Reconcile coverage in every account and Region.
- EventBridge plus SSM Automation should revalidate state, preserve evidence, use least privilege,
  cap concurrency/errors, and verify recovery.

---

**Next**: [18_kms_encryption_examples.md — KMS Encryption in Practice](18_kms_encryption_examples.md)
