# Encryption Fundamentals & AWS KMS

> **Who this is for**: Engineers preparing for SAA-C03 who can read an IAM policy but have
> never had to reason about *how* data gets encrypted or where the keys live. We start from
> first principles (at rest vs in transit, symmetric vs asymmetric, envelope encryption) and
> build up to KMS and when you'd reach for CloudHSM instead. Before this, understand the IAM
> permission model: **[IAM & Identity](../02_iam/README.md)**.

---

## 1. Encryption at Rest vs in Transit

Two completely separate problems, often tested together because a "fully encrypted" system
needs both.

| | Encryption **at rest** | Encryption **in transit** |
|---|---|---|
| Protects | Data sitting on disk / in a bucket / in a DB | Data moving across a network |
| Threat | Stolen disk, snapshot, or bucket leak | Eavesdropping / man-in-the-middle on the wire |
| Mechanism | Symmetric encryption of the stored bytes (AES-256) | TLS/SSL (HTTPS), VPN, IPsec |
| AWS examples | EBS volume encryption, S3 SSE, RDS storage encryption | HTTPS on ALB/CloudFront, TLS to RDS, VPN tunnels |

```
        IN TRANSIT                              AT REST
  client ──TLS (HTTPS)──► ALB ──TLS──► EC2 ──writes──► [ encrypted EBS ]
         (data encrypted                              (data encrypted on
          on the wire)                                 the physical disk)
```

> **Key insight**: The two are independent. An object can be encrypted at rest in S3 (SSE-KMS)
> yet downloaded over plain HTTP, or sent over HTTPS to an unencrypted disk. Exam scenarios that
> say "data must be encrypted **end to end**" almost always require **both**.

💡 At rest is virtually always **symmetric** (fast, bulk data). In transit uses TLS, which uses
asymmetric crypto only to *negotiate* a symmetric session key — then bulk traffic is symmetric.
The next section explains why.

---

## 2. Symmetric vs Asymmetric Keys

| | **Symmetric** | **Asymmetric** |
|---|---|---|
| Keys | One shared secret key (encrypt = decrypt) | A **public/private key pair** |
| Speed | Very fast — used for bulk data | Slow — used for small payloads / negotiation |
| Algorithm | AES-256 (typical) | RSA, ECC |
| Problem it has | How do you *share* the secret key safely? | Slow; not for large data |
| Use cases | Disk/object/DB encryption | TLS handshake, digital signatures, encrypting a key you'll send to someone |

The fundamental tension: symmetric is fast but you must somehow distribute the secret key;
asymmetric solves distribution but is too slow for bulk data. **Real systems combine both** —
asymmetric to protect a symmetric key, symmetric for the actual data. That hybrid is exactly
what envelope encryption is.

---

## 3. Envelope Encryption (DEK + KEK) — the Core Mental Model

This is the single most important concept for understanding KMS. You **do not** encrypt large
data directly with a KMS key. Instead:

- **DEK (Data Encryption Key)** — a symmetric key (AES-256) that actually encrypts your data.
  It's generated fresh, used locally, then thrown away.
- **KEK (Key Encryption Key)** — the long-lived KMS key that *encrypts the DEK*. The KEK
  (the KMS key) **never leaves KMS** unencrypted.

You store the **encrypted DEK right next to the encrypted data**. To read the data later, you
ask KMS to decrypt the DEK, then use the plaintext DEK locally.

```
                         ENVELOPE ENCRYPTION

  ┌──────────────────────────── KMS ─────────────────────────────┐
  │   KMS key (KEK)  — symmetric master key, NEVER leaves KMS     │
  └───────────────┬───────────────────────────▲──────────────────┘
        GenerateDataKey                   Decrypt(encrypted DEK)
                  │                             │
                  ▼                             │
   returns TWO things:                          │
   ┌──────────────────────┐                     │
   │ plaintext DEK  ───────┼──► encrypt data ──► [ ciphertext data ]
   │ encrypted DEK  ───────┼──────────────┐      ┌──────────────────┐
   └──────────────────────┘               └────► │ stored together: │
        (plaintext DEK is wiped                   │  encrypted data  │
         from memory after use)                   │  + encrypted DEK │
                                                  └──────────────────┘
```

**Why bother?**

- ✅ **Performance** — bulk data is encrypted locally with fast symmetric AES, not round-tripped
  to KMS. KMS only ever sees the tiny DEK.
- ✅ **Scale** — KMS has a 4 KB payload limit on direct encrypt/decrypt. Envelope encryption has
  no data-size limit.
- ✅ **Rotation** — rotating the KEK doesn't require re-encrypting terabytes of data; only the
  small DEKs need re-wrapping.
