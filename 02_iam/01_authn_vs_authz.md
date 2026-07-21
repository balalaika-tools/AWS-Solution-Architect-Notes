# Authentication vs Authorization & the IAM Model

> **Who this is for**: Engineers new to cloud security preparing for SAA-C03. No prior IAM
> knowledge assumed. Before reading this, understand AWS account basics:
> **[Foundations](../01_foundations/README.md)**.

---

## 1. Two Different Questions

Access control answers two separate questions, and confusing them is the single most common
source of muddled security thinking. They happen in order: you must prove who you are *before*
the system can decide what you may do.

| Concern | Question | Example | If it fails |
|---------|----------|---------|-------------|
| **Authentication (AuthN)** | *Who are you?* | Username + password, an access key, an MFA token | You can't log in at all |
| **Authorization (AuthZ)** | *What are you allowed to do?* | "May this user delete this S3 bucket?" | You're logged in but get `AccessDenied` |

> **Key insight**: A request can pass authentication and still be denied by authorization.
> "I know who you are, but you're not allowed to do that" is a normal, healthy outcome — it's
> least privilege working as intended.

In AWS terms:
- **Authentication** is handled by **credentials** — a password (console), access keys (API/CLI),
  or temporary security tokens.
- **Authorization** is handled by **policies** — JSON documents that grant or deny specific
  actions on specific resources.

---

## 2. Principals and Identities

A **principal** is any entity that can make a request to AWS — a person, an application, or an
AWS service acting on your behalf. When a principal sends a request, AWS authenticates it, then
evaluates policies to authorize it.

```
                  ┌─────────────────────────────────────────┐
                  │              PRINCIPALS                 │
                  │ (anything that can make an AWS request) │
                  └─────────────────────────────────────────┘
                       │            │              │
              ┌────────┘     ┌──────┘        ┌─────┘
              ▼              ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────────┐
        │ IAM User │   │ IAM Role │   │  Root User   │
        │ (person/ │   │ (assumed │   │ (the account │
        │  app)    │   │ identity)│   │   owner)     │
        └──────────┘   └──────────┘   └──────────────┘
              │              │               │
              └──────────────┴───────────────┘
                             ▼
                   AWS authenticates the principal,
                   then evaluates policies (AuthZ)
```

An **identity** is the IAM object that *represents* a principal and that you attach permissions
to: **users**, **groups**, and **roles**. (Groups aren't principals — you can't "log in as a
group" — but they're a convenient identity for attaching policies. More in
[02_users_groups_roles.md](02_users_groups_roles.md).)

---

## 3. Identity-Based vs Resource-Based Permissions

There are two *directions* from which a permission can be granted. Knowing which is which is
heavily tested.

| Approach | Attached to | Answers | Example |
|----------|-------------|---------|---------|
| **Identity-based** | A user, group, or role | "What can *this identity* do?" | "Alice may read objects in `reports-bucket`." |
| **Resource-based** | The resource itself (e.g. an S3 bucket, SQS queue, KMS key) | "Who can do what *to this resource*?" | "Account `B` may read this bucket." |

```
  IDENTITY-BASED                         RESOURCE-BASED
  ┌──────────────┐                       ┌────────────────────┐
  │  IAM Role    │──policy says────────► │   S3 Bucket        │
  │  "WebApp"    │  "I can read this"    │  (bucket policy    │
  └──────────────┘                       │   says who may     │
                                         │   access ME)       │
                                         └────────────────────┘
```

> **Key insight**: For access *within one account*, you usually need only **one** of the two —
> an allow on either side is enough. For **cross-account** access you generally need **both**:
> the resource policy must trust the other account, *and* the calling identity in that account
> must be allowed to make the call. (Covered in [03_iam_policies.md](03_iam_policies.md) and
> [04_organizations_sts_federation.md](04_organizations_sts_federation.md).)

---

## 4. Least Privilege

> **Principle**: Grant only the permissions required to perform a task — no more — and widen
> them only when a real need appears.

Least privilege limits the blast radius of leaked credentials or buggy code. The practical
recipe AWS recommends:

1. Start with **no permissions** and add specific allows.
2. Prefer **specific actions and resources** over wildcards (`s3:GetObject` on one bucket, not
   `s3:*` on `*`).
3. Use the **IAM Access Analyzer** and CloudTrail / "last accessed" data to find and remove
   unused permissions over time.

❌ `"Action": "*", "Resource": "*"` — full admin. Almost never the right answer on the exam
unless the question explicitly describes a break-glass admin role.

✅ Scope each policy to the smallest set of actions and ARNs the workload actually uses.

---

## 5. How AWS IAM Embodies All This

**IAM** (Identity and Access Management) is the AWS service that ties the concepts together:

- **Authentication** via credentials (passwords, access keys, MFA, temporary tokens).
- **Authorization** via JSON policies evaluated on every request.
- **Principals/identities** modeled as users, groups, and roles.
- **Both permission directions** — identity-based policies and resource-based policies.

Two exam facts about IAM as a service:

