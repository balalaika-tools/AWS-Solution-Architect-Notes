# Users, Groups, Roles & Instance Profiles

> **Who this is for**: Engineers who understand authentication vs authorization and want to know
> *which IAM identity to use when*. Read [01_authn_vs_authz.md](01_authn_vs_authz.md) first.

---

## 1. The Three Identities

IAM gives you three kinds of identity to attach permissions to. Picking the right one is a
recurring exam decision.

| Identity | Represents | Credentials | Lifetime | Typical use |
|----------|-----------|-------------|----------|-------------|
| **User** | A specific person or a long-lived app | Password (console) and/or access keys (API) | **Long-lived** until rotated | A human who logs into the console; legacy on-prem app |
| **Group** | A *collection of users* | None — not a principal | n/a | Attach a policy once, apply to many users (e.g. `Developers`) |
| **Role** | An identity *assumed temporarily* by a trusted principal | **Temporary** tokens from STS | Minutes to hours | EC2/Lambda accessing AWS; cross-account access; federated/SSO logins |

> **Key insight**: A **group is not a principal**. You never "sign in as a group" — a group is
> just a container that makes attaching the same policy to many users easy. A user can belong to
> multiple groups; permissions are the union.

⚠️ Groups cannot be nested (no group-inside-a-group), and a role cannot be added to a group.

---

## 2. Users and Their Credentials

An **IAM user** has two independent credential types — give each only what it needs:

| Credential | Used for | Notes |
|------------|----------|-------|
| **Console password** | Web console sign-in | Subject to account password policy; pair with MFA |
| **Access keys** (Access Key ID + Secret Access Key) | CLI, SDK, API | **Long-lived secrets.** Up to 2 per user (to allow rotation). Must be rotated/guarded |

> **Rule**: A user can have a password, access keys, both, or neither — they are not linked.
> A service account that only calls the API needs access keys and no password.

**MFA (Multi-Factor Authentication)** adds a second factor (something you *have*) on top of the
password (something you *know*). Supported: virtual authenticator apps (TOTP), FIDO2/U2F
security keys, and hardware TOTP tokens. ✅ Enforce MFA on all human users — and require it for
sensitive actions via a policy `Condition` (see [03_iam_policies.md](03_iam_policies.md)).

⚠️ **Access keys are the most-leaked AWS credential** (committed to Git, baked into AMIs). The
exam's preferred answer is almost always to **avoid long-lived access keys** in favor of roles.

---

## 3. Roles: Temporary Credentials Done Right

A **role** is an identity with a permissions policy but **no long-lived credentials**. A trusted
principal *assumes* the role and receives **temporary security credentials** from **STS**
(Security Token Service): an access key, a secret key, and a **session token**, all of which
expire (default ~1 hour; configurable).

Every role has two policies:

1. A **trust policy** (a resource-based policy on the role) — *who* may assume it.
2. One or more **permissions policies** — *what* the assumed session can do.

```
   Principal (EC2 / Lambda / user / another account / IdP)
        │
        │  sts:AssumeRole (allowed by the role's TRUST policy)
        ▼
   ┌──────────────────────────────────────────┐
   │  STS issues TEMPORARY credentials        │
   │  (AccessKeyId + SecretKey + SessionToken)│
   │  → auto-expire, no rotation needed       │
   └──────────────────────────────────────────┘
        │
        │  request signed with temp creds
        ▼
   AWS evaluates the role's PERMISSIONS policies
```

> **Key insight**: Roles eliminate stored secrets. Because credentials are short-lived and
> issued on demand, a leaked token expires on its own and there's nothing long-lived to rotate.
> This is *the* reason roles are the preferred answer for almost any "how should X get
> permissions" question.

---

## 4. User vs Role — When to Use Which

