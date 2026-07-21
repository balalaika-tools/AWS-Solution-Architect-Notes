# Amazon Cognito — User Pools, Identity Pools & Federated Access

> **Who this is for**: Engineers adding sign-up/sign-in to an app and deciding how it reaches
> AWS resources. Read [02_api_gateway.md](02_api_gateway.md) (Cognito is a common authorizer)
> and skim [Organizations, STS & Federation](../02_iam/04_organizations_sts_federation.md).

---

## 1. Prerequisite — App Auth: OIDC, OAuth & JWT

Two words that get confused constantly:

- **Authentication (AuthN)** — *who are you?* Proving identity (username + password, MFA, Google login).
- **Authorization (AuthZ)** — *what may you do?* Granting access to resources.

The protocols Cognito speaks:

| Term | What it is |
|------|------------|
| **OAuth 2.0** | An **authorization** framework — delegating access via tokens (e.g. "let this app act on my behalf") |
| **OIDC** | OpenID Connect — an **authentication** layer *on top of* OAuth 2.0; adds identity (the **ID token**) |
| **JWT** | JSON Web Token — a signed, self-contained token (header.payload.signature) carrying claims like user ID, expiry, groups |

```
JWT = base64(header) . base64(payload/claims) . signature
      └─ the server verifies the signature with the issuer's public key; no DB lookup needed
```

> **Key insight**: After a user signs in, Cognito hands the app **JWTs**. The app sends a JWT
> with each request; API Gateway / your backend validates the signature and reads the claims to
> authorize the request — **stateless**, no session store needed.

---

## 2. The Two Cognitos — They Solve Different Problems

This is the single most important Cognito exam concept. They are **two separate services** with
confusingly similar names.

| | **User Pool** | **Identity Pool** (Federated Identities) |
|---|---------------|------------------------------------------|
| **Purpose** | A **user directory** — authentication | Vends **temporary AWS credentials** — authorization to AWS |
| **Answers** | "Who is this user?" (sign-up / sign-in) | "What AWS resources may this user touch?" |
| **Output** | **JWTs** (ID, access, refresh tokens) | **Temporary AWS credentials** (via STS) |
| **Use to** | Authenticate users; protect your app/API | Let users call AWS directly (S3, DynamoDB) with scoped IAM |
| **Backed by** | Cognito's own directory (or federated IdPs) | An IdP (a User Pool, Google, Facebook, SAML, OIDC) + IAM roles |
| **Talks to** | Your app, API Gateway, hosted UI | AWS STS → assigns an IAM role |

> **Rule**: **User Pool = identity / login (gives you a JWT). Identity Pool = AWS access (gives
> you temporary AWS credentials).** If the question is "sign users in," it's a User Pool. If it's
> "let signed-in users upload to S3 directly from the browser," you also need an Identity Pool.

---

## 3. Cognito User Pools — The User Directory

A **User Pool** is a managed directory that handles the whole authentication lifecycle.

- **Sign-up / sign-in** — username/email/phone, password policies, account confirmation.
- **JWT tokens** — on success the user gets an **ID token** (identity claims), **access token**
  (authorization scopes), and **refresh token** (to get new tokens without re-login).
- **Hosted UI** — a ready-made, customizable sign-in/sign-up web page; you don't build login screens.
- **MFA** — SMS or TOTP, optional or required.
- **Federation** — let users sign in with **social** (Google, Facebook, Apple) or **enterprise
  SAML/OIDC** IdPs; the User Pool normalizes them all into one set of Cognito JWTs.
- **Groups** — assign users to groups (claims in the token) for role-based logic.

```
User ──▶ Cognito Hosted UI / SDK ──▶ User Pool ──issues──▶ ID + Access + Refresh JWTs
                                          │
                                          └─ (optional) federates to Google / SAML IdP
```

**Integration with API Gateway**: configure a **Cognito authorizer** (REST) or **JWT
authorizer** (HTTP API) — the gateway validates the User Pool JWT and rejects unauthenticated
calls before they hit your Lambda. (See [02_api_gateway.md](02_api_gateway.md).)

---

## 4. Cognito Identity Pools — Temporary AWS Credentials

An **Identity Pool** exchanges a proof of identity (a token from a User Pool, Google, SAML, etc.)
for **temporary, scoped AWS credentials** issued by **STS**.

```
1. User authenticates ──▶ gets a token (User Pool JWT, Google token, SAML assertion)
2. App sends that token ──▶ Identity Pool
3. Identity Pool ──assumes──▶ IAM role (authenticated role) via STS
4. STS ──returns──▶ temporary AWS credentials (access key, secret, session token)
5. App uses those creds ──▶ calls S3 / DynamoDB / etc. DIRECTLY, scoped by the IAM role
```

