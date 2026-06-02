# AWS Config — Configuration State & Compliance

> **Who this is for**: SAA-C03 candidates who now know [CloudWatch](01_cloudwatch.md)
> (health) and [CloudTrail](02_cloudtrail.md) (who-did-what), and need the third member of
> the governance trio: *is each resource configured the way policy requires, and how did its
> configuration change over time?*

---

## 1. Prerequisite: Configuration Management & Compliance

As an account grows, two questions become unanswerable by hand:

1. **"Is this resource configured correctly?"** — e.g., is every S3 bucket encrypted? Are
   there any security groups open to `0.0.0.0/0` on port 22? Do all EBS volumes have
   encryption on?
2. **"How did this resource's configuration change, and who/what changed it?"** — drift
   detection and change forensics.

**Configuration management** treats each resource's settings as state you can record,
evaluate against rules, and track over time. **Compliance** is continuously checking that
recorded state matches policy.

> **Key insight**: CloudTrail records the **action** (an API call happened). AWS Config
> records the **resulting state** (the resource now looks like *this*) and **evaluates that
> state** against rules. CloudTrail says "someone modified the SG at 14:03"; Config says "the
> SG is now non-compliant because it allows 0.0.0.0/0 on 22, and here's the before/after."

---

## 2. Core Building Blocks

### Configuration items (CIs)

A **configuration item** is a point-in-time snapshot of a single resource's attributes,
relationships (e.g., this EC2 instance → these SGs, this VPC), and metadata. Config records
a new CI **every time the resource changes**, building a complete history.

### Configuration recorder

The **configuration recorder** is the engine that detects changes and creates CIs. You turn
it on per Region and choose which resource types to record (all, or a subset). Without the
recorder running, Config sees nothing.

### Delivery channel

Config delivers configuration **snapshots and history** to an **S3 bucket** and can notify an
**SNS topic** on changes.

```
Resource changes (EC2, S3, SG, IAM, ...)
        │
   Configuration Recorder ──► Configuration Item (CI snapshot)
        │                         │
        ▼                         ▼
   Config Rules evaluate     Delivery channel ──► S3 (history) + SNS (notify)
   COMPLIANT / NONCOMPLIANT
```

---

## 3. Config Rules

A **Config rule** continuously evaluates resources and marks each **COMPLIANT** or
**NONCOMPLIANT**. Rules evaluate on configuration change and/or on a periodic schedule.

| Rule type | What it is |
|-----------|-----------|
| **AWS managed rules** | 100+ prebuilt rules (`s3-bucket-server-side-encryption-enabled`, `restricted-ssh`, `ec2-volume-inuse-check`). Just enable and parameterize. |
| **Custom rules** | Your own logic backed by a **Lambda function** or a **Guard** (policy-as-code) expression, for organization-specific policies. |

```
Rule: s3-bucket-server-side-encryption-enabled
  bucket-a  → encrypted     → COMPLIANT
  bucket-b  → not encrypted → NONCOMPLIANT  ──► (optional auto-remediation)
```

> **Rule**: Config tells you *that* a resource is non-compliant; it does **not** block the
> action. To *prevent* a misconfiguration before it happens, use **SCPs**, **IAM policies**,
> or **permission boundaries** — Config is *detective*, not *preventive*.

### Conformance packs

A **conformance pack** is a deployable collection of Config rules **and** remediation actions,
packaged as a single YAML template. Deploy a whole compliance framework (e.g., a PCI or CIS
baseline) at once, and roll it out across an Organization.

---

## 4. Remediation (Auto-fix via SSM)

Config can **automatically remediate** non-compliant resources using **AWS Systems Manager
Automation documents**.

```
Resource NONCOMPLIANT (e.g., SG allows 0.0.0.0/0 on 22)
        │
   Config triggers SSM Automation document
        │
   Remediation runs: remove the offending rule / enable encryption / detach public IP
        │
   Resource re-evaluated ──► COMPLIANT
```

- **Automatic** remediation acts as soon as a resource is flagged.
- **Manual** remediation lets an operator trigger the same document on demand.

💡 Common exam answer: "Automatically remediate any S3 bucket that becomes public" →
Config rule (`s3-bucket-public-read-prohibited`) + **auto-remediation** via an SSM document.

---

## 5. Configuration History & Timeline

Because Config stores a CI on every change, you get a **timeline** per resource:

```
Resource: sg-0abc  (configuration timeline)
 ─────────────────────────────────────────────────────────────►
  t0 created      t1 added 80     t2 added 22 (0.0.0.0/0)   t3 removed 22
  COMPLIANT       COMPLIANT       NONCOMPLIANT               COMPLIANT
```

