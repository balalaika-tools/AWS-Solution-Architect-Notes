# IAM Policies: Structure & Evaluation

> **Who this is for**: Engineers who know the IAM identities and now need to read, write, and
> reason about policy JSON for the exam. Read
> [02_users_groups_roles.md](02_users_groups_roles.md) first.

---

## 0. The Mental Model: Why Two Policy Types Exist

Before the JSON, get the intuition — it makes everything below obvious.

### 0.1 IAM only ever asks one question

Every authorization decision in AWS is the answer to **one sentence**:

> Is **principal P** allowed to do **action A** on **resource R**?

- **Principal** = the *who* (a user, role, or AWS service) — the actor.
- **Action** = the *verb* (`s3:GetObject`, `sqs:SendMessage`).
- **Resource** = the *what* (a specific bucket, queue, key).

A policy is just a **written rule that helps AWS answer that sentence**. The two policy types
are **not two different powers** — they are the *same kind of rule written in two different
places*, bolted onto two different nouns in that sentence.

```
   Principal P  ───────  wants action A  ───────▶  Resource R
   (the "who")                                     (the "what")
        │                                               │
        │ attach the rule HERE            attach the rule HERE
        ▼                                               ▼
   IDENTITY-BASED POLICY                   RESOURCE-BASED POLICY
   "I can do A on R"                        "P is allowed to do A on me"
```

That is the entire distinction: **which object holds the rule.** This is *why* `Principal` is
the tell (§1) — a resource-based policy must name *who* it admits; an identity-based policy is
already attached to its principal, so naming one would be redundant.

### 0.2 Two perspectives on the same fact

| | **Identity-based** | **Resource-based** |
|---|--------------------|--------------------|
| Attached to | The principal (user/group/role) | The resource (S3, SQS, KMS, Lambda…) |
| Point of view | *"Here's everything **I** can do."* | *"Here's **who** may touch **me**."* |
| Names a `Principal`? | No (it *is* the principal) | **Yes** (must say who) |
| Names a `Resource`? | Yes | Yes (itself) |

> **One-line mental model**
> **Identity-based** = a **badge** you carry — it lists the doors *you* may open.
> **Resource-based** = a **guest list** on the door — it lists who *the door* will admit.
> Same access, described from opposite ends.

### 0.3 Why AWS needs both (the real motivation)

If one rule answers the question, why two places to write it? Because each perspective solves a
problem the other can't:

