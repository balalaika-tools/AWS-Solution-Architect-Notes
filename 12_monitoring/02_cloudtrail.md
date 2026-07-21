# AWS CloudTrail — Auditing Every API Call

> **Who this is for**: SAA-C03 candidates who understand [IAM principals](../02_iam/README.md)
> and [CloudWatch](01_cloudwatch.md), and now need the audit/governance angle: a permanent
> record of *who did what, when, and from where* across the AWS account.

---

## 1. Prerequisite: What Audit Logging Is

A monitoring tool like CloudWatch tells you the system's *health*. An **audit log** answers a
different, security-and-governance question: **"Who performed which action, on which
resource, from which IP, at what time — and did it succeed?"**

Audit logs are the backbone of **forensics** (reconstructing an incident), **compliance**
(proving controls to an auditor), and **governance** (detecting unauthorized or unexpected
activity). They must be **tamper-evident** and **retained**, because their whole value is
being trustworthy after the fact.

> **Key insight**: Almost everything in AWS happens through an **API call** — the console,
> the CLI, an SDK, and even AWS services acting on your behalf all ultimately call APIs.
> If you record every API call, you have a complete activity history. That is exactly what
> CloudTrail does.

---

## 2. What CloudTrail Records

**AWS CloudTrail** logs API activity in your account. Each entry includes the **identity**
that made the call (IAM user/role/service), the **time**, the **source IP**, the **request
parameters**, and the **response** — all as a JSON event.

```
Someone calls ec2:TerminateInstances
        │
   CloudTrail captures the event:
   {
     "eventTime":   "2026-06-02T14:03:22Z",
     "eventSource": "ec2.amazonaws.com",
     "eventName":   "TerminateInstances",
     "userIdentity": { "type": "IAMUser", "userName": "dev-bob" },
     "sourceIPAddress": "203.0.113.10",
     "requestParameters": { "instanceId": "i-0abc..." },
     "errorCode":   null            ← present if the call was denied/failed
   }
```

### The event types

| Event type | Captures | Enabled by default? | Cost |
|------------|----------|---------------------|------|
| **Management events** | Control-plane operations: create/modify/delete resources, IAM changes, login (`RunInstances`, `AttachRolePolicy`, `ConsoleLogin`) | ✅ Yes (read+write) | Free for the first copy |
| **Data events** | Data-plane, high-volume object-level ops: **S3 `GetObject`/`PutObject`**, **Lambda `Invoke`**, DynamoDB item activity | ❌ No — opt in | Paid (high volume) |
| **Insights events** | Anomalies in recorded API call or error rates, for selected management or data events | ❌ No — opt in | Paid analysis of the underlying events |
| **Network activity events** | AWS API calls made through **VPC endpoints**, including VPC endpoint ID/account context | ❌ No — opt in per event source | Paid |

⚠️ **Exam trap**: Data events are **off by default** because they can be enormous (every S3
object read). If a scenario needs "log who read objects in this S3 bucket," you must
**enable S3 data events**.

💡 **Newer visibility pattern**: Network activity events answer "which AWS API calls crossed this
VPC endpoint?" Useful for endpoint-owner visibility and `VpceAccessDenied` investigations. They are
not packet captures; VPC Flow Logs are still the packet/flow tool.

---

## 3. Event History vs Trails

This distinction is the most-tested part of CloudTrail.

| | **Event History** | **Trail** |
|---|---|---|
| Always on? | ✅ Yes, automatically | ❌ You create it |
| Retention | **Last 90 days** only | **Indefinite** (you control) |
| Stored where | Inside CloudTrail (viewable in console) | **S3 bucket** you own (+ optionally CloudWatch Logs) |
| Events covered | Management events only | Management, data, Insights, and network activity events |
| Cost | Free | S3 storage + paid event categories (data, Insights, network activity) |
| Querying | Console search | Athena, custom tooling |

```
            ┌─────────────── Event History (free, 90 days, management only) ─────────────┐
API calls ──┤                                                                             │
            └──► Trail ──► S3 bucket (indefinite) ──► (Athena queries / CW Logs / SIEM)   ┘
```

> **Rule**: For anything longer than **90 days**, for data events, or for sending logs to
> S3/Athena/a SIEM, you need a **trail**. Event History alone is not a compliance solution.

