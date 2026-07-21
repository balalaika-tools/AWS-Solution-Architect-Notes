# Organizations, STS & Federation

> **Who this is for**: Engineers ready to scale identity beyond one account — multi-account
> governance, temporary credentials, and external identity providers. Read
> [03_iam_policies.md](03_iam_policies.md) first.

---

## 1. AWS Organizations

**AWS Organizations** lets you centrally manage **many AWS accounts** as one tree. You start
with a **management account** (the payer, formerly "master") and add **member accounts**.

```
                ┌──────────────────────┐
                │  Management account  │  ← payer; creates the Org; applies SCPs
                └──────────┬───────────┘
                           │
                    ┌──────┴───────┐  Root (of the Org)
                    │              │
              ┌─────▼─────┐  ┌─────▼─────┐
              │  OU: Prod │  │  OU: Dev  │   ← Organizational Units (folders)
              └─────┬─────┘  └─────┬─────┘
                ┌───┴───┐      ┌───┴───┐
            acct  acct      acct  acct        ← member accounts
```

- **OU (Organizational Unit)** — a folder grouping accounts so you can apply policy by group.
  OUs can nest.
- **Consolidated billing** — all member accounts roll up to the management account's bill.
  Benefits: one invoice, and **combined usage unlocks volume/tiered pricing and shared Reserved
  Instance / Savings Plan discounts** across accounts.

💡 Why multiple accounts at all? Strong **blast-radius isolation**, separate dev/prod/security
boundaries, and per-account billing visibility — a recurring "best practice" exam theme.

---

## 2. Service Control Policies (SCPs) vs IAM Policies

An **SCP** is an Organizations policy attached to the Org root, an OU, or an account. It sets
the **maximum available permissions** (a *guardrail*) for the identities **in those accounts** —
it **grants nothing on its own**.

| | **SCP** | **IAM policy** |
|---|---------|----------------|
| Lives in | AWS Organizations | IAM (within an account) |
| Attached to | Org root / OU / account | User, group, or role |
| Effect | **Caps** the maximum — never grants | **Grants** (identity-based) |
| Applies to | IAM identities and root in member accounts, but not service-linked roles | The specific identity |
| Affects the **root user**? | Restricts root in member accounts | No (root is unrestricted) |
| Management account | **SCPs do not restrict it** | n/a |

> **Rule**: An action is allowed only if it is **(a)** granted by an IAM policy **and**
> **(b)** *not blocked* by any SCP in the path from the Org root down to the account. SCP **∩**
> IAM. An empty/absent grant on either side = deny.

```
   Allowed?  =  SCP allows it  AND  IAM policy allows it  AND  no explicit Deny
                (guardrail)         (actual grant)
```

⚠️ Classic trap: an admin in a member account gets `AccessDenied` despite having
`AdministratorAccess`. Cause: an **SCP** on the OU denies that action. The IAM policy is fine;
the guardrail wins.

⚠️ SCPs **do not** affect the **management account**, and (with the default `FullAWSAccess`
attached) they restrict only what you explicitly deny or scope.

### 2.1 Resource Control Policies: the resource-side guardrail

An **RCP** is the complement to an SCP. An SCP caps what **principals in a member account** can
do; an RCP caps which permissions policies can grant to **supported resources in a member
account**, including access by principals from outside the organization.

```
SCP on account B  → caps a role that belongs to B, wherever it sends a request
RCP on account C  → caps a supported resource that belongs to C, whoever calls it
```

RCPs attach to the organization root, OUs, or accounts and **never grant** access. Use an RCP,
for example, to deny access to organization resources unless the caller belongs to an approved
organization. The normal identity/resource-policy allows are still required. RCP availability
is service-specific, so do not substitute one for a resource policy on an unsupported service.

---

## 3. AWS STS — Temporary Credentials

**STS (Security Token Service)** issues short-lived credentials. It's the engine behind every
role assumption. Key API calls:

| API | Caller | Use |
|-----|--------|-----|
| `AssumeRole` | An IAM user/role | Switch roles, cross-account access |
| `AssumeRoleWithSAML` | A SAML 2.0 IdP user | Enterprise SSO federation |
| `AssumeRoleWithWebIdentity` | OIDC / web identity (Cognito, Google, an OIDC IdP) | Mobile/web apps, IRSA for EKS |
| `GetSessionToken` | A user | MFA-protected temporary creds |