- ✅ **Blast radius** — each object/volume can get its own DEK, so one leaked DEK exposes only
  that one object.

> **Key insight**: "The master key never leaves KMS; only data keys travel, and they travel
> encrypted." Every AWS service that says "encrypted with KMS" (EBS, S3 SSE-KMS, RDS) is doing
> envelope encryption under the hood — you usually never see the DEK at all.

---

## 4. KMS Keys (formerly CMK)

A **KMS key** (the API still calls older ones "Customer Master Keys / CMK") is the KEK — a
logical key managed inside KMS, identified by an ARN and (optionally) an **alias** like
`alias/prod-app`. The key material is **regional** and never exportable in plaintext.

### Key ownership types

| Type | Who creates/controls it | Rotation | Visible in your account | Cost | Use when |
|------|------------------------|----------|--------------------------|------|----------|
| **AWS-owned** | AWS, shared across many accounts | AWS-managed | ❌ Not visible | Free | Default when you do nothing; you don't need control |
| **AWS-managed** | Auto-created per service (`aws/s3`, `aws/ebs`, `aws/rds`) | **Auto, every 1 year**, can't disable | ✅ Visible, can't edit policy | Free (you pay API calls) | Easy default for a single service |
| **Customer-managed (CMK)** | You | **Optional**, configurable (default 1 yr) | ✅ Full control of policy, grants, enable/disable, delete | ~$1/key/month + API calls | You need key policies, rotation control, cross-account sharing, audit, or your own deletion schedule |

⚠️ Exam discriminator: if a question requires **control over rotation, key policy, disabling, or
cross-account access**, the answer is a **customer-managed key**. AWS-managed keys give you none
of that.

### Symmetric vs asymmetric KMS keys

- **Symmetric** (default) — single AES-256 key; used for `Encrypt`, `Decrypt`, `GenerateDataKey`.
  This is what EBS/S3/RDS use. **The key material never leaves KMS.**
- **Asymmetric** — RSA or ECC public/private pair. The **public key can be downloaded** so callers
  outside AWS can encrypt or verify signatures; the private key stays in KMS. Use for
  signing/verification or encrypt-outside-AWS scenarios. Cannot be auto-rotated.

---

## 5. Key Policies vs IAM, and Grants

Access to a KMS key is governed by a **key policy** (a resource-based policy attached to the
key) — and this differs from most AWS services.

| | **Key policy** | **IAM policy** | **Grants** |
|---|---|---|---|
| Attached to | The KMS key itself | An IAM user/role | The key (programmatically) |
| Required? | **Always present** — primary access control for the key | Optional, must be *allowed by* the key policy too | Optional |
| Good for | Defining who can administer/use the key | Granting org-wide patterns to identities | **Temporary, granular** delegation (often to AWS services) |
| Revocation | Edit the policy | Edit the policy | `RevokeGrant` — clean, scoped |

> **Rule**: For a customer-managed key, an IAM policy granting `kms:*` does **nothing** unless
> the **key policy** also allows that principal (directly, or by delegating to IAM via the
> `"Principal": {"AWS": "arn:aws:iam::<acct>:root"}` + `kms:*` statement). This is the classic
> "I have admin but get AccessDenied on KMS" gotcha.

**Grants** are temporary permissions a principal hands to another principal/service to use a key
for specific operations, often auto-created when a service needs to use your key on your behalf
(e.g., an EBS volume using your CMK). They support **grant constraints** (e.g., only with a
specific encryption context) and are revocable independently of policies.

---

## 6. Automatic Rotation & Multi-Region Keys

**Automatic rotation** (customer-managed symmetric keys):
- Enable it and KMS generates new backing key material **every year** (now configurable down to
  ~90 days for newer keys); the **key ID and ARN stay the same**.
- Old material is retained so previously encrypted data still decrypts — **no re-encryption
  needed**. Fully transparent.
- ❌ Not available for asymmetric keys or imported key material (you rotate those manually by
  creating a new key + alias).

**Multi-Region keys**:
- A primary key in one region plus **replica keys** in others that share the **same key ID and
  key material**.
- Lets you encrypt in `us-east-1` and **decrypt in `eu-west-1`** without a cross-region KMS call.
- Exam use cases: **cross-region DR**, global DynamoDB tables, and S3 Cross-Region Replication of
  encrypted objects. Regular KMS keys are region-locked, so without multi-region keys you'd have
  to re-encrypt on the far side.

💡 Multi-Region keys are an explicit opt-in; by default KMS keys are single-region.

---

## 7. How Services Integrate — the `GenerateDataKey` Flow

When you enable "KMS encryption" on EBS, S3 (SSE-KMS), or RDS, the service performs envelope
encryption for you:

```
  S3 PutObject with SSE-KMS
  ──────────────────────────
   1. S3 calls KMS: GenerateDataKey(keyId)
   2. KMS returns { plaintext DEK, encrypted DEK }
   3. S3 encrypts the object with the plaintext DEK (AES-256, local)
   4. S3 stores the encrypted object + the encrypted DEK
   5. S3 wipes the plaintext DEK from memory

  S3 GetObject
  ────────────
   1. S3 calls KMS: Decrypt(encrypted DEK)   ← key policy/grant checked here
   2. KMS returns the plaintext DEK
   3. S3 decrypts the object and returns it
```

- **EBS** — generates a DEK per volume; snapshots and volumes created from them inherit
  encryption. Cannot encrypt an existing unencrypted volume in place — you copy/snapshot to a new
  encrypted one.
- **RDS** — encryption enabled **only at creation**; can't toggle on an existing instance (you
  restore an encrypted snapshot to a new instance). Read replicas and snapshots inherit it.
- **S3** — SSE-KMS vs SSE-S3: SSE-KMS uses *your* key with full CloudTrail audit and key policy
  control; SSE-S3 uses an AWS-managed key transparently with no extra control.

⚠️ Because every decrypt is a KMS API call, very high-throughput workloads can hit KMS request
limits — **S3 Bucket Keys** reduce KMS calls by caching a bucket-level data key. Worth knowing
the name for the exam.

---

## 8. KMS vs CloudHSM — When You Need a Dedicated HSM

| | **AWS KMS** | **AWS CloudHSM** |
|---|---|---|
| Model | **Multi-tenant**, fully managed service | **Single-tenant** dedicated hardware (FIPS 140-2 **Level 3**) |
| Tenancy | Shared AWS-controlled HSMs | Your own HSM cluster in your VPC |
| Key control | AWS manages the HSMs; you manage logical keys | **You** fully control keys; AWS cannot access them |
| Access | AWS API + IAM/key policies | Industry-standard **PKCS#11, JCE, CNG** |
| Compliance | FIPS 140-2 Level 3 (validated) | When you need **exclusive, single-tenant** control or specific regulatory mandates |
| Effort | Low — fully managed | High — you manage the cluster, users, HA |

> **Rule**: Default to **KMS**. Reach for **CloudHSM** only when an exam states you need a
> **single-tenant / dedicated HSM**, **you must control the keys with no AWS access**, you need
> **PKCS#11/JCE/CNG**, or specific compliance demands a dedicated HSM. KMS can even use a CloudHSM
> cluster as a **custom key store** to get dedicated hardware behind the familiar KMS API.

---

## Key Exam Points

- **Envelope encryption** = DEK encrypts data, KEK (KMS key) encrypts the DEK; the master key
  never leaves KMS. This is *the* concept to know cold.
- `GenerateDataKey` returns **both** a plaintext and an encrypted DEK; store the encrypted one
  with the data.
- Need **control over rotation, policy, cross-account, or deletion** → **customer-managed key**.
- KMS access requires the **key policy** to allow the principal — IAM alone is not enough.
- **Automatic rotation** keeps the same ARN, no re-encryption, symmetric customer-managed keys
  only (yearly; not asymmetric or imported material).
- **Multi-Region keys** for cross-region DR / replication of encrypted data.
- RDS and EBS encryption is set **at creation**; flip it on an existing resource by restoring
  from an encrypted snapshot/copy.
- **CloudHSM** = single-tenant, FIPS 140-2 Level 3, you hold the keys; **KMS** = multi-tenant,
  managed, default choice.

---

## Common Mistakes

- ❌ Thinking you encrypt large data directly with a KMS key — KMS has a **4 KB limit**; use
  envelope encryption / `GenerateDataKey`.
- ❌ Granting `kms:*` in an IAM policy and expecting it to work without the **key policy**
  allowing the principal.
- ❌ Believing you can "turn on encryption" for an existing **RDS instance** or **EBS volume**
  in place — you must recreate from an encrypted snapshot/copy.
- ❌ Assuming automatic rotation re-encrypts existing data — it doesn't; old key material is
  retained transparently.
- ❌ Choosing **CloudHSM** for cost savings — it's *more* expensive and operationally heavy;
  it's for dedicated/compliance needs, not the default.
- ❌ Forgetting that a single-region KMS key can't decrypt in another region — use **multi-Region
  keys** for cross-region scenarios.

💡 Hands-on walkthroughs of these flows live in
[KMS Encryption Examples](../18_practical_examples/18_kms_encryption_examples.md).

---

**Next**: [02_secrets_manager_and_acm.md — Secrets Manager, Parameter Store & ACM](02_secrets_manager_and_acm.md)