### Trail scope options

- **Multi-Region trail** — captures events from **all Regions** into one S3 bucket. ✅ Best
  practice; a single-Region trail misses activity elsewhere. New trails default to
  multi-Region.
- **Organization trail** — created by the **management account or a CloudTrail delegated
  administrator**, it logs the management account and **every member account** into one central
  bucket. Member accounts can see it but can't disable or alter it.

### Organization-trail operating model

Keep governance administration and evidence storage separate from workloads:

```
AWS Organizations
  management + current/future member accounts, all enabled Regions
                         │
              multi-Region organization trail
                         │
                         ▼
     Log archive account: dedicated S3 bucket + KMS key
       versioning / Object Lock / lifecycle / validation digests
                         │
              ┌──────────┴───────────┐
              ▼                      ▼
       Athena investigation     alerting / SIEM pipeline
```

The management account enables trusted access and can register a security account as a
**CloudTrail delegated administrator**. The delegated account can create and manage organization
trails, but the management account remains the owner of those organization resources. Removing the
delegated administrator therefore does not delete the trails it created. Keep routine work out of
the management account and tightly restrict who can change the trail.

Use a multi-Region organization trail in each AWS partition that the company uses. New member
accounts are enrolled automatically. An opt-in Region still has account-enablement considerations,
so verify trail status after enabling Regions or adding accounts rather than assuming coverage.
Member-account **Event History is not aggregated**: signing in to the management account shows only
that account's 90-day history. Organization-wide investigation uses the central S3 logs.

### Central immutable S3 storage and policy dependencies

Put the destination bucket in a security-owned log archive account where workload administrators
cannot delete objects, change lifecycle, or disable the key. The bucket policy must:

- allow `cloudtrail.amazonaws.com` to check the bucket ACL and write log/digest objects;
- grant `s3:PutObject` to both the management-account and organization prefixes, including
  `AWSLogs/<organization-id>/*`;
- require `bucket-owner-full-control` and restrict the service permission with the exact
  `aws:SourceArn` of the **management-account-owned** trail;
- deny non-TLS requests and avoid granting member accounts write or delete access.

Enable versioning and set lifecycle from the evidence-retention requirement. Use **S3 Object Lock**
when logs must be WORM-protected; governance mode allows a controlled bypass, while compliance mode
cannot be shortened or bypassed before expiry. A separately administered cross-Region replica can
add Region/account isolation, but it also needs replication, KMS, retention, and investigation
testing.

SSE-KMS adds key control but also a delivery dependency. The customer managed key policy must let
CloudTrail call `kms:GenerateDataKey*` and `kms:DescribeKey`, constrained by the trail ARN and
CloudTrail encryption context. Investigator roles need separate decrypt permission. A disabled key,
incorrect encryption context, wrong bucket prefix, or missing bucket statement can stop delivery
even though the organization trail object exists in member accounts. Alarm on delivery errors and
check `get-trail-status` during rollout and after every bucket/key-policy change.

### Event selection and cost control

Start with one authoritative organization trail recording read and write **management events** in
all Regions. The first copy of management events in a Region is free, but additional copies on other
trails are charged; remove accidental duplication before trimming security coverage. If justified,
exclude high-volume KMS or RDS Data API management events, documenting the forensic visibility that
is lost.

Data and network activity events are paid and potentially much larger. Use **advanced event
selectors** to include only the resource types, ARNs, API names, read/write direction, identities,
VPC endpoints, or error conditions needed by the threat model. For example, record writes and reads
on a regulated S3 prefix rather than every `GetObject` in every bucket, and collect network activity
events for sensitive endpoints or `VpceAccessDenied` investigations. Revisit selectors when new
accounts and data stores are created.

Enable **CloudTrail Insights** where an unusual API call rate or API error rate would be meaningful.
Insights must analyze the underlying recorded events and is charged by analyzed volume. A call-rate
Insight needs applicable write events; an error-rate Insight can use applicable read or write
events. It detects deviation from that account's learned baseline—it is not a replacement for a
deterministic rule such as "any `DeleteTrail` call must alert."

---

## 4. Log File Integrity Validation