- 💡 **IAM is global.** Users, groups, roles, and policies are *not* tied to a Region. You don't
  pick a Region when creating them, and they work across all Regions.
- 💡 **IAM itself is free.** You pay for the resources your identities use, never for IAM
  objects.

---

## 6. Worked Authorization Scenario: Five Different Policy Jobs

Suppose a data-engineering role in account `111111111111` reads
`s3://central-reports/quarter=Q3/*` in account `222222222222`. The engineer first assumes the
role with STS. A production design can involve all five policy layers below:

| Policy | Where it lives | Its job in this request | What it cannot do |
|--------|----------------|-------------------------|-------------------|
| **Identity policy** | On the data-engineering role in account `111111111111` | Grants `s3:GetObject` on the reports objects | Cannot make account `222222222222` trust the caller |
| **Resource policy** | On the bucket in account `222222222222` | Admits the foreign role to those objects | Cannot delegate permission inside the caller's account; cross-account direct access still needs the caller-side identity grant |
| **Permissions boundary** | On the data-engineering role | Caps what its identity policies may grant—for example, read-only S3 | Cannot grant `GetObject`; silence in the identity policy is still a deny |
| **Session policy** | Passed when the role session is created | Narrows this session to `quarter=Q3/*`, even if the role normally reads all quarters | Cannot add permissions missing from the role; it only intersects with them |
| **SCP** | On the caller account's OU | Caps the role session as an account-wide guardrail—for example, denies `s3:DeleteObject` | Cannot grant S3 access, and an SCP on the bucket owner's account does not govern a principal that belongs to another account |

For `GetObject` on a Q3 object to succeed, both accounts must agree: the role's identity policy
must allow the call, the bucket policy must admit the foreign role, and the boundary, session
policy, and caller-account SCP must all leave the action available. A matching explicit deny in
any applicable policy wins.

Changing one layer gives a predictable result:

- Remove the **bucket policy allow** → account `222222222222` never admitted the foreign caller.
- Remove the **identity policy allow** → account `111111111111` never delegated its side of the
  cross-account permission.
- Ask for Q4 while the **session policy** permits only Q3 → denied for this session; assuming a
  correctly authorized broader session may work.
- Ask for `s3:PutObject` while the **boundary** is read-only → no identity policy can elevate the
  role past the boundary.
- Ask for `s3:DeleteObject` while the **SCP** denies it → even an administrator policy cannot
  override the guardrail.

> **Exam method**: Identify the policy's job before reading its JSON. Identity and resource
> policies can grant. Boundaries, session policies, and SCPs only reduce the available
> permissions.

The detailed evaluation path, including role trust and explicit denies, is in
[03_iam_policies.md](03_iam_policies.md).

---

## 7. The Root User — Handle With Extreme Care

When you create an AWS account, the **root user** is the identity tied to the account's email
address. IAM policies cannot restrict it. An Organizations SCP **can** restrict the root user of
a **member account**, but SCPs do not affect the Organizations management account.

> **Rule**: Use the root user only for the handful of tasks that *require* it, then lock it away.

⚠️ Tasks that genuinely require root (memorize a few — they appear as distractors):
- Changing the account name, email, or root password
- Closing the AWS account
- Changing or cancelling your AWS Support plan
- Restoring IAM user permissions (if you lock yourself out)
- Certain billing settings and registering as a seller in the Marketplace

✅ Root user best practices (frequently tested):
1. **Enable MFA on the root user** — ideally a hardware MFA device.
2. **Do not create access keys for root.** Delete any that exist.
3. **Create an IAM admin user (or use IAM Identity Center)** for day-to-day administration.
4. Use a strong, unique password and a group/distribution email you control.

❌ Logging in as root every day, or sharing root credentials with a team, is an automatic
red flag in exam scenarios.

---

## Key Exam Points

- **AuthN = who you are** (credentials); **AuthZ = what you can do** (policies). They are
  evaluated in that order and can have different outcomes.
- A **principal** makes requests; **identities** (users, groups, roles) carry permissions.
- **Identity-based** policies attach to identities; **resource-based** policies attach to
  resources. Cross-account access usually needs both sides to agree.
- **IAM is global and free.** No Region selection; no charge for IAM objects.
- **Least privilege**: start with nothing, grant specific actions/resources, prune with Access
  Analyzer.
- **Root user** is not restricted by IAM policies. An SCP can restrict a member-account root,
  but not the Organizations management account. Enable root MFA, delete root access keys, and
  use an admin IAM identity instead.

---

## Common Mistakes

- ⚠️ Treating "logged in" as "authorized." Authentication success says nothing about whether a
  specific action is allowed.
- ⚠️ Believing an IAM policy restricts the **root** user, or that an SCP restricts the
  Organizations **management account**. SCPs do restrict root in member accounts.
- ⚠️ Assuming IAM is regional and re-creating users per Region. It's global.
- ⚠️ Reaching for `*:*` admin policies instead of scoping to the task.
- ⚠️ Granting cross-account access on only one side and expecting it to work.

---

**Next**: [02_users_groups_roles.md — Users, Groups, Roles & Instance Profiles](02_users_groups_roles.md)
