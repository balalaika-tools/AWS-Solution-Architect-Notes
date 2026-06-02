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

### The three event types

| Event type | Captures | Enabled by default? | Cost |
|------------|----------|---------------------|------|
| **Management events** | Control-plane operations: create/modify/delete resources, IAM changes, login (`RunInstances`, `AttachRolePolicy`, `ConsoleLogin`) | ✅ Yes (read+write) | Free for the first copy |
| **Data events** | Data-plane, high-volume object-level ops: **S3 `GetObject`/`PutObject`**, **Lambda `Invoke`**, DynamoDB item activity | ❌ No — opt in | Paid (high volume) |
| **Insights events** | ML-detected *anomalies* in management-event call patterns (e.g., a sudden burst of `DeleteBucket`) | ❌ No — opt in | Paid |

⚠️ **Exam trap**: Data events are **off by default** because they can be enormous (every S3
object read). If a scenario needs "log who read objects in this S3 bucket," you must
**enable S3 data events**.

---

## 3. Event History vs Trails

This distinction is the most-tested part of CloudTrail.

| | **Event History** | **Trail** |
|---|---|---|
| Always on? | ✅ Yes, automatically | ❌ You create it |
| Retention | **Last 90 days** only | **Indefinite** (you control) |
| Stored where | Inside CloudTrail (viewable in console) | **S3 bucket** you own (+ optionally CloudWatch Logs) |
| Events covered | Management events only | Management **and** data **and** Insights |
| Cost | Free | S3 storage + data/Insights events |
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
- **Organization trail** — created in the **management account**, it logs **every member
  account** of the AWS Organization into one central bucket. Member accounts can't disable
  or alter it — ideal for centralized, tamper-resistant governance.

---

## 4. Log File Integrity Validation

When enabled, CloudTrail produces **digest files** containing SHA-256 hashes of the log
files, signed with SHA-256 + RSA. You can later verify that delivered logs were **not
modified or deleted** since CloudTrail wrote them.

> **Exam pattern**: "Prove to auditors the audit logs haven't been tampered with" →
> **enable log file integrity validation**. Pair it with an **S3 bucket policy + MFA Delete
> / Object Lock** so even an admin can't quietly delete evidence.

💡 The chain of: *multi-Region/org trail → S3 with Object Lock → integrity validation* is the
canonical "make our audit trail tamper-proof" answer.

---

## 5. Integrations

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

### CloudTrail Lake

A managed, immutable event data store with built-in SQL querying — an alternative to the
S3 + Athena pattern when you want CloudTrail to host and query the data itself.

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
  invokes (**off by default**, high volume, paid). **Insights** = anomaly detection.
- ✅ **Multi-Region trail** captures all Regions; **organization trail** centralizes all member
  accounts (and members can't disable it).
- ✅ **Log file integrity validation** (digest files) proves logs weren't tampered with.
- ✅ Send to **CloudWatch Logs + metric filter + alarm** for near-real-time security alerts
  (root login, IAM changes). Query S3 trail logs with **Athena**.

---

## Common Mistakes

- ❌ Expecting S3 object reads in CloudTrail by default — you must enable **S3 data events**.
- ❌ Relying on Event History for compliance — it's only **90 days** and management events.
- ❌ Using CloudTrail to monitor performance/CPU/latency — that's **CloudWatch**.
- ❌ Forgetting a single-Region trail misses other Regions — use **multi-Region**.
- ❌ Thinking CloudTrail delivers instantly — S3 delivery lags minutes; route to CloudWatch
  Logs for fast alerting.

---

**Next**: [03_config.md — AWS Config: Configuration State & Compliance](03_config.md)
