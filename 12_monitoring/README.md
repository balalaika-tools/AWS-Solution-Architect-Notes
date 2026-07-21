# Observability, Governance & Automated Operations

> The three AWS services the exam asks you to tell apart: **CloudWatch** (is it healthy? — metrics, logs, alarms), **CloudTrail** (who did what? — API audit log), and **AWS Config** (how is it configured / is it compliant? — resource state and change history). Each note starts from the underlying concept (observability, audit logging, configuration management) before the AWS service.
> **Systems Manager** then uses that evidence to inventory, configure, patch, access, and remediate
> fleets across accounts and Regions.

[![AWS](https://img.shields.io/badge/AWS-CloudWatch%20%7C%20CloudTrail%20%7C%20Config-FF9900.svg?logo=amazoncloudwatch&logoColor=white)](https://aws.amazon.com/cloudwatch/)
[![Exam](https://img.shields.io/badge/Exam-SAA--C03-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_cloudwatch.md](01_cloudwatch.md) | CloudWatch | Metrics, logs, alarms, dashboards, agents, canaries, OAM cross-account observability, centralized subscriptions/streams, cardinality/cost controls, and KPI/SLO improvement scenarios. |
| [02_cloudtrail.md](02_cloudtrail.md) | CloudTrail | Event types and history, organization trails, immutable central S3/KMS design, validation, selectors/costs, Insights, EventBridge detection, Athena, and cross-account/Region investigation. |
| [03_config.md](03_config.md) | AWS Config | Recorders/items, rules and organization conformance packs, safe SSM remediation, delegated administration, organization aggregators, recording cost/scope, and Config vs CloudFormation drift. |
| [Systems Manager](#systems-manager-automated-operations-at-organization-scale) | Automated operations | Managed-node foundation, Inventory, Patch Manager, State Manager, Automation runbooks, Session Manager, Change Manager availability, and multi-account/Region rollout. |

---

## Reading Order

1. **CloudWatch** — the broadest service and the one every other AWS service emits to. Start here.
2. **CloudTrail** — once you know how CloudWatch surfaces *performance*, learn how CloudTrail records *API activity*.
3. **AWS Config** — closes the loop with *configuration state and compliance*, and ends with the three-way comparison that ties the section together.
4. **Systems Manager** — turn inventory, compliance findings, events, and approved changes into
   controlled fleet operations.

---

## Systems Manager: Automated Operations at Organization Scale

**AWS Systems Manager** is the operations control plane for managed nodes and many AWS resources. It
does not replace CloudWatch, CloudTrail, or Config:

- CloudWatch or EventBridge detects an operational condition.
- CloudTrail records the API actions.
- Config records/evaluates resource state.
- Systems Manager performs the approved inventory, configuration, patch, access, or remediation work.

### Establish the managed-node foundation

An EC2 instance, on-premises server, edge device, or VM becomes a **managed node** only when it can
authenticate and reach Systems Manager. Standardize these prerequisites before building automation:

1. Install and maintain **SSM Agent**. Some AWS images include it, but installed is not the same as
   running, supported, or registered.
2. Give the node a least-privilege instance profile/service role, or use Systems Manager's Default
   Host Management Configuration where supported. Hybrid/multicloud nodes need their registration
   and identity mechanism.
3. Provide outbound HTTPS to the required regional Systems Manager endpoints, directly or through
   VPC interface endpoints. Session Manager does not need inbound SSH/RDP, but it does need this
   outbound control channel, DNS, routes, and endpoint security-group/policy access.
4. Tag nodes with stable operational dimensions such as application, environment, owner, patch ring,
   and criticality. Automation targeted by tags is only as safe as the tag governance.
5. Monitor managed-node connectivity and agent version. An offline node is not patched or
   configured merely because a central policy exists.

Use **Quick Setup** to deploy common Systems Manager prerequisites and configurations across selected
Organizations OUs, accounts, and Regions. Run normal fleet administration from a delegated operations
account, not the Organizations management account, while respecting feature-specific exceptions.

```
Organizations management account
  enables trusted access / registers delegated operations account
                              │
                              ▼
Operations account: Quick Setup + Automation + central views
     │                │                 │
     ├── test OU      ├── production OU └── selected Regions
     ▼                ▼
managed nodes: SSM Agent + role + network path + governed tags
```

### Inventory: know what is actually installed

**Systems Manager Inventory** collects metadata from managed nodes, including applications,
operating-system details, AWS components, network configuration, services, and custom inventory. A
State Manager association using the inventory document defines what is collected and how often.

Use Inventory to answer fleet questions such as "which nodes still have this package version?" It
does not scan for arbitrary vulnerabilities or preserve every historical state by itself. For
multi-account/Region analysis, configure a **resource data sync** for each participating
account/Region to a central, encrypted S3 bucket and query the data with Athena. Use AWS Config's
`SSM:ManagedInstanceInventory` resource type when inventory change history is required.

Validate collection coverage, not only results: stale inventory can mean an offline agent,
association failure, missing permission, or a Region/account that was never enrolled.

### Patch Manager: policy, rollout rings, and evidence

**Patch Manager** compares a node with a patch baseline, reports missing updates with `Scan`, and
applies approved updates with `Scan and install`. Baselines define operating-system-specific approval
rules, approval delays, and explicit approved/rejected patches. A scan is compliance evidence; an
install is a potentially disruptive change.

The recommended organization pattern is a **patch policy in Quick Setup**, which can target accounts,
Regions, tags, and schedules from one configuration. As of July 2026, organization patch-policy
configurations must be created and maintained from the Organizations management account; the Quick
Setup delegated administrator does not support that configuration type. Keep access to that narrow
and operate other supported Systems Manager workflows from the delegated account.

A production patch policy should define:

- a short **scan** cadence and a controlled install window;
- patch baselines per OS and an approval delay that leaves time to test newly released updates;
- canary/dev, pre-production, and production rings with explicit promotion criteria;
- concurrency/error thresholds so a bad patch does not affect the fleet at once;
- reboot behavior, application drain/health checks, backup or immutable-image rollback strategy;
- exception ownership/expiry and central compliance reports.

Patch Manager can install a package update; it cannot prove the business application still works or
roll back every package safely. Run smoke/canary tests after each ring. For immutable fleets, prefer
patching and testing the image pipeline, then replacing instances, while still scanning deployed
nodes for evidence.

### State Manager, Run Command, Automation, and Maintenance Windows

These tools overlap, but their intent differs:

| Tool | Problem it solves | Use it for | Do not treat it as |
|------|-------------------|------------|--------------------|
| **State Manager association** | Keep a declared state applied on a schedule or when targets change | Agent/configuration settings, package presence, inventory collection, recurring compliance | A one-time emergency shell command |
| **Run Command** | Execute an imperative command on selected managed nodes without SSH | Bounded diagnostics or scripted fleet action with output/rate controls | A durable desired-state policy or multi-step transaction |
| **Automation runbook** | Orchestrate steps across AWS APIs, managed nodes, scripts, approvals, and branches | Repeatable remediation, AMI creation, recovery, deployment rollback | An unversioned collection of administrator privileges |
| **Maintenance Window** | Start disruptive tasks only during an approved time window | Coordinated patch/command/Automation/Lambda/Step Functions work | Continuous configuration enforcement |

State Manager targets nodes/resources by ID, tags, or resource groups and reports association
compliance. Associations normally run when created and when targets/documents change as well as on
their schedule; select the apply-at-next-cron behavior when an immediate run would be unsafe. Use
Maintenance Windows when a task must start only inside a defined change window.

### Automation runbooks: safe remediation as code

An **Automation runbook** is a versioned YAML/JSON SSM document whose steps can call AWS APIs, run
commands/scripts, branch, wait, assert state, and request approval. AWS-managed runbooks accelerate
common work; customer runbooks encode organization-specific checks and rollback.

Build production runbooks with this sequence:

1. Validate typed inputs, account/Region, current resource state, tags, and an approved change or
   incident reference.
2. Assume a least-privilege execution role. Separate the role that starts Automation from the role
   that changes the target resource, and pin the tested runbook version.
3. Make each mutation idempotent; record old state before changing it and define compensating or
   rollback steps where the API supports them.
4. Use approval for high-risk work, then canary targets, bounded concurrency, an error threshold,
   timeouts, and CloudWatch alarms that can halt a rollout.
5. Verify the intended Config rule, SLO, health check, or application behavior and publish outputs to
   the incident/change record. A `Success` execution without a postcondition is incomplete.

Config remediation and EventBridge/CloudWatch alarm events can start a runbook. Keep automatic
actions narrow and deterministic; ambiguous diagnosis creates an OpsItem/page for a human.

For **multi-account, multi-Region Automation**, use a dedicated or delegated central account with
`AWS-SystemsManager-AutomationAdministrationRole`, and an
`AWS-SystemsManager-AutomationExecutionRole` in each target account. Select accounts/OUs and Regions,
share custom runbooks to targets, and set both account-Region and per-resource concurrency/error
limits. Start with one canary account/Region and confirm child execution outputs before expanding.

### Session Manager: controlled administrative access

**Session Manager** provides shell and tunnel access to managed nodes through the SSM Agent, without
inbound SSH/RDP ports, public IPs, bastions, or distributed SSH keys. It improves the network posture,
but authorization and evidence still need design:

- Grant `StartSession` only for approved node tags and session documents; restrict port forwarding
  and interactive shells separately. Use short-lived IAM Identity Center roles and require MFA.
- Configure session preferences for idle/max duration, shell profile, CloudWatch Logs or S3 output,
  and optional customer managed KMS encryption. Give the logging service and operators only the KMS
  permissions they need.
- CloudTrail records `StartSession`, `ResumeSession`, and `TerminateSession` API activity. Session
  output logging records interactive content only when the session type supports it.
- **SSH-over-Session-Manager and port-forwarding contents cannot be logged by Session Manager.** If
  command-level evidence is mandatory, disallow those session documents or add host/application
  auditing; do not claim CloudTrail records every command typed.
- Alert on unexpected sessions and periodically reconcile active sessions, role access, node tags,
  and logging delivery.

Session Manager is for controlled human access. Prefer Run Command or Automation for repeatable work
because it produces consistent inputs, steps, rate limits, and machine-readable results.

### Change Manager and approval workflows

**Systems Manager Change Manager** provides change templates, requests, approvers, calendars,
execution, and reporting around Automation runbooks. Existing customers can use it to separate the
requester, approver, and implementer and to prevent execution outside approved change windows.

⚠️ **Current availability**: Change Manager has not been open to new customers since **November 7,
2025**. Existing customers can continue using it. New customers that need enterprise ITSM change
requests, related incidents, approvals, and reporting should evaluate AWS Partner Network solutions,
while keeping the underlying operational action in versioned, least-privilege Automation runbooks
where appropriate. Do not design a new environment around a service it cannot activate.

### Multi-account operating model and failure boundaries

| Responsibility | Recommended owner |
|----------------|-------------------|
| Trusted access, initial delegated-administrator registration, organization patch-policy exception | Organizations management account with tightly controlled access |
| Quick Setup, inventory views, runbook library, Automation rollout, central OpsItems | Delegated operations account |
| Node role, SSM Agent/network health, application drain and post-change tests | Workload team under organization baseline |
| Runbook approval, execution-role boundaries, CloudTrail/Config evidence | Operations and security/change-control roles with separated duties |

Roll policies out by OU/account and Region, include new-account onboarding, and define what happens
when a node is offline, an execution role is missing, an opt-in Region is not configured, a KMS key
denies logging, or concurrency/error thresholds stop a rollout. Central visibility does not make a
failed target compliant: retain failed-command/Automation output, create an owner-visible OpsItem or
ticket, retry only known transient failures, and expire exceptions.

---

## Governance Services to Recognize

| Service | Exam clue |
|---------|-----------|
| **CloudFormation** | Infrastructure as code, repeatable stacks, drift detection. |
| **Systems Manager** | Fleet inventory, patch/compliance policy, desired state, controlled sessions, and versioned operational automation. |
| **Service Catalog** | Let teams launch approved products/stacks with governance. |
| **Control Tower** | Create and govern a multi-account landing zone. |
| **Organizations** | Multi-account structure, consolidated billing, SCP guardrails. |
| **Trusted Advisor** | Best-practice checks for cost, security, fault tolerance, performance, service limits. |
| **Compute Optimizer** | Right-size EC2/EBS/Lambda/ECS based on utilization. |
| **Health Dashboard** | AWS service/account events that affect your resources. |
| **Service Quotas** | View/request quota increases and avoid limit-related scaling failures. |

---

## Prerequisites

- [IAM & Identity](../02_iam/README.md) — CloudTrail records the IAM principal behind every API call; Config and CloudWatch use service roles.
- [Messaging & Events](../11_messaging/README.md) — alarms and events fan out through **SNS** and **EventBridge**.
- [Storage](../05_storage/README.md) — CloudTrail trails and Config snapshots land in **S3**; Athena queries them in place.
