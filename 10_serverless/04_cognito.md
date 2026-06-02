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

## 6. Key Exam Points

- **User Pool = authentication / user directory → issues JWTs.** **Identity Pool = authorization to
  AWS → issues temporary STS credentials** mapped to an IAM role.
- User Pools provide **hosted UI, MFA, password policies, and federation** (social + SAML/OIDC).
- API Gateway uses a **Cognito (User Pool) authorizer** to validate JWTs and protect endpoints.
- Identity Pools support **authenticated and unauthenticated (guest) roles**.
- "Direct AWS access from a browser/mobile client" → **Identity Pool** (never embed long-term keys).
- The common pattern is **User Pool feeding an Identity Pool**: authenticate, then get scoped creds.
- JWTs are **stateless** — validated by signature; no server-side session store.

---

## 7. Common Mistakes

- ❌ Thinking a User Pool gives AWS credentials. ✅ It gives JWTs; you need an Identity Pool for STS creds.
- ❌ Embedding IAM access keys in a mobile/web app. ✅ Use an Identity Pool for temporary, scoped creds.
- ❌ Building custom login pages. ✅ Use the Cognito hosted UI unless you need a fully custom flow.
- ❌ Confusing the two pools because of the names. ✅ User Pool = *login*; Identity Pool = *AWS access*.
- ❌ Treating federation as Identity-Pool-only. ✅ User Pools federate to social/SAML IdPs too.

---

## 8. Limits & Quick Facts

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