| Use a **role** when... | Use a **user** when... |
|------------------------|------------------------|
| An **AWS service** (EC2, Lambda, ECS) needs permissions | A specific **human** signs into the console *without* SSO |
| **Cross-account** access is needed | A legacy/on-prem system genuinely can't assume a role |
| **Federated / SSO** users sign in (SAML, OIDC, Identity Center) | You need a long-lived service account *and* can't use a role |
| Anything where you'd otherwise store long-lived keys | (Increasingly rare — prefer roles/Identity Center) |

> **Rule of thumb for the exam**: If a question asks how an **EC2 instance, Lambda function, or
> another AWS account** should get permissions — the answer is a **role**, never embedded access
> keys.

---

## 5. Common Role Flavors

- **Service role** — a role an **AWS service** assumes to act on your behalf (e.g. a Lambda
  execution role, an EC2 instance role, a CodeBuild role). The trust policy names the service
  principal (e.g. `lambda.amazonaws.com`).
- **EC2 instance role** — lets code on an EC2 instance call AWS APIs without stored keys
  (see §6).
- **Cross-account role** — lives in account `B`, trusts account `A`; principals in `A` assume
  it to operate in `B`. Detailed in [04_organizations_sts_federation.md](04_organizations_sts_federation.md).
- **Role for federation / Identity Center** — assumed by users authenticated by an external IdP
  or by IAM Identity Center, mapping SSO sign-ins to AWS permissions.

---

## 6. Instance Profiles — How EC2 Actually Uses a Role

You cannot attach a role to an EC2 instance directly. An **instance profile** is the container
that holds a single role and is what actually gets attached to the instance. The console hides
this — when you "attach a role" to an instance, it transparently creates/uses an instance
profile of the same name. With the CLI/CloudFormation you may handle it explicitly.

```
   EC2 Instance
       │  attached to
       ▼
   Instance Profile  ── contains ──►  IAM Role  ── grants ──►  permissions
       │
       │  credentials delivered via the
       ▼  Instance Metadata Service (IMDS)
   http://169.254.169.254/latest/meta-data/iam/security-credentials/<role>
   → auto-rotated temporary credentials, picked up by the SDK/CLI automatically
```

💡 The SDK and CLI find these credentials automatically through the **default credential
provider chain** — your code needs **no access keys at all**. Use **IMDSv2** (token-based,
session-oriented) over IMDSv1 for defense against SSRF.

⚠️ An instance profile holds **exactly one role**. To change an instance's permissions, modify
that role's policies (no reboot needed) or swap the attached profile.

---

## 7. Enterprise Cross-Account Role Sessions

A role is not merely "temporary credentials." In a large organization, the session must also
answer four questions: how long it lasts, which tenant requested it, which human or workload
started the chain, and which attributes should travel with it.

### 7.1 Session duration and role chaining

Each role has a **maximum session duration** from **1 to 12 hours**; the default is 1 hour. An
`AssumeRole` caller requests `DurationSeconds`, and STS rejects a value above that role's
configured maximum.

**Role chaining** means using one role session to assume another role. The chained session is
capped at **1 hour**, even if the destination role allows 12 hours. This often appears in a
hub-and-spoke design:

```
Workforce SSO session
    └── assumes SecurityAuditHub
            └── assumes ReadOnlySpoke in a workload account  ← chained; at most 1 hour
```

If an operator needs an uninterrupted 4-hour session in `ReadOnlySpoke`, issue that role
directly from the IdP or Identity Center instead of hopping through `SecurityAuditHub`. Do not
solve session expiry by creating long-lived access keys; SDKs and credential processes should
refresh role credentials.

### 7.2 `ExternalId`: identify a vendor's customer

Use an `sts:ExternalId` condition when a **third-party vendor** assumes a role in your account.
The vendor should generate a unique value for each customer and pass it on `AssumeRole`. This
prevents another vendor customer from tricking the shared vendor principal
into assuming your role—the confused-deputy problem.

`ExternalId` is visible to principals that can inspect the trust policy, so it is **not a
password or secret**. It also does not identify the vendor employee who initiated the call.
Keep the trusted `Principal` narrow, require an external ID, and let the vendor rotate it with a
controlled trust-policy change.