Returned creds = **AccessKeyId + SecretAccessKey + SessionToken**. For `AssumeRole`, the caller
requests 15 minutes up to the role's configured 1–12-hour maximum (default maximum: 1 hour); a
chained role session is capped at 1 hour. Other STS APIs have their own duration limits. A role
session uses the **role's** permissions, narrowed by any session policy, not the caller's
original service permissions.

```
   Caller ──(1) sts:AssumeRole(RoleArn)──► STS
                                            │ checks the role's TRUST policy
                                            │
          ◄──(2) temp creds (expire) ───────┘
   Caller ──(3) signed request with temp creds──► target service
                                            (uses the ROLE's permissions)
```

---

## 4. Cross-Account Access Pattern

The canonical "give account A access to account B" answer:

1. In **account B**, create a role with the needed **permissions policy** and a **trust policy**
   naming account A as principal (`sts:AssumeRole`).
2. In **account A**, give the user/role permission to call `sts:AssumeRole` on B's role ARN.
3. A's principal calls `AssumeRole` → STS returns temp creds → A operates in B.

```
   ┌──────────── Account A ────────────┐      ┌──────────── Account B ────────────┐
   │  IAM Role/User                    │      │  Cross-account Role               │
   │  identity policy:                 │      │  trust policy: "Principal: A"     │
   │   Allow sts:AssumeRole on B-role  │──────┤  permissions policy: what to do   │
   └───────────────────────────────────┘ STS  └───────────────────────────────────┘
```

✅ For trusting a **third-party vendor**, add an `sts:ExternalId` condition to the trust policy
to prevent the **confused deputy** problem (see [03_iam_policies.md](03_iam_policies.md)).

---

## 5. Identity Federation

**Federation** lets users authenticate with an **external identity provider (IdP)** and receive
temporary AWS credentials — **no IAM user per person**. AWS trusts the IdP; the IdP proves who
the user is; STS issues role credentials.

| Federation type | Protocol | Typical users | STS call |
|-----------------|----------|---------------|----------|
| **SAML 2.0** | SAML | Corporate workforce (AD, Okta, etc.) | `AssumeRoleWithSAML` |
| **Web identity / OIDC** | OIDC / OAuth | Mobile/web app users; Google, Cognito; EKS IRSA | `AssumeRoleWithWebIdentity` |

> **Key insight**: Federated identities map to **roles**, never to IAM users. The IdP handles
> authentication; the role handles authorization. This is why "users sign in with their
> corporate credentials" → **role + federation**, not "create IAM users."

💡 For consumer-facing apps (sign in with Google/Apple, guest access), **Amazon Cognito**
brokers web-identity federation so your app code never touches STS directly.

---

## 6. IAM Identity Center (successor to AWS SSO)

**IAM Identity Center** (formerly **AWS SSO**) is the recommended way to manage **workforce**
access across an Organization. One sign-in, central control.

- Connects to an external IdP (Okta, Entra ID/Azure AD, etc.) or its built-in directory.
- **Permission sets** define what users get; they're provisioned as roles into member accounts.
- Users get a portal listing the accounts/roles they may assume — all via **temporary**
  credentials.

| | **IAM users** | **IAM Identity Center** |
|---|---------------|--------------------------|
| Credentials | Long-lived, per account | Temporary, central SSO |
| Scope | One account | Whole Organization |
| Best for | Legacy/edge cases | Human workforce access (the modern default) |

✅ Exam default for "central human access to many accounts" = **IAM Identity Center**, not IAM
users in each account.

---

## 7. How AWS Services Get Access to Resources

When one AWS **service** acts on resources, two patterns appear:

- **Service role** — *you* create a role the service assumes (e.g. a Lambda execution role, an
  EC2 instance role). The trust policy names the service principal (e.g.
  `ec2.amazonaws.com`). You control its permissions.
- **Service-linked role** — a predefined role **owned by the service**, created/managed for you
  (e.g. for Elastic Load Balancing, RDS). You can't freely edit its trust policy; it exists so
  the service can call other AWS APIs reliably.

