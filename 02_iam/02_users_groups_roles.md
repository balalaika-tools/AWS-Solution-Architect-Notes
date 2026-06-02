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

## Key Exam Points

- **User** = long-lived identity for a person/app, with a password and/or **up to two access
  keys**. **Group** = policy container, *not* a principal. **Role** = assumed identity with
  **temporary STS credentials and no long-lived keys**.
- Roles have a **trust policy** (who can assume) + **permissions policies** (what they can do).
- **EC2/Lambda/cross-account → use a role.** Never embed access keys in an instance, AMI, or
  code.
- **Instance profile** is the wrapper that attaches a role to an EC2 instance; it holds exactly
  one role. Credentials arrive via **IMDS** (prefer **IMDSv2**).
- Enforce **MFA** on human users; prefer eliminating access keys via roles and Identity Center.

---

## Common Mistakes

- ⚠️ Storing access keys on an EC2 instance instead of attaching an instance role.
- ⚠️ Thinking you "log in as a group" — groups carry no credentials.
- ⚠️ Forgetting a role needs a **trust policy**; a correct permissions policy alone won't let
  anyone assume it.
- ⚠️ Trying to put a role into a group, or nesting groups — neither is allowed.
- ⚠️ Treating role credentials as permanent — they expire; the SDK refreshes them via IMDS/STS.

---

**Next**: [03_iam_policies.md — IAM Policies: Structure & Evaluation](03_iam_policies.md)