### 7.3 `SourceIdentity`: preserve who started the session

`SourceIdentity` is a stable string such as an employee ID or workload ID that STS records in
the session and CloudTrail. The target role's trust policy must permit
`sts:SetSourceIdentity`; for cross-account role chaining, the caller's permissions policy must
permit it too. Once set, the value persists through role chaining and cannot be changed in
later sessions.

Use it for attribution: a CloudTrail event from `ReadOnlySpoke` can still show that employee
`E12345` began the chain. Validate the format in the trust policy and derive it from an
authenticated IdP claim; do not let users choose an arbitrary colleague's identifier.

### 7.4 Session tags: carry authorization attributes

Session tags are key/value attributes such as `Department=Finance`, `Environment=Prod`, or
`Ticket=INC-4821`. Policies read them through `aws:PrincipalTag/<key>`, which makes
attribute-based access control (ABAC) possible without one role per team.

- The trust policy must allow `sts:TagSession`; use `aws:TagKeys` and
  `aws:RequestTag/<key>` conditions to restrict which values callers may assert.
- Mark only required keys as **transitive** if they must survive role chaining. Non-transitive
  tags stop at the next role.
- A session tag overrides a role tag with the same key. An inherited transitive tag that
  collides with a tag in the next request can make `AssumeRole` fail.
- STS accepts up to 50 session tags. Session policies and tags also share a packed-size limit,
  so an apparently small plaintext request can still return `PackedPolicyTooLarge`.

### 7.5 Putting the controls together

For a vendor operating production resources across accounts, create a separate role in each
target account and require:

1. The vendor's tightly scoped AWS principal plus a customer-unique `ExternalId`.
2. A required `SourceIdentity` for per-operator or per-workload audit attribution.
3. Approved session-tag keys and values, with only necessary tags marked transitive.
4. A short maximum session duration; expect a 1-hour ceiling if the vendor chains from another
   role.
5. Least-privilege role permissions, an SCP guardrail, and—when delegated role administration
   is needed—a permissions boundary. None of the session attributes grants permission by
   itself.

This separates **tenant binding** (`ExternalId`), **actor attribution** (`SourceIdentity`),
**authorization context** (session tags), and **credential lifetime** (session duration).

---

## Key Exam Points

- **User** = long-lived identity for a person/app, with a password and/or **up to two access
  keys**. **Group** = policy container, *not* a principal. **Role** = assumed identity with
  **temporary STS credentials and no long-lived keys**.
- Roles have a **trust policy** (who can assume) + **permissions policies** (what they can do).
- **EC2/Lambda/cross-account → use a role.** Never embed access keys in an instance, AMI, or
  code.
- **Instance profile** is the wrapper that attaches a role to an EC2 instance; it holds exactly
  one role. Credentials arrive via **IMDS** (prefer **IMDSv2**).
- A role can allow 1–12-hour sessions, but a **chained role session is capped at 1 hour**.
  `ExternalId` protects third-party role assumption, `SourceIdentity` preserves attribution,
  and session tags carry constrained ABAC attributes.
- Enforce **MFA** on human users; prefer eliminating access keys via roles and Identity Center.

---

## Common Mistakes

- ⚠️ Storing access keys on an EC2 instance instead of attaching an instance role.
- ⚠️ Thinking you "log in as a group" — groups carry no credentials.
- ⚠️ Forgetting a role needs a **trust policy**; a correct permissions policy alone won't let
  anyone assume it.
- ⚠️ Setting a destination role to 12 hours and expecting a chained session to last longer than
  1 hour.
- ⚠️ Treating `ExternalId` as a secret or letting callers assert unvalidated source identities
  and session tags.
- ⚠️ Trying to put a role into a group, or nesting groups — neither is allowed.
- ⚠️ Treating role credentials as permanent — they expire; the SDK refreshes them via IMDS/STS.

---

**Next**: [03_iam_policies.md — IAM Policies: Structure & Evaluation](03_iam_policies.md)