💡 Guard service-role trust policies with `aws:SourceArn` / `aws:SourceAccount` conditions to
prevent confused-deputy abuse.

---

## 8. Tag Policies

**Tag Policies** are an Organizations policy type — separate from SCPs — that standardize tag
keys and values across member accounts. They define allowed keys, allowed values, and whether
key names are case-sensitive.

```
   SCP         → restricts WHAT actions IAM identities can take
   Tag Policy  → restricts HOW resources must be tagged (key names, value lists, case)
   These are orthogonal — both can apply to the same OU simultaneously.
```

- **Reporting mode** (default): non-compliant tags are flagged in the Tag Editor console; API
  calls are not blocked.
- **Enforcement mode**: non-compliant `CreateResource` or `TagResource` calls are rejected.
  Must be explicitly enabled per account.

**Common pattern** — enforce that every resource in `OU: Prod` carries a `CostCenter` tag from
an approved list, and that the `Environment` key is always capitalized (`Production`, not
`production`).

✅ Combine with an SCP `Condition` that requires `aws:RequestTag/CostCenter` on EC2 launches —
the Tag Policy ensures the format, the SCP ensures presence.

---

## 9. AWS Control Tower

**Control Tower** builds and operates a prescriptive **landing zone** on top of Organizations.
It coordinates IAM Identity Center, CloudFormation, AWS Config, CloudTrail, and Organizations
policies and exposes their governance state in one console.

> **Key insight**: Control Tower does not replace Organizations. Organizations supplies the
> account tree, policy framework, billing, trusted access, and delegated administration.
> Control Tower adds an opinionated baseline, account vending, a control catalog, and drift
> monitoring.

### 9.1 Landing-zone account and OU design

A practical starting structure separates organization administration, evidence, security
operations, shared infrastructure, and workloads:

```
Organization root
├── Management account          ← billing and Organizations/Control Tower only
├── Security OU (foundational)
│   ├── Log Archive account     ← protected central log destination
│   └── Audit account           ← security/compliance cross-account access and notifications
├── Infrastructure OU
│   ├── Network account
│   └── Shared Services account
├── Workloads OU
│   ├── Prod OU
│   └── NonProd OU
└── Sandbox OU                  ← tighter spend and service guardrails
```

- Keep application workloads out of the **management account**. A compromise there carries
  organization-wide authority, and SCPs do not restrict it.
- The **Log Archive account** receives organization audit and configuration logs. Restrict
  writes to logging services, restrict human access, encrypt the destination, and choose
  retention or Object Lock to match evidence requirements.
- The **Audit account** gives security and compliance operators cross-account visibility and
  receives aggregated notifications. A larger environment often adds a separate security
  tooling account for GuardDuty, Security Hub CSPM, and incident-response automation.
- Apply controls to workload OUs according to their risk. Avoid one flat OU: production,
  sandbox, and regulated workloads rarely need identical guardrails.

Control Tower creates or brings in the Log Archive and Audit shared accounts in its
foundational Security OU. Do not move these accounts or modify Control Tower-managed resources
outside supported workflows; those changes create drift.

### 9.2 Account Factory: repeatable account vending

**Account Factory** provisions new or enrolled accounts through a standardized workflow rather
than a ticket full of manual steps. A request selects details such as account name, email, OU,
and IAM Identity Center user access. The new account receives the landing-zone baseline and the
controls inherited from its OU.

Use **Account Factory Customization** for a CloudFormation or Terraform blueprint, or **Account
Factory for Terraform (AFT)** for a GitOps-style provisioning and customization pipeline. Put
organization-specific configuration—VPC baselines, backup policies, security integrations,
and mandatory tags—in versioned customizations. Account Factory is for account provisioning
and baseline customization, not for deploying every application workload.

### 9.3 Preventive, proactive, and detective controls

Control **behavior** answers *when* a violation is handled. This is separate from control
**guidance** (`mandatory`, `strongly recommended`, or `elective`).

| Behavior | When it acts | Implementation | Result |
|----------|--------------|----------------|--------|
| **Preventive** | At an API authorization decision | Organizations SCPs, RCPs, or declarative policies that enforce service settings | Blocks an action that would violate the rule |
| **Proactive** | Before a CloudFormation resource is provisioned | CloudFormation hooks | Fails a non-compliant CloudFormation deployment; does not inspect resources created outside CloudFormation |
| **Detective** | After a resource exists or changes | AWS Config rules | Reports compliant/non-compliant state; does not undo the change |