Each CI links to the **CloudTrail event** that caused the change, so you can pivot from
"*what* changed" (Config) to "*who* changed it" (CloudTrail). This is the foundation of
**drift detection** and change auditing.

---

## 6. Multi-Account, Multi-Region Aggregators

A **Config aggregator** collects compliance and configuration data from **multiple accounts
and Regions** into a single view — typically in a central security/audit account, often
across an entire AWS Organization.

> **Exam pattern**: "Single dashboard of compliance across all accounts and Regions" →
> Config **aggregator**. (Config itself is **regional** — the recorder runs per Region — so
> aggregation is how you get the org-wide picture.)

---

## 7. The Big One: CloudWatch vs CloudTrail vs Config

This three-way distinction is a near-guaranteed exam question. Match the *question word* to
the service.

| | **CloudWatch** | **CloudTrail** | **AWS Config** |
|---|---|---|---|
| **Question it answers** | *Is it healthy / performing?* | *Who did what (API audit)?* | *How is it configured / compliant?* |
| **Category** | Monitoring / observability | Auditing / governance | Configuration / compliance |
| **Core data** | Metrics & logs (time-series, events) | API call events (identity, time, IP) | Configuration items (resource state) |
| **Time orientation** | Now / trends | Activity log (past actions) | State now + change timeline |
| **Reacts via** | Alarms → SNS/ASG/EC2 | Metric filters (via CW Logs), Insights | Rules → COMPLIANT/NONCOMPLIANT → SSM remediation |
| **Prevent or detect?** | Detect (alert) | Detect (record) | Detect (evaluate) — *not* preventive |
| **Typical trigger phrases** | "CPU > 80%", "latency", "scale out", "memory metric" | "who deleted the bucket", "root login alert", "API audit for compliance" | "ensure all buckets encrypted", "detect SG opened to world", "compliance across accounts" |
| **Example** | Alarm when EC2 CPU > 70% → scale | Log every `DeleteBucket` to S3 for 7 years | Flag & auto-fix any public S3 bucket |

```
            ┌──────────────┬───────────────┬───────────────────────┐
            │  CloudWatch  │  CloudTrail   │      AWS Config        │
   focus →  │  HEALTH      │  WHO DID WHAT │  HOW IT'S CONFIGURED   │
            │  (perf)      │  (API audit)  │  (state + compliance)  │
            └──────────────┴───────────────┴───────────────────────┘
```

> **Mnemonic**: **W**atch = performance, **T**rail = the audit *trail* of actions, **C**onfig
> = the *configuration*. They are complementary, not alternatives — a mature account runs all
> three, and they cross-reference (a Config CI links to the CloudTrail event; CloudTrail logs
> route to CloudWatch alarms).

The hands-on walkthrough that picks between all three on realistic prompts is in
[Practical Examples — CloudTrail vs CloudWatch vs Config](../18_practical_examples/17_cloudtrail_vs_cloudwatch_vs_config.md).

---

## Key Exam Points

- ✅ AWS Config answers **"how is this resource configured, is it compliant, and how did it
  change?"** — detective, not preventive.
- ✅ **Configuration recorder** must be on; it produces **configuration items** delivered to
  **S3** (+ SNS).
- ✅ **Managed rules** (prebuilt) vs **custom rules** (Lambda/Guard). Bundle rules +
  remediation into **conformance packs**.
- ✅ **Auto-remediation uses SSM Automation documents** (e.g., auto-fix public S3 buckets).
- ✅ **Configuration timeline** + link to CloudTrail gives full change forensics / drift.
- ✅ **Aggregator** = multi-account, multi-Region compliance view (Config is regional).
- ✅ **CloudWatch = health, CloudTrail = who-did-what, Config = configuration/compliance.**

---

## Common Mistakes

- ❌ Expecting Config to **block** a misconfiguration — it detects and can remediate *after*;
  prevention is **SCP/IAM**.
- ❌ Confusing Config with CloudTrail. Config = resulting **state & compliance**; CloudTrail =
  the **API action** that caused it.
- ❌ Forgetting the **recorder** must be enabled — no recorder, no data.
- ❌ Assuming Config is global — it's **per Region**; use an **aggregator** for the org-wide view.
- ❌ Picking Config for performance/latency alerts — that's **CloudWatch**.

---

**Next**: [../13_security_services/01_encryption_and_kms.md — Encryption & KMS](../13_security_services/01_encryption_and_kms.md)
