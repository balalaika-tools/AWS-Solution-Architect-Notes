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
| Applies to | All IAM identities in the account(s)… | …the specific identity |
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

Returned creds = **AccessKeyId + SecretAccessKey + SessionToken**, expiring in 15 min–12 h
(default ~1 h). They carry the **role's** permissions, not the caller's.

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

**Control Tower** is the "governed multi-account environment in a few clicks" service. It
orchestrates Organizations + IAM Identity Center + CloudFormation + Config + CloudTrail behind
one console.

```
   Control Tower
   ├── Landing Zone     — baseline account structure (Management, Log Archive, Audit)
   ├── Account Factory  — template-based account vending; new accounts arrive pre-configured
   └── Guardrails
       ├── Preventive   — SCPs (mandatory or optional; can be org-wide or OU-scoped)
       └── Detective    — AWS Config rules (monitor and flag drift)
```

| Component | What it does |
|-----------|-------------|
| **Landing Zone** | Bootstraps three baseline accounts: Management, Log Archive (centralised CloudTrail/Config logs), Audit (read-only security access) |
| **Account Factory** | Service Catalog-backed form that provisions new accounts with your baseline — VPC layout, guardrails, IAM Identity Center assignments |
| **Preventive guardrails** | SCPs that block non-compliant actions (e.g. "Disallow public S3 buckets") |
| **Detective guardrails** | Config rules that flag drift (e.g. "Detect MFA not enabled for root") |

> **Key insight**: Control Tower does not replace Organizations — it *uses* Organizations. The
> guardrails are SCPs and Config rules that Control Tower manages for you. Direct Organizations
> console access still works; Control Tower is a governance layer on top.

⚠️ Exam pattern: "A company wants to enforce security baselines for every new AWS account
automatically" → **Control Tower + Account Factory**, not manually-applied SCPs per account.

---

## Key Exam Points

- **Organizations**: management + member accounts in **OUs**; **consolidated billing** gives one
  invoice and **shared volume/RI/Savings Plan discounts**.
- **SCPs are guardrails** — they **cap**, never grant. Effective access = **SCP ∩ IAM policy**.
  SCPs don't restrict the **management account**; an SCP deny beats `AdministratorAccess`.
- **Tag Policies** standardize tag keys/values; separate from SCPs; enforcement mode blocks
  non-compliant API calls.
- **STS** issues temporary credentials; `AssumeRole` (cross-account), `AssumeRoleWithSAML`
  (SAML), `AssumeRoleWithWebIdentity` (OIDC/Cognito).
- **Cross-account** = role in B trusts A + identity in A allowed to `AssumeRole`. Use
  **`ExternalId`** for third parties.
- **Federation maps external users to roles**, never IAM users. **IAM Identity Center**
  (ex-AWS SSO) is the modern default for workforce SSO across an Org.
- **Service roles** vs **service-linked roles** for AWS services accessing resources.
- **Control Tower** = Organizations + IAM Identity Center + Config + CloudTrail wired up
  automatically; guardrails are SCPs (preventive) and Config rules (detective).

---

## Common Mistakes

- ⚠️ Thinking an SCP *grants* permissions — it only restricts. You still need an IAM grant.
- ⚠️ Expecting SCPs to limit the **management account** — they don't.
- ⚠️ Confusing **Tag Policies** with SCPs — Tag Policies enforce tagging format, not IAM actions.
- ⚠️ Creating IAM users for each employee instead of using **Identity Center / federation**.
- ⚠️ Setting up only one side of cross-account access (forgetting either the trust policy or the
  caller's `sts:AssumeRole` permission).
- ⚠️ Omitting `ExternalId` when granting a third party `AssumeRole` (confused-deputy risk).
- ⚠️ Thinking Control Tower replaces Organizations — it is a governance layer on top of it.

---

**Next**: [05_directory_services.md — AWS Directory Services](05_directory_services.md)