Choose the behavior from the requirement. "Must never be created" calls for a preventive
control where supported, or a proactive control for CloudFormation deployments. "Find and
remediate anything that drifts" calls for a detective control plus an EventBridge or Systems
Manager remediation path. Enabling a detective control alone does not prevent exposure.

### 9.4 Drift is an operational state, not just a failed Config rule

**Drift** occurs when the actual landing zone no longer matches the resources and relationships
Control Tower expects—for example, an account is moved outside the registered OU workflow, a
managed role or StackSet is changed, or an expected Organizations policy is detached. This is
different from a detective control reporting that an application resource is non-compliant.

Control Tower surfaces drift in its console and sends relevant notifications to the Audit
account. Repair the cause through the supported action: reset/update the landing zone,
re-register an OU, or update/re-enroll the account. Some landing-zone drift blocks account
enrollment, so treat drift alerts as platform incidents. Do not "repair" a Control
Tower-managed SCP or StackSet by hand; Control Tower may overwrite it or continue to report
drift.

### 9.5 When Organizations alone is the better fit

Use **Control Tower** when the requirement is a standardized landing zone, governed account
vending, prebuilt controls, and a governance dashboard with drift detection. This is the usual
answer for a company starting or standardizing a large multi-account estate.

Use **Organizations without Control Tower** when you need only consolidated billing, OUs and
organization policies, or when an established platform already provides custom account
provisioning, logging, identity, and compliance automation. Organizations is also the cleaner
choice when Control Tower's prescriptive resources and lifecycle would conflict with a mature
landing zone. The trade-off is ownership: your platform team must build and operate account
vending, baseline deployment, evidence collection, drift detection, and remediation.

⚠️ Exam pattern: "Apply a security baseline automatically to every new account and report
drift" → **Control Tower + Account Factory**. "Centralize billing and apply a few SCPs to an
existing custom account hierarchy" → **Organizations alone** may be sufficient.

---

## 10. Organization-Wide Operations Patterns

Organizations becomes useful at scale when member accounts enroll in central services
automatically. The management account establishes authority; day-to-day service operations
belong in dedicated member accounts.

### 10.1 Trusted access and delegated administrators

These are related but not interchangeable:

- **Trusted access** lets an AWS service use Organizations information and service-linked roles
  to perform supported operations across member accounts. Only the management account should
  enable it, preferably through the integrated service's console or API.
- A **delegated administrator** is a member account authorized to administer one integrated
  service for the organization. Delegation reduces daily use of the management account; it
  does not make that member an Organizations management account, and capabilities vary by
  service.

Use a security tooling account as delegated administrator for security services and an
infrastructure deployment account for StackSets. Delegated administrators can have very broad
reach, so protect them with MFA, short-lived access, approval workflows, and narrow operator
roles. Disabling trusted access before removing service configuration can strand resources or
break automation; follow the service-specific removal order.

### 10.2 Deploy baselines with CloudFormation StackSets

Enable trusted access for **CloudFormation StackSets**, then use **service-managed
permissions** from the management account or a delegated administrator. StackSets creates the
required service roles in target accounts, so you do not bootstrap an execution role manually
in every account.

Target OUs and Regions, enable automatic deployment for accounts that later join those OUs,
and choose whether to retain or delete stack instances when an account leaves. Set concurrency
and failure tolerance deliberately for production rollouts. Typical StackSets install Config
recorders, IAM access roles, EventBridge forwarding rules, security agents, and baseline
alarms. Do not modify Control Tower-owned StackSets; deploy your baseline in separate stack
sets to avoid drift.

### 10.3 Central security, audit, and event services

