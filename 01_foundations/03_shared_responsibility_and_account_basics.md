# Shared Responsibility & Account Basics

> **Who this is for**: Engineers who know the cloud model
> ([01_cloud_computing_fundamentals.md](01_cloud_computing_fundamentals.md)) and AWS infrastructure
> ([02_aws_global_infrastructure.md](02_aws_global_infrastructure.md)), and now need the security
> contract and the account scaffolding before diving into identity. This is the bridge into
> [IAM & Identity](../02_iam/README.md).

---

## 1️⃣ The Shared Responsibility Model

Security on AWS is a **partnership**. AWS secures the infrastructure; you secure what you put on
it. The dividing line is captured in one phrase:

> **AWS is responsible for security _OF_ the cloud. You are responsible for security _IN_ the cloud.**

```
        ┌───────────────────────────────────────────────┐
  YOU   │   Customer data                               │
 secure │   Platform, applications, identity (IAM)      │  ← security
   IN   │   OS, network & firewall config (SG/NACL)     │     IN the
  the   │   Client- & server-side encryption, data      │     cloud
 cloud  │   integrity, network traffic protection       │
        ╞═══════════════════════════════════════════════╡  ← the line
  AWS   │   Compute, storage, database, networking      │  ← security
 secures│   (the software AWS runs)                     │     OF the
   OF    │   Regions, AZs, edge locations (hardware)    │     cloud
  the   │   Physical security of data centers           │
 cloud  └───────────────────────────────────────────────┘
```

| Layer | Who owns it | Examples |
|-------|------------|----------|
| Physical data centers, hardware | **AWS** | Buildings, racks, power, hardware disposal |
| Global infrastructure (Regions/AZs/edge) | **AWS** | Network backbone, hypervisor |
| Managed service software | **AWS** | The S3/DynamoDB/RDS engine itself |
| Patching the **guest OS** on EC2 | **You** | OS updates, antivirus on your instances |
| **Network & firewall** config | **You** | Security Groups, NACLs, VPC design |
| **IAM** — users, roles, policies | **You** | Who can do what |
| **Customer data** & its classification | **You** | What you store and how you protect it |
| **Encryption** choices | **You** | Enabling encryption at rest/in transit, key management |

⚠️ The split **shifts with the service model**. For **EC2 (IaaS)** you patch the OS. For a managed
service like **RDS or Lambda (PaaS)**, AWS patches the OS/runtime — but you still own *your data,
access control (IAM), and encryption settings*. **The customer always owns their data and who can
access it.**

> **Key insight**: No matter the service, **IAM and customer data are always your responsibility.**
> AWS will never secure your data *from your own misconfigured permissions* (e.g., a public S3 bucket).

---

## 2️⃣ Root Account vs IAM

Every AWS account is created with a single **root user** — the email address you signed up with.
The root user has **unrestricted access to everything**, including actions nothing else can do
(closing the account, changing the support plan, billing settings).

| | **Root user** | **IAM user / role** |
|---|---|---|
| Created | Automatically, at sign-up | By you, as needed |
| Power | Unlimited; some actions root-only | Scoped by IAM policies (least privilege) |
| Daily use | **Never** | Yes — this is what people and apps use |
| Best practice | Lock away; enable **MFA**; create an admin IAM user/role instead | One identity per person/workload |

✅ **Correct**: secure the root user with **MFA**, delete its access keys, and use IAM
users/roles (ideally via IAM Identity Center) for everyday work.

❌ **Anti-pattern**: using the root account for daily administration or embedding root access keys
in code. A leaked root credential compromises the entire account.

> Identity mechanics (users, groups, roles, policy evaluation, federation) are the focus of the
> next section: [IAM & Identity](../02_iam/README.md).

---

## 3️⃣ Account & Billing Basics

The **AWS account** is the fundamental container and the **boundary for billing and most
resource isolation** — everything you create belongs to an account, and you get one bill per account.

Key billing tools to recognize:

- **AWS Billing Dashboard / Cost Explorer** — visualize and analyze spend over time.
- **AWS Budgets** — set a spending or usage threshold and get alerted (or take action) when approaching it.
- **Cost Allocation Tags** — tag resources (e.g., `Project=alpha`) to break the bill down by team/project.
- **Free Tier usage alerts** — warn you before you exceed free-tier limits.

💡 A very common first step in any account: set a **Budget alert** so a runaway resource doesn't
produce a surprise bill.

---

## 4️⃣ AWS Organizations and the Multi-Account Bridge

As you grow beyond one account, **AWS Organizations** lets you centrally manage **multiple
accounts** as a single tree.

```
         ┌──────────────────────┐
         │   Management account │  (the org owner / payer)
         └──────────┬───────────┘
                    │
        ┌───────────┼────────────┐
        ▼           ▼            ▼
   [ OU: Prod ] [ OU: Dev ]  [ OU: Sandbox ]
        │           │            │
     accounts    accounts     accounts
```

Two headline benefits:

- **Consolidated billing** — one bill for all accounts, plus **volume/aggregated discounts** across them.
- **Service Control Policies (SCPs)** — organization-wide **guardrails** that set the *maximum*
  permissions an account can have (e.g., "no one in Dev can use regions outside the EU").

> **Rule**: SCPs **limit** what IAM can grant — they don't grant anything themselves. An action is
> only allowed if **both** the SCP and an IAM policy permit it. Details in
> [IAM & Identity](../02_iam/README.md).

This simple tree is only the starting point. At SAP-C02 scale, the goal is a repeatable account
environment that separates workloads from governance functions and applies controls by policy.
The main building blocks are:

| Building block | Why it exists | Practical use |
|----------------|---------------|---------------|
| **Organizational units (OUs)** | Group accounts that need the same controls | Build OUs around policy and risk boundaries such as production, infrastructure, and sandbox. Do not copy the company org chart unless those boundaries genuinely match. |
| **Control Tower landing zone** | Establish a governed multi-account baseline using AWS Organizations and related services | Start new environments with identity integration, organization-wide controls, and shared **Log Archive** and **Audit/security** accounts instead of assembling the baseline manually in every account. |
| **Delegated administration** | Let a trusted member account administer a supported AWS service across the organization | Run security and operational services from a dedicated account. Keep day-to-day administration out of the management account, which should be reserved for organization-level tasks that require it. |
| **Centralized logging and security accounts** | Separate evidence and security operations from the workloads they monitor | Send organization trails and configuration/security findings to tightly controlled accounts so a compromised workload administrator cannot quietly alter the central record. |
| **Account vending** | Create accounts from an approved, automated template | Use Control Tower **Account Factory** or an infrastructure-as-code workflow to place a new account in the correct OU and apply its identity, logging, network, tagging, and guardrail baseline consistently. |

> **Design principle**: Use accounts as isolation boundaries, OUs as policy-application
> boundaries, and delegated services as the operating model. Avoid running production workloads
> in the management, logging, or security accounts.

The dedicated SAP-C02 treatment is
[Organizations, STS & Federation](../02_iam/04_organizations_sts_federation.md). It develops the
OU and SCP model, cross-account role access, IAM Identity Center, tag policies, and the Control
Tower landing zone and Account Factory mechanics. This foundations chapter supplies the account
layout mental model; use the dedicated chapter when designing or troubleshooting multi-account
governance.

---

## 5️⃣ Support Plans

AWS sells four support tiers. You don't need pricing, but you must match a plan to a scenario.

| Plan | Cost | Who answers / response | Trusted Advisor | Key differentiator |
|------|------|------------------------|-----------------|--------------------|
| **Basic** | Free | No technical support (docs, forums only) | **Core checks only** | Comes with every account |
| **Developer** | $ | **Cloud Support Associates**, business-hours email | Core checks | For experimenting / non-prod |
| **Business** | $$ | **24×7** phone/chat/email; <1hr for production-down | **Full set** of checks | First tier with full Trusted Advisor + 24×7 |
| **Enterprise** | $$$$ | 24×7; **<15-min** response for business-critical | Full set | **Technical Account Manager (TAM)**, Concierge, IEM |

> **Rule**: Need a dedicated **TAM** and 15-minute critical response → **Enterprise**. Need 24×7
> production support and **full Trusted Advisor** → **Business**. Non-production tinkering →
> **Developer**.

---

## 6️⃣ Account Health & Governance Tools

A handful of always-relevant services help you operate the account safely:

| Tool | What it does | Exam trigger |
|------|-------------|--------------|
| **Trusted Advisor** | Inspects your account and recommends fixes across **cost, performance, security, fault tolerance, service limits** | "automated best-practice recommendations" / "check for open security groups or unused resources" |
| **AWS Health Dashboard** | Shows the status of AWS services **and** events affecting *your* specific resources | "is AWS having an outage?" / "personalized event notifications" |
| **Service Quotas** | View and **request increases** to per-account/per-Region limits (e.g., max EC2 instances, VPCs) | "increase the limit on..." / "hit a soft limit" |

⚠️ **Trusted Advisor full checks require Business or Enterprise support.** Basic/Developer get only
the core checks. This pairing (support tier ↔ Trusted Advisor depth) is a classic exam combo.

💡 Most AWS limits are **soft limits** raised via **Service Quotas**; a few are hard limits. If a
scenario says "we keep hitting a ceiling on resource X," the answer is usually *request a quota increase*.

---

## Key Exam Points

- **Security OF the cloud = AWS; security IN the cloud = you.** The line shifts toward AWS for
  managed services, but **you always own your data and IAM**.
- For **EC2 you patch the OS**; for **RDS/Lambda AWS patches it** — you still own data, access, and encryption config.
- **Never use the root user** day-to-day; protect it with **MFA**, remove its keys, use IAM identities.
- **AWS Organizations** → consolidated billing + **SCP guardrails** (SCPs cap, never grant,
  permissions). At scale, organize accounts into policy-based **OUs** and separate management,
  logging, security, and workload duties.
- **Control Tower** establishes a governed landing zone; **Account Factory** vends standardized
  accounts. **Delegated administration** moves supported organization-wide service operations
  out of the management account.
- Support: **Enterprise** = TAM + 15-min critical; **Business** = 24×7 + full Trusted Advisor; **Developer** = non-prod.
- **Trusted Advisor** = best-practice recommendations (full checks need Business/Enterprise);
  **Health Dashboard** = service/account status; **Service Quotas** = view/raise limits.

---

## Common Mistakes

- ❌ Assuming AWS secures your data from your own bad permissions — a **public S3 bucket is your fault**, not AWS's.
- ❌ Thinking a managed service means "AWS handles all security." You still own **IAM, data, and encryption choices**.
- ❌ Using the **root account** for routine work or storing its access keys in an app.
- ❌ Believing an **SCP grants** permissions. SCPs only **restrict** the maximum; an IAM policy must still allow the action.
- ❌ Treating the **management account** as a shared administration or workload account. Delegate
  supported services and keep workloads in member accounts.
- ❌ Creating accounts manually without a baseline for OU placement, identity, logging, security,
  and guardrails. Use an account-vending workflow.
- ❌ Expecting **full Trusted Advisor checks** on Basic/Developer support — those need **Business or Enterprise**.
- ❌ Confusing the **Health Dashboard** (status/events) with **Trusted Advisor** (recommendations).

---

**Next**: [../02_iam/01_authn_vs_authz.md — Authentication vs Authorization](../02_iam/01_authn_vs_authz.md)