1. **Cross-account access** — the killer reason. An identity-based policy in *your* account
   can't grant access into *someone else's* account (you don't own their identities). The
   resource owner uses a **resource-based** policy to say "Account B's role may read me"
   directly — no mirror identity needed. *(See the full two-sided rule in §0.4 and the example
   in §5.1.)*
2. **"Many principals, one resource" vs "one principal, many resources."** Write the rule
   wherever there's *one* of something and *many* of the other, so you write it **once**:
   - One role that touches 50 services → put it on the **role** (identity-based).
   - 50 principals that read one bucket → put it on the **bucket** (resource-based).
3. **Some grants have nowhere else to live.** A public S3 bucket, or an SNS topic an AWS
   service publishes to — the "principal" is *everyone* or *a service*. You can't edit a policy
   on "everyone," so the rule **must** live on the resource.

### 0.4 How the two combine

This is the part exam questions hide in. The full deny-wins evaluation flow is in §3; the
*combination* rule across the two types is:

```
   SAME account:    Allow in EITHER the identity policy OR the resource policy  →  ALLOW
                    (they are ADDITIVE — you only need one Allow)

   CROSS account:   Allow in BOTH the resource policy (target side)
                    AND the caller's identity policy (caller side)             →  ALLOW
                    (two doors — the resource owner unlocks theirs, but your
                     own account must also let you walk out)

   ALWAYS:          an explicit Deny in ANY policy overrides every Allow.
```

⚠️ The cross-account "both sides" rule is the single most-tested idea here — a bucket policy
alone does **not** grant a foreign principal access if that principal's own identity policy
stays silent. *(Trap restated in §5.1 and §8.)*

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

## 7. Advanced Policy Elements

### 7.1 `NotAction`, `NotResource`, `NotPrincipal`

These invert the scope of a statement. They are useful but dangerous — read them as "everything
*except*."

| Element | Meaning | Typical use |
|---------|---------|-------------|
| `NotAction` | Match ALL actions **except** listed | "Allow everything except IAM and billing" |
| `NotResource` | Match ALL resources **except** listed | Rarely used; prefer scoped `Resource` ARNs |
| `NotPrincipal` | Match ALL principals **except** listed | Deny statements only — blocks everyone but the named principal |

**`NotAction` — allow all non-IAM actions** (common pattern for a developer role that must not
self-escalate):

```json
{
  "Effect": "Allow",
  "NotAction": ["iam:*", "organizations:*", "account:*"],
  "Resource": "*"
}
```

⚠️ `NotPrincipal` in an `Allow` would grant access to every principal in the universe except the
listed one — almost never the intent. Keep `NotPrincipal` in `Deny` statements only (e.g.,
"deny everyone except the break-glass role from deleting this bucket").

### 7.2 Policy Variables

Policy variables embed context values into `Resource` ARNs and `Condition` values at evaluation
time — one policy can serve many users.

| Variable | Resolves to |
|----------|------------|
| `${aws:username}` | The calling IAM user's name |
| `${aws:userid}` | Unique ID of the calling principal |
| `${aws:PrincipalTag/key}` | A tag on the calling principal |
| `${s3:prefix}` | The S3 prefix used in the request |

**Classic example** — every user may read/write only their own "home folder" in a shared bucket,
with one policy attached to a group:

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::corp-home/${aws:username}/*"
}
```

The variable resolves at request time. No per-user policies needed.

---

## 8. Resource-based Policies vs IAM Roles — When to Use Which

Both enable cross-account and service access, but the caller's identity is handled differently.

| | **Resource-based policy** | **IAM role (`AssumeRole`)** |
|---|--------------------------|----------------------------|
| Where it lives | On the resource (S3, SQS, KMS, Lambda…) | IAM — a separate identity |
| Caller's identity | **Preserved** — caller acts as themselves | **Replaced** — caller takes the role's identity |
| Requires `AssumeRole`? | No — direct API call to the resource | Yes — caller exchanges identity via STS |
| Works for EC2/Lambda callers? | Only if the resource supports it | Yes, via instance profile / execution role |
| Useful for chaining many services? | No — per-resource | Yes — one credential set across many services |

**Decision guide:**

```
Is the target S3/SQS/KMS/Lambda AND the caller is in another account?
  → A bucket/queue/key policy can grant access directly (no AssumeRole required from the
    caller's side, saving the second-account grant step).

Does the caller need to touch many services in one workflow?
  → Use a role — one credential set, scope enforced at the role's permissions policy.

Is the caller an EC2 instance or Lambda function?
  → Always use an IAM role (instance profile / execution role) — they cannot hold long-lived
    access keys.
```

⚠️ Classic exam trap: an **S3 bucket policy** can grant another account direct access without
`AssumeRole`. But if an EC2 instance in that other account is the actual caller, the instance
still needs an **instance role** with the `s3:PutObject` permission — the bucket policy says
"trust that account," but the instance must also have it.

---

## Key Exam Points

- **Mental model**: one question — *can principal P do action A on resource R?* The two policy
  types are the same rule on different nouns. **Identity-based = a badge** (doors *you* may
  open, no `Principal`); **resource-based = a guest list** (who *the resource* admits, names a
  `Principal`). Same account → *either* Allow suffices; cross-account → need *both* sides.
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
- **`NotAction`** = allow/deny everything *except* listed actions; keep **`NotPrincipal`** in
  `Deny` statements only.
- **Policy variables** (`${aws:username}` etc.) embed context into ARNs at eval time — one
  policy for many identities.
- **Resource-based policy** preserves the caller's identity; **AssumeRole** replaces it.
  Use a role when the caller is compute (EC2/Lambda) or needs multi-service access.

---

**Next**: [04_organizations_sts_federation.md — Organizations, STS & Federation](04_organizations_sts_federation.md)
