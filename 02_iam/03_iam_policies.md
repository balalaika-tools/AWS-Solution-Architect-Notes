# IAM Policies: Structure & Evaluation

> **Who this is for**: Engineers who know the IAM identities and now need to read, write, and
> reason about policy JSON for the exam. Read
> [02_users_groups_roles.md](02_users_groups_roles.md) first.

---

## 1. Policy JSON Structure

A policy is a JSON document made of one or more **statements**. Each statement is an allow/deny
rule. Memorize these elements — questions hinge on them.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadReportsBucket",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::acme-financial-reports",
        "arn:aws:s3:::acme-financial-reports/*"
      ],
      "Condition": {
        "IpAddress": { "aws:SourceIp": "203.0.113.0/24" }
      }
    }
  ]
}
```

| Element | Required? | Purpose |
|---------|-----------|---------|
| `Version` | Yes | Policy language version — **always `2012-10-17`** (not the date you wrote it) |
| `Statement` | Yes | One rule, or an array of rules |
| `Sid` | Optional | Human-readable statement label |
| `Effect` | Yes | `Allow` or `Deny` |
| `Action` | Yes* | API actions, e.g. `s3:GetObject` (`*` = all; `s3:Get*` = prefix wildcard) |
| `Resource` | Yes* | ARNs the statement applies to |
| `Condition` | Optional | Extra constraints (IP, MFA, tags, time, encryption…) |
| `Principal` | **Resource-based only** | *Who* the statement applies to — used in trust/bucket policies, **not** identity-based policies |

\* Identity-based policies use `Action`/`Resource`. Resource-based and trust policies add
`Principal` and may use `NotAction`/`NotResource`.

> **Key insight**: `Principal` is the tell. If a policy has a `Principal`, it's
> **resource-based** (it's attached to a resource and names who may use it). Identity-based
> policies omit `Principal` — the identity they're attached to *is* the principal.

---

## 2. The Four Policy Types (and How They Interact)

| Type | Attached to | Grants or limits? | Example use |
|------|-------------|-------------------|-------------|
| **Identity-based** | User / group / role | **Grants** permissions to that identity | "Devs can read this bucket" |
| **Resource-based** | A resource (S3 bucket, SQS, KMS, Lambda…) | **Grants** access *to that resource*, names a `Principal` | Allow another account to read a bucket |
| **Permissions boundary** | A user or role | **Limits** the *maximum* permissions an identity-based policy can grant | Cap what a delegated admin can hand out |
| **SCP** (Service Control Policy) | An Org account / OU | **Limits** the maximum for *everyone in the account* (except root in some cases) | "No one may disable CloudTrail" |

> **Rule**: Boundaries and SCPs are **guardrails — they never grant access**. An action is only
> allowed if it's both *granted* by some policy **and** *permitted* by every applicable
> guardrail. SCPs are covered in [04_organizations_sts_federation.md](04_organizations_sts_federation.md).

```
   Effective permissions = (granting policies)  ∩  (permissions boundary)  ∩  (SCPs)
                            ─────grant─────         ────────cap────────────────
   ...and ANY explicit Deny anywhere wins.
```

---

## 3. Policy Evaluation Logic

For a single account, AWS evaluates **all** applicable policies and reduces them to one decision:

```
          Request from a principal
                    │
                    ▼
   ┌──────────────────────────────────────┐
   │ 1. Is there an explicit DENY?        │── YES ──► ❌ DENY  (always wins)
   └──────────────────────────────────────┘
                    │ no
                    ▼
   ┌──────────────────────────────────────┐
   │ 2. Is the action ALLOWED by a policy │── NO  ──► ❌ DENY  (implicit deny)
   │    AND permitted by boundary/SCP?    │
   └──────────────────────────────────────┘
                    │ yes
                    ▼
                ✅ ALLOW
