# Secrets Manager, Parameter Store & ACM

> **Who this is for**: Engineers preparing for SAA-C03 who understand KMS and now need to know
> where AWS stores the *secrets* (DB passwords, API keys) and *certificates* (TLS) that ride on
> top of that key material — and which service the exam expects for a given scenario. Before
> this, read **[01_encryption_and_kms.md](01_encryption_and_kms.md)** — both services below
> encrypt their data with KMS.

---

## 1. The Problem: Where Do Secrets Live?

Hard-coding a database password in code or an environment variable is the classic anti-pattern —
it leaks into source control, logs, and AMIs. AWS gives you two managed stores, and the exam
loves to make you pick between them:

- **AWS Secrets Manager** — purpose-built for secrets, with **automatic rotation**.
- **SSM Parameter Store** — a general config/parameter store that *can* hold secrets as
  encrypted `SecureString` values, **free**, but with **no built-in rotation**.

Both encrypt sensitive values with **KMS** and gate access with **IAM**.

---

## 2. Secrets Manager vs Parameter Store

| Feature | **Secrets Manager** | **SSM Parameter Store** |
|---------|---------------------|--------------------------|
| Primary purpose | Storing & **rotating** secrets | Config + parameters (secrets via `SecureString`) |
| **Automatic rotation** | ✅ Built-in, via a **Lambda** function | ❌ None (you'd build it yourself) |
| Native RDS/Redshift/DocumentDB rotation | ✅ Managed rotation templates | ❌ |
| Encryption | KMS (always) | `SecureString` type uses KMS; `String` is plaintext |
| Cost | **~$0.40 / secret / month** + API calls | **Free** for standard params; advanced tier paid |
| Versioning | ✅ Staged versions (`AWSCURRENT` / `AWSPREVIOUS`) | ✅ Version history per parameter |
| Cross-account / cross-region | ✅ Resource policy + replica secrets | Limited |
| Size limit | 64 KB | 4 KB standard / 8 KB advanced |
| Hierarchy / path queries | Flat (with tags) | ✅ Path hierarchy (`/app/prod/db/password`) `GetParametersByPath` |
| Generate random secret | ✅ `GetRandomPassword` | ❌ |

> **Rule of thumb for the exam**:
> - Needs **automatic rotation** of a credential (especially **RDS**) → **Secrets Manager**.
> - Just need to store config / a secret cheaply with **no rotation requirement** → **Parameter
>   Store `SecureString`** (free).
> - "Most cost-effective way to store a secret" with no rotation mentioned → **Parameter Store**.
> - "Automatically rotate the DB password every 30 days" → **Secrets Manager**.

💡 Parameter Store can actually *reference* a Secrets Manager secret, so they compose — but for
the exam, treat the picker above as the decision.

---

## 3. Secrets Manager Deep Dive

**Automatic rotation** is the headline feature. You attach a **Lambda rotation function**;
Secrets Manager invokes it on a schedule (e.g., every 30 days) to generate a new secret value
and update both the secret and the target service.

```
   Rotation schedule fires
          │
          ▼
   ┌──────────────────┐   1. createSecret  ┌───────────────┐
   │ Secrets Manager  │ ─────────────────► │ Rotation       │
   │  (AWSPENDING)    │   2. setSecret     │ Lambda         │
   │                  │ ◄───────────────── │ (changes the   │
   │                  │   3. testSecret    │  password on   │
   │                  │   4. finishSecret  │  the database) │
   └──────────────────┘ ──► AWSCURRENT     └───────┬────────┘
                                                    │ ALTER USER ...
                                                    ▼
                                              ┌──────────┐
                                              │   RDS    │
                                              └──────────┘
```

- **RDS / Aurora / Redshift / DocumentDB** have **managed rotation** — AWS supplies the Lambda;
  you just enable it. Other secrets use a custom Lambda.
- **Versioning via staging labels**: the new value is `AWSPENDING`, promoted to `AWSCURRENT` on
  success, and the old one becomes `AWSPREVIOUS` (instant rollback if the new one fails its test
  step).
- Encrypted with KMS; access controlled by IAM identity policies **and** an optional resource
  policy for cross-account sharing.

✅ Best fit: production database credentials that compliance requires rotating, or any secret
where a leaked-then-rotated credential should auto-heal.

---

## 4. ACM — AWS Certificate Manager

**TLS termination recap**: a client connects over HTTPS; somewhere the encrypted connection is
*terminated* (decrypted) and the server presents an **X.509 certificate** proving its identity.
That certificate must be issued by a trusted CA and kept renewed. ACM manages that lifecycle.

```
  client ──HTTPS (TLS)──► [ ALB / CloudFront / API GW ]  ──HTTP──► backend
                          ▲ presents ACM certificate          (re-encrypt
                          │ (TLS terminated here)              optional)
```

What ACM gives you:

- **Free public TLS/SSL certificates** for use with integrated AWS services.
- **Automatic renewal** — ACM re-issues before expiry (for DNS-validated certs), eliminating the
  classic "expired cert took down prod at 2am" outage. ⚠️ This auto-renewal only applies to
  certs **deployed on integrated AWS services**; a cert you exported/imported is *not*
  auto-renewed.
- **Domain validation** via DNS (Route 53 — preferred, hands-off) or email.

### Where ACM certs attach

| Integration | Notes |
|-------------|-------|
| **Elastic Load Balancing (ALB/NLB)** | **Regional** cert — must be in the **same region** as the load balancer |
| **CloudFront** | Cert **must be in `us-east-1` (N. Virginia)** — global edge service |
| **API Gateway** | Edge-optimized endpoints → `us-east-1`; regional endpoints → that region |
| **EC2 / on-instance** | ❌ Cannot attach an ACM public cert directly to an EC2 web server (no export of the private key for public certs) |

> **Rule**: For **CloudFront**, the ACM certificate **must live in `us-east-1`**, regardless of
> where your origin is. This is one of the most frequently tested ACM facts. For **ALB**, the
> cert must be in the **same region** as the ALB.

### ACM Private CA

**AWS Private CA** (a separate, paid service) issues **private certificates** for internal
services, mutual TLS, and IoT — certs trusted only inside your org, not by public browsers. ACM
can manage and auto-renew these private certs. Use it when you need an internal PKI; use plain
public ACM certs for anything internet-facing.

---

## Key Exam Points

- **Automatic rotation** of credentials (especially **RDS**) → **Secrets Manager** (rotation via
  Lambda; AWS supplies managed Lambdas for RDS/Redshift/DocumentDB).
- **Cheapest** secret storage with **no rotation** requirement → **Parameter Store
  `SecureString`** (free).
- Both encrypt with **KMS** and authorize with **IAM**.
- **ACM** = free public certs + **automatic renewal**; integrates with ELB, CloudFront, API GW.
- **CloudFront certs must be in `us-east-1`**; ALB/NLB certs must be in the **same region** as
  the LB.
- You **cannot** install an ACM **public** cert directly on EC2 — terminate TLS at the ALB/CF, or
  use Private CA for internal certs.
- Parameter Store offers a **path hierarchy** and `GetParametersByPath`; Secrets Manager offers
  staged versions (`AWSCURRENT`/`AWSPENDING`/`AWSPREVIOUS`).

---

## Common Mistakes

- ❌ Choosing Secrets Manager when no rotation is required and cost is the deciding factor — the
  free **Parameter Store** is the intended answer.
- ❌ Putting a CloudFront ACM cert in the origin's region instead of **`us-east-1`**.
- ❌ Expecting ACM to auto-renew a cert you **exported/imported** onto your own server — only
  certs on integrated AWS services auto-renew.
- ❌ Storing secrets as a plaintext `String` parameter instead of `SecureString` — `String` is
  not encrypted.
- ❌ Trying to attach a public ACM cert directly to an **EC2** instance — terminate at the load
  balancer instead.
- ❌ Confusing **Secrets Manager rotation** (managed, Lambda-driven) with Parameter Store, which
  has **no** rotation at all.

---

**Next**: [03_threat_detection_services.md — WAF, Shield, GuardDuty, Inspector, Macie, Detective & Security Hub](03_threat_detection_services.md)