- **Authenticated vs unauthenticated roles** — Identity Pools can grant a (very limited) guest
  role to unauthenticated users *and* a broader role to authenticated ones.
- **Role-based / attribute-based access** — map different IdP groups/claims to different IAM roles.
- Credentials are short-lived and automatically refreshed by the SDK.

> **Key insight**: Identity Pools exist so your app **doesn't embed long-term AWS keys**. The
> browser/mobile client gets temporary STS credentials scoped by an IAM role — the secure way to
> let end users touch AWS resources directly.

---

## 5. When Each — And Both Together

```
SCENARIO A: "Sign users into my app / protect my API"
   User ──▶ User Pool ──JWT──▶ API Gateway (Cognito authorizer) ──▶ Lambda
   (User Pool only — no AWS resource access by the client)

SCENARIO B: "Let users access AWS directly with no login"
   Guest ──▶ Identity Pool ──(unauth role)──▶ STS creds ──▶ S3 (read public assets)
   (Identity Pool only — for guest/anonymous AWS access)

SCENARIO C (the common combo): "Sign users in AND let them upload to S3 from the browser"
   User ──▶ User Pool ──JWT──▶ Identity Pool ──▶ STS creds (scoped IAM role) ──▶ S3 PutObject
   (User Pool authenticates → Identity Pool authorizes AWS access)
```

| You need | Use |
|----------|-----|
| Sign-up/sign-in, hosted UI, MFA, social/enterprise login | **User Pool** |
| Validate logged-in users at API Gateway | **User Pool** authorizer |
| Direct, scoped AWS resource access for end users (browser/mobile) | **Identity Pool** |
| Both: authenticate users *and* give them direct AWS access | **User Pool → Identity Pool** |

⚠️ **Exam trap**: "users need to upload directly to an S3 bucket from a mobile app" → you need an
**Identity Pool** (for STS credentials), typically fed by a **User Pool** (for the login). A User
Pool alone produces a JWT but **no AWS credentials**.

---

## 6. Enterprise Identity and Recovery

### Federation and managed login

A user pool can federate external SAML and OIDC identity providers as well as
social providers. Map external attributes to a stable internal subject and decide
which provider remains authoritative for profile updates, account disablement,
MFA, and recovery. Use domain and IdP discovery carefully so a user cannot choose
a weaker provider for the same privileged account.

Cognito **managed login** (the current hosted sign-in experience; older material
calls it the hosted UI) implements OAuth/OIDC redirects, sign-up, recovery, MFA,
and federation. It reduces security-sensitive UI work, but the application still
must use Authorization Code with PKCE for public clients, validate issuer,
audience/client ID, signature, expiry, and state/nonce, and register exact redirect
and sign-out URIs. Never put a client secret in browser or mobile code.

### Token and session strategy

Keep access/ID tokens short-lived and use refresh tokens for controlled session
renewal. Revoking a refresh token or performing global sign-out prevents future
refresh and invalidates the relevant token family for Cognito-aware validation,
but a service that only verifies an already-issued JWT locally may accept it
until expiry. For immediate high-risk revocation, use short access-token lifetime
plus a server-side session/deny check on sensitive operations.

Store browser tokens in a way that addresses XSS and CSRF for the chosen app
architecture. Rotate refresh tokens where supported, detect reuse, and do not log
tokens. Group and custom claims help authorization, but backend policy must not
trust editable profile attributes as privilege.

**Threat protection** can evaluate sign-in risk and apply adaptive authentication
such as additional MFA or blocking, depending on configuration/tier. Start with
audit visibility, tune for the user population, and retain a recovery path for
false positives. Adaptive authentication supplements strong MFA and rate controls;
it does not replace them.

### Lambda triggers and failure boundaries

User pools can invoke Lambda triggers for custom messages, pre-sign-up checks,
token customization, migration, and other lifecycle points. Keep synchronous
triggers fast, least-privileged, idempotent, and highly available. A timeout,
permission error, bad deployment, or unavailable dependency can block sign-in or
sign-up. Version triggers, canary changes, alarm on Cognito and Lambda errors,
and retain a tested rollback.

### Multi-Region limitations and disaster recovery

Cognito supports **multi-Region replication (MRR)** for eligible user pools on
modern infrastructure and the required feature plans. MRR creates one secondary
replica Region with a shared directory, a multi-Region OIDC issuer, and eventual
replication. It requires a multi-Region customer managed KMS key. The primary
remains authoritative for user/configuration writes; this is business-continuity
replication, not active-active administration.