| Capability | Organization-wide pattern | Important boundary or failure mode |
|------------|---------------------------|------------------------------------|
| **CloudTrail** | Create an organization trail and deliver logs to an encrypted S3 bucket in Log Archive; delegate CloudTrail administration where appropriate | Protect the destination bucket and KMS key. An organization trail supplies activity evidence, but data events and Insights require deliberate configuration and can add cost |
| **AWS Config** | Enable recorders in required account/Region pairs; aggregate configuration and compliance into a delegated administrator; deploy organization rules or conformance packs | An aggregator **does not enable recording** in source accounts. A missing recorder or Region is a visibility gap |
| **Security Hub CSPM** | Use a security account as delegated administrator; apply central configuration policies from a home Region across linked Regions, accounts, and OUs | Aggregation is not universal enablement unless the central policy enables the service, standards, and controls in the targets |
| **GuardDuty** | Designate the same security delegated administrator in every enabled Region; auto-enable GuardDuty and selected protection plans for existing and new accounts | GuardDuty administration and detectors are Regional. Missing a Region or leaving auto-enable at `NEW` can leave existing accounts uncovered |
| **EventBridge** | Put a custom event bus in the operations/security account; allow senders with an `aws:PrincipalOrgID` condition; deploy member-account rules that target the central bus using an IAM role | The receiver's event-bus resource policy and the sender role must both allow delivery. Add DLQs/retries and filter at source to control cost and noise |

Build these as one onboarding path: Account Factory creates the account in the right OU;
controls apply immediately; StackSets deploy regional resources; trusted services enroll it;
logs go to Log Archive; findings and events go to the security/operations account. Test new
account onboarding in every governed Region—"centralized" does not mean "global by default."

---

## Key Exam Points

- **Organizations**: management + member accounts in **OUs**; **consolidated billing** gives one
  invoice and **shared volume/RI/Savings Plan discounts**.
- **SCPs and RCPs are guardrails**—they cap, never grant. SCPs cap member-account principals;
  RCPs cap supported member-account resources. SCPs don't restrict the management account; an
  SCP deny beats `AdministratorAccess`.
- **Tag Policies** standardize tag keys/values; separate from SCPs; enforcement mode blocks
  non-compliant API calls.
- **STS** issues temporary credentials; `AssumeRole` (cross-account), `AssumeRoleWithSAML`
  (SAML), `AssumeRoleWithWebIdentity` (OIDC/Cognito).
- **Cross-account** = role in B trusts A + identity in A allowed to `AssumeRole`. Use
  **`ExternalId`** for third parties.
- **Federation maps external users to roles**, never IAM users. **IAM Identity Center**
  (ex-AWS SSO) is the modern default for workforce SSO across an Org.
- **Service roles** vs **service-linked roles** for AWS services accessing resources.
- **Control Tower** adds a landing-zone baseline, Account Factory, controls, and drift
  monitoring on top of Organizations. Preventive controls block, proactive controls check
  CloudFormation before provisioning, and detective controls report after deployment.
- Use **trusted access** so an integrated service can operate across the organization; use a
  **delegated administrator** so a member account can run that service without daily management
  account access.
- StackSets deploy repeatable cross-account/Region baselines. CloudTrail, Config, Security Hub
  CSPM, GuardDuty, and EventBridge each require explicit account and Region coverage.

---

## Common Mistakes

- ⚠️ Thinking an SCP *grants* permissions — it only restricts. You still need an IAM grant.
- ⚠️ Applying an SCP to the bucket owner's account and expecting it to restrict an external
  caller. Use a supported RCP as the Organizations resource-side guardrail.
- ⚠️ Expecting SCPs to limit the **management account** — they don't.
- ⚠️ Confusing **Tag Policies** with SCPs — Tag Policies enforce tagging format, not IAM actions.
- ⚠️ Creating IAM users for each employee instead of using **Identity Center / federation**.
- ⚠️ Setting up only one side of cross-account access (forgetting either the trust policy or the
  caller's `sts:AssumeRole` permission).
- ⚠️ Omitting `ExternalId` when granting a third party `AssumeRole` (confused-deputy risk).
- ⚠️ Thinking Control Tower replaces Organizations — it is a governance layer on top of it.
- ⚠️ Assuming a Config aggregator enables Config recording, or that a Regional delegated
  administrator automatically covers every Region.
- ⚠️ Editing Control Tower-managed SCPs, roles, or StackSets directly and creating landing-zone
  drift.

---

**Next**: [05_directory_services.md — AWS Directory Services](05_directory_services.md)
