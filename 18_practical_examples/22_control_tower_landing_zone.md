# Professional Build: A Multi-Account Landing Zone

> **Scenario**: A company has three product teams and regulated customer data. It needs accounts
> that are safe by default, federated workforce access, central evidence, and an exception process
> that does not make the management account an everyday administration account.

This is a **governance system**, not an account-creation script. AWS Control Tower orchestrates a
landing zone on AWS Organizations; Organizations supplies the account hierarchy and policy types;
IAM Identity Center supplies workforce access. None of those replaces workload-level IAM,
networking, logging, backup, or incident ownership.

---

## 1. Target Organization

```
organization management account       emergency organization changes only
│
├── Security OU
│   ├── Log Archive account            immutable organization evidence
│   └── Security Tooling account       audit/admin, findings and delegated services
├── Infrastructure OU
│   ├── Network account                TGW, DX/VPN, DNS and optional inspection
│   └── Shared Services account        directory, build artifacts, approved images
├── Workloads OU
│   ├── Production accounts            one or more accounts per product boundary
│   └── Nonproduction accounts         development and test separated from prod
├── Sandbox OU                         disposable, tightly cost-limited experiments
└── Suspended/Quarantine OU            accounts denied normal workload activity
```

Use an OU for a **common control intent**, not merely for the reporting chart. Splitting production
from nonproduction lets SCPs, controls, change authority, backup policies and cost guardrails
differ without hundreds of account-specific exceptions.

### Policy layers

| Layer | Purpose | Important boundary |
|-------|---------|--------------------|
| Control Tower **controls** | Preventive, detective or proactive governance for governed OUs | A detective control reports drift; it does not necessarily block or repair it. |
| **SCP** | Maximum permissions available to principals in member accounts | It never grants access and it does not replace identity/resource policies. Test denials in a canary OU. |
| **Resource control policy (RCP)** | Maximum permissions available to supported resources in member accounts | It constrains resource-based access; identity authorization must still allow the request. Validate service and action support. |
| **Declarative policy** | Centrally enforced settings for supported services | Unlike an SCP, it declares service configuration. Confirm current policy-type and Region support. |
| IAM Identity Center permission set | Workforce role and session permissions provisioned into accounts | Separate read-only, operator, developer, security and break-glass responsibilities. |
| Tag, backup and AI/data policies | Organization-wide governance for supported services | Validate inheritance and the effective policy at child OUs/accounts. These and declarative policies are distinct mechanisms. |
| Workload IAM/resource policies | The application's actual least-privilege access | Product owners still own these permissions. |

---

## 2. Build Sequence

1. **Protect the management account.** Use hardware-backed MFA for root, no root access keys,
   tightly limited organization administrators, alternate contacts and a monitored break-glass
   procedure. Do not run normal workloads there.
2. **Choose home Regions and data residency.** Control Tower has a home Region; governed Regions,
   denied Regions, log destinations and service availability must match legal and recovery needs.
   Never apply a Region-deny policy without exceptions for required global services.
3. **Deploy the landing zone.** Create the Security accounts and baseline OUs. Register existing
   OUs deliberately; enrollment can surface resources that conflict with controls. In landing zone
   4.0 and later, do not assume that controls an older landing zone called mandatory are enabled
   automatically—select and verify the required controls and governed OU scope explicitly.
4. **Federate people.** Connect the enterprise identity provider to IAM Identity Center, map groups
   to permission sets and remove long-lived IAM users except documented emergency/service cases.
5. **Delegate services.** Assign supported security/operations services to the Security Tooling
   account—such as GuardDuty, Security Hub, Inspector, Macie, Config aggregation and AWS Backup—so
   their administrators do not require the organization management account.
6. **Centralize evidence.** Send organization CloudTrail and Config history to the Log Archive
   account with restrictive S3/KMS policies, retention and immutability appropriate to policy.
   Alert when trails, configuration recorders or required controls change.
7. **Create the account product.** Account Factory, or Account Factory for Terraform (AFT) when a
   Terraform workflow is required, collects owner, environment, cost-center, data classification,
   network and support metadata. Bootstrap approved IAM, logging, budgets, backup, DNS, patching and
   networking through versioned automation.
8. **Establish lifecycle workflows.** Account update, OU move, quarantine, closure, evidence
   retention and owner transfer need the same rigor as account creation.

Do not place application deployment code in the landing-zone pipeline. Keep platform baselines and
product releases independently versioned so an app release cannot weaken organizational controls.

### A small enforceable artifact

This SCP is intentionally narrow: member-account principals cannot remove an account from the
organization. It does not grant another permission and it does not affect the management account.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyMemberAccountLeavingOrganization",
    "Effect": "Deny",
    "Action": "organizations:LeaveOrganization",
    "Resource": "*"
  }]
}
```

Attach it to a canary OU first. The versioned account-vending request should include, at minimum,
`accountName`, unique email, target OU, owner group, environment, cost center, data classification,
network profile and support tier. Schema validation rejects missing owners and unapproved production
OUs before Account Factory or AFT runs.

---

## 3. Safe Change and Exception Flow

Roll a new SCP or control through a nonproduction **canary OU**, inspect denied CloudTrail events and
service behavior, then expand in waves. An exception must record owner, business reason, compensating
control, scope and expiry. Prefer moving a narrowly scoped account to an exception OU or using a
conditioned policy over editing a broad production SCP under pressure.

If a change blocks workloads:

1. stop the rollout and preserve the effective policies and failing API evidence;
2. use the pre-tested organization administrator path to revert the exact change;
3. restore the previous OU/policy assignment, then confirm workload and logging recovery;
4. correct the canary tests before reapplying—do not leave an untracked permanent exception.

---

## 4. Acceptance Evidence

- A newly vended account reaches the correct OU with owner/cost/data tags and baseline stacks.
- A normal administrator cannot disable the organization trail, leave the organization, make S3
  evidence public, or use an unapproved Region; the expected explicit deny appears in CloudTrail.
- Central security administrators can see findings without logging into each workload account.
- A federated operator can assume only the intended permission-set role; session and source identity
  are visible in audit records.
- A quarantine exercise removes workload access without destroying evidence or the recovery path.
- Drift, account-vending failure, budget breach and root activity create owned alerts.

Track account-vending lead time, baseline failure rate, control compliance, exception age, finding
response time and unallocated spend. These KPIs show whether governance enables delivery or merely
creates tickets.

---

## 5. Cost, Limits, and When Not to Use It

Organizations and Control Tower coordination are only part of the bill: Config evaluations,
CloudTrail data events, retained logs, security-service coverage, NAT/TGW inspection and support
can dominate. Apply high-volume selectors and retention based on threat/compliance needs, not an
unbounded “log everything forever” default.

Control Tower is appropriate when the organization wants an AWS-managed landing-zone lifecycle and
can adopt its supported structure. A very small environment may begin with Organizations and IaC,
while a heavily customized existing organization may require a staged enrollment or a custom
landing-zone design. The decision is about operating model and migration risk, not account count
alone.

**Related**: [IAM and Organizations](../02_iam/04_organizations_sts_federation.md) ·
[IaC delivery](24_iac_cicd_rollback.md) · [Central inspection and logging](23_centralized_inspection_and_logging.md)
