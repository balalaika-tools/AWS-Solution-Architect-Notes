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
a new CI when the resource changes under **continuous** recording, or the latest changed state in
the last 24 hours under **daily** recording, building the history selected by your recorder policy.

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

Treat a pack as versioned policy-as-code, not a one-time console setup. Test the rules and
remediations in a sandbox OU, store the approved template in a controlled repository/S3 bucket or
SSM document, then deploy an **organization conformance pack** from the management or delegated
administrator account. Deployment creates the rules and remediation configurations in member
accounts/Regions; the central Config aggregator only displays their results and cannot deploy them.

Scope pack parameters and account exclusions explicitly. A generic framework template is a starting
point, not proof of compliance: map every rule to the organization's control, resource scope,
evidence owner, exception process, and remediation risk. Monitor organization deployment status so a
new or failed account does not become a silent coverage gap.

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

For production remediation, the runbook needs more than an API call:

1. Use a least-privilege `AutomationAssumeRole` in the target account and pass the evaluated resource
   ID as an explicit parameter. Pin an approved SSM document version.
2. Re-read the live resource and verify the offending condition before changing it. Config's
   automatic-remediation bootstrap can act from a stale compliance snapshot, including after a
   resource has already become compliant.
3. Make the action idempotent and preserve business-critical properties. Record outputs and notify
   the resource owner; use manual approval for destructive, ambiguous, or outage-causing fixes.
4. Set concurrency, error percentage, retry window, and maximum attempts so one defective document
   cannot change the whole fleet. Repeated failures become remediation exceptions that need review.
5. Re-evaluate the Config rule and verify the application after Automation succeeds. A successful
   runbook execution proves the steps ran, not that the resource is now compliant or usable.

Use EventBridge/SNS for escalation when remediation fails. Do not let Config remediation fight an
IaC controller: if CloudFormation owns the resource, correct the template and redeploy it, or the next
stack update can recreate the noncompliant setting.

---

## 5. Configuration History & Timeline

With continuous recording, Config stores a CI on every detected change and gives you a **timeline**
per resource. Daily recording retains only the latest changed state for each 24-hour period, so it
does not provide the same intermediate detail:

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

### Organization operating model

```
Management account
  enables Config trusted access / registers delegated administrator
                              │
                              ▼
Security or compliance account
  organization conformance packs ──► rules/remediation in member accounts
  organization aggregator      ◄──── configuration/compliance from each Region
                              │
                              ▼
                    central query and investigation
```

Register a member security account as the Config delegated administrator for aggregation and
multi-account rule/conformance-pack administration. Keep the Organizations management account for
the actions that only it can perform. The Config integrations for aggregation and organization rules/
packs use different service principals, so validate both during setup rather than assuming one
delegation enables every Config workflow.

An organization aggregator automatically discovers organization accounts, and can include all
current and future supported Regions. It is a **read-only replicated view** and has no added
aggregator charge. It does not:

- turn on a configuration recorder or delivery channel in a source account/Region;
- deploy Config rules, conformance packs, or remediation;
- grant permission to mutate a source resource.

Bootstrap recording and the baseline rules in every required account/Region first, then alarm on
missing/stale source status. Otherwise a green aggregator can mean "nothing was recorded," not
"everything is compliant."

### Recording scope and cost

Config cost grows mainly with recorded configuration items and rule/conformance-pack evaluations;
SSM Automation, S3, and SNS add their own charges. Choose the recorder policy from detection needs:

| Recording choice | Use when | Trade-off |
|------------------|----------|-----------|
| **All current and future supported types, with exclusions** | Governance should cover new AWS resource types automatically | Broadest coverage; review noisy/ephemeral types and new-service cost |
| **Specific resource types** | A tightly scoped account has a documented control boundary | Lower volume, but newly used resource types are invisible until explicitly added |
| **Continuous** | Security-sensitive state must be detected and investigated change by change | More CIs/evaluations for high-churn resources |
| **Daily** | The latest daily state is sufficient for lower-risk/high-churn types | Can miss intermediate drift and delays detection; unsupported for some Config compliance types |

Ephemeral compute, snapshots, and compliance resources can create large CI/evaluation volume. Use
daily overrides or exclusions only after documenting the visibility loss; keep continuous recording
where Firewall Manager or rapid security detection depends on it. Record global IAM resource types
in one supported Region to avoid duplicate CIs, and verify newer Region/resource coverage instead of
assuming every type exists everywhere.

Use Cost Explorer/CUR plus CI and evaluation counts to find high-churn resource types and rules that
evaluate too broadly. An aggregator itself is free, but the source recorders and evaluations are not.

---

## 7. Multi-Account Drift Investigation

Suppose the organization aggregator flags an internet-facing security group in a production account:

1. Confirm the aggregator source account, Region, resource ID, rule, evaluation time, and annotation.
   Aggregated data can lag, so open the source account/Region for the current evaluation.
2. Use the Config resource timeline to compare the last compliant and first noncompliant CIs and
   inspect related ENIs, instances, load balancers, and VPCs. This answers **what state changed**.
3. Pivot from the CI to CloudTrail and follow the role session/API call. This answers **who or what
   changed it**. Search other accounts/Regions if the principal assumed roles or automation ran at
   organization scale.
4. Identify ownership. If the security group belongs to a CloudFormation stack, run CloudFormation
   drift detection and compare the live property with the template's expected property.
5. Correct the source of truth, execute the approved remediation with limited concurrency, then
   re-evaluate Config and the stack. Preserve the before/after CIs, trail event, approval, Automation
   execution, and final compliance result as evidence.

**Config drift/compliance and CloudFormation drift are different questions:**

| Tool | Comparison | Coverage | Use it for |
|------|------------|----------|------------|
| **AWS Config** | Current/recorded resource state against a Config rule; also state over time | Recorded supported resources, whether or not created by IaC | Organization policy, history, relationships, and "when did this become noncompliant?" |
| **CloudFormation drift detection** | Actual properties against the expected properties explicitly represented in a stack template | Supported resources/properties in that stack | "Did someone change this stack resource outside its declared template?" |

A resource can be `IN_SYNC` with an insecure template and still be noncompliant in Config. It can
also be Config-compliant but drifted from a stricter template. Use both results before deciding
whether to update the template, restore it, approve an exception, or change the policy.

---

## 8. The Big One: CloudWatch vs CloudTrail vs Config

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
- ✅ **Organization aggregator** = read-only multi-account, multi-Region view; it does not enable
  recorders or deploy policy. **Organization conformance packs** deploy rules/remediation.
- ✅ Use continuous recording for fast/high-value detection and daily/exclusions only when the
  lower cost justifies losing intermediate state. High-churn resources drive CI/evaluation cost.
- ✅ Config compliance compares state to policy; **CloudFormation drift** compares a stack
  resource to expected template properties. Neither substitutes for the other.
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
- ❌ Creating an aggregator and assuming member accounts are now recorded. Empty source coverage
  can look clean; deploy and monitor recorders/rules separately.
- ❌ Automatically remediating from a stale evaluation without a live precondition, idempotency,
  concurrency/error limits, or a least-privilege Automation role.
- ❌ Auto-fixing a CloudFormation-owned resource without correcting its template, causing the next
  deployment to recreate the violation.

---

**Next**: [../13_security_services/01_encryption_and_kms.md — Encryption & KMS](../13_security_services/01_encryption_and_kms.md)