When enabled, CloudTrail produces **digest files** containing SHA-256 hashes of the log
files, signed with SHA-256 + RSA. You can later verify that delivered logs were **not
modified or deleted** since CloudTrail wrote them.

> **Exam pattern**: "Prove to auditors the audit logs haven't been tampered with" →
> **enable log file integrity validation**. Pair it with an **S3 bucket policy + MFA Delete
> / Object Lock** so even an admin can't quietly delete evidence.

💡 Integrity validation is **tamper-evidence**, not tamper-prevention. Enabling it makes CloudTrail
deliver hourly signed digest files; an investigator must still run `aws cloudtrail validate-logs`
for the account, Region, and time range. S3 Object Lock/MFA Delete and separated administrators
prevent or limit deletion. Use both controls when the requirement says immutable *and* provable.

Protect digest files as carefully as logs and keep them in the original delivered location for the
standard CLI validation workflow. Test validation periodically and preserve the output with the case
record; merely enabling the checkbox is not proof that a specific investigation range validated.

---

## 5. Integrations

### CloudTrail/API events → EventBridge (deterministic detection)

Use an EventBridge rule with detail type `AWS API Call via CloudTrail` for high-value events such as
`StopLogging`, `DeleteTrail`, `PutBucketPolicy`, changes to KMS key state, or a security-group rule
opened to the internet. Send the match to a central event bus, SNS/SQS, Lambda, Step Functions, or a
guardrailed Systems Manager Automation runbook.

EventBridge rules run in the account and Region where the event is emitted. A central organization
trail's S3 destination does **not** move those EventBridge events to the log archive account. Deploy
rules to the required accounts/Regions and forward them to a security event bus, or use the
CloudTrail → CloudWatch Logs path below for centralized metric filters. Test the exact API event:
not every AWS service event has identical fields or delivery behavior, and automated remediation
must be idempotent and narrowly scoped.

### CloudTrail → CloudWatch Logs (real-time alerting)

By itself CloudTrail delivers to S3 about every ~5 minutes. To **react in near-real-time**,
send the trail to **CloudWatch Logs**, add a **metric filter**, and alarm.

```
CloudTrail ──► CloudWatch Logs ──► metric filter ("ConsoleLogin" without MFA,
                                    or "DeleteTrail", or root account usage)
                                        │
                                     Alarm ──► SNS ──► security team
```

> **Exam pattern**: "Notify security when someone uses the **root account**" or "alert on
> any IAM policy change" → CloudTrail → CloudWatch Logs metric filter → alarm → SNS. This is
> where CloudTrail and CloudWatch combine.

### CloudTrail → Athena (querying)

Trail logs in S3 are JSON. **Amazon Athena** queries them in place with SQL — no ETL — for
forensic questions like "show every `AssumeRole` from this IP last week." You can create the
Athena table directly from the CloudTrail console.

For an organization trail, preserve or project partitions by organization/account, Region, and UTC
date so a query scans only the incident window. Restrict the Athena workgroup, result bucket, and
KMS decrypt access to investigation roles, and set query scan limits to prevent an accidental
full-archive scan.

### CloudTrail Lake

A managed, immutable event data store with built-in SQL querying — an alternative to the
S3 + Athena pattern when you want CloudTrail to host and query the data itself.

⚠️ **Current availability**: as of **May 31, 2026**, CloudTrail Lake is **not open to new
customers**. Existing customers can continue using it, but new designs should favor an
organization/multi-Region trail delivered to S3 and queried with Athena/OpenSearch/SIEM tooling, or
CloudWatch-based event analysis where that fits the requirement.

### Cross-account and cross-Region investigation workflow

When an alert says a role changed a KMS key in one account, investigate the whole identity session,
not only the triggering event:

1. Preserve the alert, UTC time, source account/Region, `eventID`, `requestID`, principal ARN,
   session issuer, source IP, user agent, and affected resource ARN.
2. Query the organization trail around that time across **all Regions and accounts** for the same
   access-key ID, role session, source IP, request ID, and affected resource. Follow preceding
   `AssumeRole` activity to the original identity and look for follow-on discovery or persistence.
3. Check `errorCode`/`errorMessage`; denied calls show intent and may explain a successful call that
   followed with broader credentials. Account for clock/time-zone conversion and delivery lag.