Important limitations shape the runbook:

- The secondary can authenticate existing replicated users during failover, but
  cannot sign up/admin-create users, reset passwords, or modify profiles.
- TOTP MFA is not supported in the secondary, and some counters/state are not
  synchronized. A federated user must previously have signed in through the
  primary before the secondary can authenticate that user.
- Email/SMS, Lambda triggers, WAF, log export, tags, and other regional settings
  need review/configuration per replica. A trigger dependency can still make the
  recovery Region unavailable.
- Automatic managed-login/federation failover uses a custom domain and a Route
  53 health signal. Applications calling regional Cognito APIs/SDKs must select
  the healthy regional endpoint themselves.

For ineligible pools, a separately created pool plus external-IdP sign-in,
user-migration triggers, or forced native-user reset may be the fallback, but it
does not preserve all passwords, MFA state, sessions, or subject identifiers.
Infrastructure as code recreates configuration; it is not user-directory backup.

Test API authorizers, identity-pool roles, KMS/secrets, email/SMS, triggers,
DNS/custom domains, quotas, replication delay, and issuer validation. Document
RTO/RPO in user terms: existing-token behavior, new sign-in, password/profile
operations, MFA limitations, and direct AWS access. Test return to the primary;
MRR has one writable authority, and a custom two-pool fallback can change subject
identifiers and authorization claims.

### Cognito or IAM Identity Center?

Use Cognito for **customer/consumer application identities** and optional direct
app access to AWS services. Use **IAM Identity Center** for employees and other
workforce identities accessing AWS accounts and business applications, normally
federated from the corporate directory. Do not build workforce account access by
placing employees in a Cognito user pool and vending broad identity-pool roles.

---

## 7. Key Exam Points

- **User Pool = authentication / user directory → issues JWTs.** **Identity Pool = authorization to
  AWS → issues temporary STS credentials** mapped to an IAM role.
- User Pools provide **hosted UI, MFA, password policies, and federation** (social + SAML/OIDC).
- API Gateway uses a **Cognito (User Pool) authorizer** to validate JWTs and protect endpoints.
- Identity Pools support **authenticated and unauthenticated (guest) roles**.
- "Direct AWS access from a browser/mobile client" → **Identity Pool** (never embed long-term keys).
- The common pattern is **User Pool feeding an Identity Pool**: authenticate, then get scoped creds.
- JWTs are **stateless** — validated by signature; no server-side session store.
- Managed login and external IdPs reduce login implementation, but clients/backends still validate OAuth/OIDC state, issuer, audience, signature, expiry, and redirects.
- Eligible Cognito user pools can use MRR with one secondary, a multi-Region key/issuer, and a single writable primary; DR must account for secondary write, MFA, federation, trigger, and API-routing limitations.
- Cognito serves application users; **IAM Identity Center** serves workforce access to AWS accounts and business applications.

---

## 8. Common Mistakes

- ❌ Thinking a User Pool gives AWS credentials. ✅ It gives JWTs; you need an Identity Pool for STS creds.
- ❌ Embedding IAM access keys in a mobile/web app. ✅ Use an Identity Pool for temporary, scoped creds.
- ❌ Building custom login pages. ✅ Use the Cognito hosted UI unless you need a fully custom flow.
- ❌ Confusing the two pools because of the names. ✅ User Pool = *login*; Identity Pool = *AWS access*.
- ❌ Treating federation as Identity-Pool-only. ✅ User Pools federate to social/SAML IdPs too.
- ❌ Revoking refresh tokens and assuming every backend will reject all previously issued JWTs immediately.
- ❌ Adding a synchronous Lambda trigger with an unreliable dependency and no rollback, making that dependency part of sign-in availability.
- ❌ Calling an IaC copy of a user pool multi-Region DR without a plan for passwords, MFA, tokens, subjects, and external IdPs.
- ❌ Using Cognito as the employee portal for AWS accounts instead of IAM Identity Center.

---

## 9. Limits & Quick Facts

| Item | Detail |
|------|--------|
| User Pool tokens | ID token, access token (1 hr default), refresh token (configurable up to 10 yrs) |
| MFA | SMS, TOTP |
| User Pool federation | Social (Google/Facebook/Apple), SAML 2.0, OIDC |
| Identity Pool credential source | AWS STS (temporary credentials) |
| Identity Pool roles | Authenticated + (optional) unauthenticated/guest |
| Pricing | Per monthly active user (MAU); generous free tier |

---

**Next**: [Messaging & Events — Decoupling Concepts](../11_messaging/01_messaging_concepts.md)