```

> **Rule (memorize)**: **Explicit `Deny` > explicit `Allow` > implicit (default) deny.**
> Everything starts denied. One matching `Allow` opens it. Any matching `Deny` slams it shut and
> cannot be overridden.

⚠️ Because explicit deny always wins, a broad `Deny` (e.g. "deny all S3 outside our VPC
endpoint") in *any* attached policy will block the action even if ten other policies allow it.

---

## 4. Managed vs Inline Policies

| | **Managed policy** | **Inline policy** |
|---|--------------------|-------------------|
| Definition | Standalone object with its own ARN | Embedded directly in one identity |
| Reuse | Attach to many users/roles/groups | 1:1 with its identity — dies with it |
| Versioning | Yes (up to 5 versions, rollback) | No |
| Kinds | **AWS-managed** (maintained by AWS) and **customer-managed** | n/a |
| Best for | Almost everything | Tight 1:1 coupling you never want to leak elsewhere |

✅ Prefer **customer-managed** policies for reusable, least-privilege grants. AWS-managed
policies (e.g. `AdministratorAccess`, `ReadOnlyAccess`) are convenient but often broader than you
need. 💡 Inline policies are useful when you want the permission to be inseparable from one
specific role.

---

## 5. Realistic Examples

### 5.1 Cross-account S3 read (bucket policy — resource-based)

Attached to bucket `acme-shared-data`. Grants account `222222222222` read access *to this
bucket*:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPartnerAccountRead",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::222222222222:root" },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::acme-shared-data",
        "arn:aws:s3:::acme-shared-data/*"
      ]
    }
  ]
}
```

⚠️ This is only *half* of cross-account access — the IAM identity in account `222222222222`
also needs an identity-based policy allowing the same S3 actions.

### 5.2 EC2 control gated by a Condition (identity-based)

Lets an operator start/stop only instances tagged `team = data-eng`, and **only when signed in
with MFA**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ManageDataEngInstancesWithMFA",
      "Effect": "Allow",
      "Action": ["ec2:StartInstances", "ec2:StopInstances"],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": { "aws:ResourceTag/team": "data-eng" },
        "Bool": { "aws:MultiFactorAuthPresent": "true" }
      }
    }
  ]
}
```

### 5.3 AssumeRole trust policy (resource-based, on a role)

Lets a Lambda function assume this role and adds a **confused-deputy guard** with
`sts:ExternalId` for third-party access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowLambdaToAssume",
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    },
    {
      "Sid": "AllowVendorWithExternalId",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::333333333333:root" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": { "sts:ExternalId": "acme-7f3c9a2b" }
      }
    }
  ]
}
```

---

## 6. Common Mistakes

- ❌ **Overly broad wildcards** — `"Action": "*", "Resource": "*"`. Grants account-wide admin.
  Scope to specific actions and ARNs.
- ❌ **Wrong `Version`** — using today's date instead of the literal `"2012-10-17"`.
- ❌ **Putting `Principal` in an identity-based policy** — it belongs only in resource-based /
  trust policies.
- ❌ **Forgetting the second half of cross-account access** — both the resource policy *and* the
  caller's identity policy must allow it.
- ⚠️ **The confused deputy problem** — a trusted third party is tricked into using *its*
  permissions on your behalf. Mitigate with `sts:ExternalId` (third parties) or
  `aws:SourceArn` / `aws:SourceAccount` conditions (AWS service principals).
- ⚠️ **`s3:ListBucket` on the wrong ARN** — list is a *bucket* action (`arn:...:::bucket`); get
  is an *object* action (`arn:...:::bucket/*`). Mixing them up breaks listing.

---

## Key Exam Points

- Policy elements: `Version` (always `2012-10-17`), `Statement`, `Effect`, `Action`,
  `Resource`, `Condition`; `Principal` **only** in resource-based/trust policies.
- **Four policy types**: identity-based and resource-based **grant**; **permissions boundaries**
  and **SCPs** only **cap** (never grant).
- **Evaluation: explicit Deny > Allow > implicit deny.** Default is deny; any matching deny
  wins.
- **Managed** (AWS- or customer-) policies are reusable and versioned; **inline** policies are
  1:1 with an identity.
- Cross-account access usually needs **both** the resource policy and the caller's identity
  policy.
- Guard against the **confused deputy** with `ExternalId` / `aws:SourceArn`.

---

**Next**: [04_organizations_sts_federation.md — Organizations, STS & Federation](04_organizations_sts_federation.md)