4. Pivot to AWS Config for before/after resource state, CloudWatch/application logs for effect, and
   VPC Flow Logs when network behavior matters. CloudTrail alone is API evidence, not packet or
   application evidence.
5. Validate the relevant digest chain, record the query and immutable object versions used, then
   remediate through the incident process. Do not edit or move the evidentiary originals.

Shared-resource calls may be recorded in more than one account with different perspectives. Use
`recipientAccountId`, resource ownership, session context, and request identifiers to correlate them
instead of assuming the account that stores an event owned the caller.

---

## 6. What CloudTrail Is NOT

⚠️ The single most common exam confusion — keep these straight:

| If the question is about… | The answer is… | NOT CloudTrail because… |
|---------------------------|----------------|-------------------------|
| CPU, memory, latency, throughput | **CloudWatch** | CloudTrail records API *calls*, not performance metrics |
| Application/server logs (`app.log`) | **CloudWatch Logs** | CloudTrail only logs AWS *API* activity |
| "Is this resource configured correctly / compliant?" | **AWS Config** | CloudTrail logs the *change action*, not the resulting *config state* |
| Network packet/flow visibility | **VPC Flow Logs** | CloudTrail is API-level, not packet-level |

> **Mental model**: CloudWatch = *health/performance*. CloudTrail = *who did what (API
> audit)*. Config = *how things are configured & compliant* (next file). CloudTrail is the
> **"security camera over the AWS API."**

The full three-way breakdown and a hands-on comparison live in the next file and in
[Practical Examples — CloudTrail vs CloudWatch vs Config](../18_practical_examples/17_cloudtrail_vs_cloudwatch_vs_config.md).

---

## Key Exam Points

- ✅ CloudTrail = **audit log of API calls**: identity, time, source IP, parameters, result.
- ✅ **Event History** is free, automatic, **90 days**, management events only.
- ✅ A **trail** stores indefinitely in **S3**; needed for >90 days, data events, Athena, SIEM.
- ✅ **Management events** = control plane (default on). **Data events** = S3 objects / Lambda
  invokes (**off by default**, high volume, paid). **Insights** = anomaly detection. **Network
  activity events** = API calls through VPC endpoints (opt in, paid).
- ✅ **Multi-Region trail** captures all Regions; **organization trail** centralizes all member
  accounts (and members can't disable it). A delegated administrator operates it; the management
  account remains the owner.
- ✅ **Log file integrity validation** detects modification/deletion through signed digest files;
  **Object Lock** prevents deletion/overwrite during retention. Use both for immutable evidence.
- ✅ Cross-account SSE-KMS delivery needs correct **bucket and KMS key policies** constrained to
  the management-account-owned trail ARN. Monitor delivery after policy/key changes.
- ✅ Use advanced selectors to control paid data/network event volume; use **Insights** for
  abnormal call/error rates and **EventBridge** for deterministic sensitive API events.
- ✅ Send to **CloudWatch Logs + metric filter + alarm** for near-real-time security alerts
  (root login, IAM changes). Query S3 trail logs with **Athena**.
- ✅ **CloudTrail Lake** is existing-customer/legacy for new designs after May 31, 2026.

---

## Common Mistakes

- ❌ Expecting S3 object reads in CloudTrail by default — you must enable **S3 data events**.
- ❌ Relying on Event History for compliance — it's only **90 days** and management events.
- ❌ Using CloudTrail to monitor performance/CPU/latency — that's **CloudWatch**.
- ❌ Forgetting a single-Region trail misses other Regions — use **multi-Region**.
- ❌ Thinking CloudTrail delivers instantly — S3 delivery lags minutes; route to CloudWatch
  Logs for fast alerting.
- ❌ Assuming the management account's Event History contains member-account events. Query the
  organization trail's central S3 data instead.
- ❌ Enabling log validation and calling the logs immutable. Validation detects tampering; S3
  Object Lock and separated delete/key administration provide prevention.
- ❌ Enabling every data event across the organization without selectors, or duplicating
  management events on several trails, and discovering the volume only after billing.
- ❌ Creating a central organization trail and expecting EventBridge rules in one account to see
  every source-account/Region API event. Deploy/forward the detection plane explicitly.

---

**Next**: [03_config.md — AWS Config: Configuration State & Compliance](03_config.md)
