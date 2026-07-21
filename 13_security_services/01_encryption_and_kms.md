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
| **Customer-managed** | You | Optional and configurable; default automatic period is 365 days when enabled | ✅ Full control of policy, grants, enable/disable, delete | Per-key and API charges | You need key policies, rotation control, cross-account sharing, audit, or your own deletion schedule |

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
- For keys whose origin is `AWS_KMS`, enable it and KMS generates new backing key material on a
  configurable schedule (365 days by default); the **key ID and ARN stay the same**.
- Old material is retained so previously encrypted data still decrypts — **no re-encryption
  needed**. Fully transparent.
- It is not available for asymmetric, HMAC, or custom-key-store keys. Imported symmetric keys
  can use **on-demand rotation**, but not automatic rotation; the owner remains responsible for
  retaining every imported key-material version required to decrypt old ciphertext.

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
| Model | **Multi-tenant**, fully managed service using FIPS 140-3 validated HSMs | **Single-tenant** dedicated hardware; FIPS Level 3 validation depends on the HSM/cluster mode |
| Tenancy | Shared AWS-controlled HSMs | Your own HSM cluster in your VPC |
| Key control | AWS manages the HSMs; you manage logical keys | **You** fully control keys; AWS cannot access them |
| Access | AWS API + IAM/key policies | Industry-standard **PKCS#11, JCE, CNG** |
| Compliance | Standard KMS HSMs are FIPS 140-3 Security Level 3 validated | When you need **exclusive, single-tenant** control or specific regulatory mandates |
| Effort | Low — fully managed | High — you manage the cluster, users, HA |

> **Rule**: Default to **KMS**. Reach for **CloudHSM** only when an exam states you need a
> **single-tenant / dedicated HSM**, **you must control the keys with no AWS access**, you need
> **PKCS#11/JCE/CNG**, or specific compliance demands a dedicated HSM. KMS can even use a CloudHSM
> cluster as a **custom key store** to get dedicated hardware behind the familiar KMS API.

---

## 9. Sharing Encrypted Data Across Accounts & Regions

Two scenarios the exam loves, because both fail in the same non-obvious way: **the data is
shareable, but the recipient also needs access to the KMS key that wraps it.** Encryption is
only half the grant — the key is the other half.

### 9a. Encrypted AMI Sharing

An AMI references one or more **EBS snapshots**, and those snapshots can be encrypted with a KMS
key. To share an encrypted AMI with another account you must line up **three** things:

1. **Launch permission** on the AMI — add the target account ID to the AMI's permissions.
2. **Snapshot access via the key** — the AMI's snapshots are encrypted, so the target account
   needs `kms:Decrypt`/`ReEncrypt`/`CreateGrant` etc. on the **CMK**. That means a
   **customer-managed key**, with the target account added to the **key policy**.
3. On the receiving side, the target account's IAM principal needs IAM permissions to use that
   shared key.

```
  Source account (111111111111)            Target account (222222222222)
  ────────────────────────────             ────────────────────────────
   AMI ──► encrypted EBS snapshot
            │ encrypted with CMK
            ▼
   ┌─────────────────────────┐   share AMI (launch perm)
   │ Customer-managed KMS key │ ───────────────────────────►  launches EC2
   │  key policy allows 222…  │   + key policy grants 222…    (recommended:
   └─────────────────────────┘                                copy AMI → re-encrypt
                                                               with its OWN key)
```

⚠️ **You cannot share an AMI encrypted with an AWS-managed key** (`aws/ebs`). AWS-managed and
AWS-owned keys can't have other accounts added to their policy — so encrypted AMI sharing
**requires a customer-managed key**. This is the single most-tested fact here.

💡 Best practice once shared: the target account **copies the AMI and re-encrypts it with its
own CMK**, so it isn't permanently dependent on the source account's key (which could be revoked
or deleted).

### 9b. S3 Replication with KMS Encryption

S3 Cross-Region (CRR) or Same-Region (SRR) Replication of SSE-KMS-encrypted objects has its own
wrinkles, because **a KMS key is regional** — the destination region can't use the source
region's key.

- You must **explicitly enable** replication of KMS-encrypted objects in the replication rule
  (it's off by default) and **specify the destination KMS key** to encrypt replicas with.
- The **replication IAM role** needs `kms:Decrypt` on the **source** key and
  `kms:Encrypt`/`GenerateDataKey` on the **destination** key.
- For cross-region, either use a **destination-region key** or a **Multi-Region key** (see §6)
  so the same logical key ID works on both ends.
- **SSE-C is different**: S3 can replicate eligible SSE-C objects without additional KMS
  permissions. The special opt-in and source/destination key permissions here apply to SSE-KMS
  and DSSE-KMS objects.

```
  us-east-1                              eu-west-1
  ─────────                              ─────────
  [ object + SSE-KMS (key A) ]
        │   replication role:
        │   Decrypt with key A,  Encrypt with key B
        ▼
  ───────────── CRR ─────────────►  [ replica + SSE-KMS (key B) ]
```

> **Key insight**: Replication moves the *object*, not the *key*. The destination must have its
> own usable key and the replication role must be allowed to decrypt on the source and encrypt on
> the destination — otherwise replication silently fails for those objects.

---

## 10. Enterprise Key Architecture

The difficult production decision is usually not the cipher. It is **who owns the key, who may
administer it, which workloads may use it, and how the organization can recover from a bad
change**.

### Prefer workload ownership unless central ownership has a reason

| Pattern | Good fit | Benefit | Main cost or failure mode |
|---------|----------|---------|---------------------------|
| **Workload-owned keys** in each application account | Normal EBS, S3, RDS, and application encryption | Local lifecycle and service integration; a key incident is contained to one workload | Governance must be enforced consistently through IaC, Config, and organization guardrails |
| **Central key account** | A security team must control decrypt authorization, several accounts intentionally share encrypted data, or regulation requires separation of duties | One controlled policy boundary and central audit ownership | Cross-account dependency, larger blast radius, shared quotas, and harder service integration and recovery |

Use workload-owned keys as the default and centralize **standards and evidence** rather than
putting every workload behind one organization-wide key. A central key account is appropriate
only after verifying that the AWS service accepts a cross-account KMS key ARN. Some services do
not, and consoles do not always list external keys even when the API accepts their ARN.

In either pattern:

- Use separate roles for **key administration** and **key usage**. A workload role that can
  decrypt data should not be able to edit the policy or schedule deletion.
- Assign an owner, data classification, workload, environment, and recovery contact with tags
  and an inventory. Use stable aliases in application configuration, but authorize the key ARN,
  not an alias that another principal could repoint.
- Keep a tightly controlled break-glass path that can restore a policy or cancel deletion, and
  test it before an incident.
- Avoid one key for an entire organization. Split at least by workload, environment, and
  sensitivity so policy errors, throttling, and deletion have a bounded impact.

### Policies for standing access; grants for delegated use

A **key policy** should define durable administrators and usage boundaries. IAM policies then
grant named roles only the operations they need. A **grant** is better when an AWS service or a
provisioning workflow needs a narrow, revocable delegation without repeatedly editing the key
policy.

Grants deserve the same care as policies. `CreateGrant` is powerful; constrain who can call it,
the allowed grant operations, the grantee, and whether the request comes through the intended
AWS service. Grants are eventually consistent, so use the returned grant token if a just-created
grant must be used immediately. Retire or revoke grants when the resource is deleted.

For symmetric encryption, bind authorization to the data with an **encryption context**:

```json
{
  "Condition": {
    "StringEquals": {
      "kms:EncryptionContext:tenant-id": "tenant-123",
      "kms:ViaService": "s3.eu-west-1.amazonaws.com"
    }
  }
}
```

The encryption context is authenticated additional data: the exact context used to encrypt is
required to decrypt. Policies can test `kms:EncryptionContext:<name>` and grants can use
`EncryptionContextEquals` or `EncryptionContextSubset`. It is logged in plaintext in CloudTrail,
so put stable identifiers there, **never a password, token, or personal data**. Also use
`kms:ViaService`, `kms:CallerAccount`, and the service's supported `aws:SourceArn` or
`aws:SourceAccount` conditions to stop a key from being used outside the intended integration.

### Cross-account and cross-service authorization

Cross-account KMS use requires two independent permissions:

1. The **key policy in the owner account** permits the external account or role.
2. An **identity policy in the caller account** permits that caller to use the key.

Then verify the service's own resource policy/service role and that the service supports an
external key. Cross-account KMS permissions are primarily cryptographic and grant operations;
an external account cannot become a key administrator merely because its IAM policy says
`kms:*`. Prefer a specific role over an account-wide principal, pass the full key ARN, and scope
service use with encryption-context and source conditions. CloudTrail records the operation in
both the caller and key-owner accounts, so an investigation must query both.

### Multi-Region keys are related, not globally administered

The primary and every replica share key ID, key material, and rotation state, but each is a
separate Regional resource with its own ARN. **Key policies, grants, aliases, tags, and
descriptions do not synchronize.** A grant created on the primary does not authorize a replica.

That means a DR deployment must provision and test equivalent authorization in every Region.
Use the KMS condition keys `kms:MultiRegion`, `kms:ReplicaRegion`, and `kms:PrimaryRegion` to
control who can create replicas or promote one. Choose multi-Region keys only when the same
ciphertext must move across Regions without re-encryption; they deliberately copy key material
across a Regional boundary and can complicate residency, policy, and audit requirements.

### Pick the key-origin model deliberately

| Model | What it adds | Operational consequence |
|-------|--------------|-------------------------|
| Standard KMS key (`AWS_KMS`) | Highest availability and the broadest features/service integration | AWS operates the HSM fleet; this is the production default |
| Imported material (`EXTERNAL`) | Bring your own symmetric material and optionally set expiration | You own durable backup and re-import recovery; expired or deleted material makes ciphertext unavailable |
| KMS custom key store backed by CloudHSM | KMS API and integrations with dedicated HSMs you operate | At least two active HSMs in different AZs; you manage cluster users, backup, capacity, availability, and cost |
| External key store (XKS) | Key material and cryptographic operations remain outside AWS | Your proxy, external key manager, network, latency, durability, and availability are now in the data path |
| CloudHSM directly | Dedicated HSM plus PKCS#11/JCE/CNG and direct key control | The application and operations team own HA, client integration, users, backups, and key lifecycle |

A custom key store is not automatically “more secure.” It gives more control by transferring
availability and durability work to you. CloudHSM and external custom key stores do not support
asymmetric/HMAC KMS keys, imported material, automatic rotation, or multi-Region keys. If a
CloudHSM custom store is disconnected, KMS cryptographic operations using its keys fail. Build
and load-test the cluster as a production dependency rather than treating it as passive storage.

### Lifecycle, quotas, and recovery

- **Disable before deleting.** Observe decrypt failures and CloudTrail use, identify every
  snapshot, object, database, AMI, backup, and application ciphertext that depends on the key,
  then schedule deletion for 7–30 days. Cancellation is possible during the window; after
  deletion, AWS cannot recover the key.
- For a multi-Region key, all replicas must be deleted before the primary can be deleted. The
  primary's waiting period does not begin until the last replica is gone.
- Rotate to reduce exposure, not as a substitute for revoking compromised access. KMS-managed
  rotation retains old material. Manual alias migration requires keeping the old key enabled
  until all old ciphertext has been re-encrypted or aged out.
- Treat imported material, XKS, and CloudHSM backups as a tested recovery system. A copy of
  encrypted data is useless if its key material, policy, external service, or HSM quorum cannot
  be restored.
- Design for quotas before centralizing. KMS quotas are scoped by account and Region; service
  calls made on your behalf consume the relevant request quota. Grants also have a per-key quota,
  and custom stores have additional throughput limits. Monitor usage, request adjustable quota
  increases early, shard keys where appropriate, and use service features such as S3 Bucket Keys
  to reduce call volume.

### Audit design

Use an organization CloudTrail delivered to a protected log-archive account, with retention and
integrity controls appropriate to the evidence requirement. KMS API calls are CloudTrail
**management events**; do not exclude KMS events just to save cost without understanding the
investigation gap. Alert through EventBridge/Security Hub
on high-risk changes such as `PutKeyPolicy`, `CreateGrant`, `DisableKey`,
`ScheduleKeyDeletion`, imported-material deletion or expiration, multi-Region replication or
promotion, and custom-store disconnects. AWS Config rules can continuously check enabled state,
rotation policy, and approved configuration.

An effective review can answer: **who changed the key, who used it, through which service, for
which workload/context, in which account and Region, and can the organization still decrypt its
required recovery data?** Preserve both sides of cross-account KMS events and periodically test a
restore, not just a key-policy screenshot.

---

## Key Exam Points

- **Envelope encryption** = DEK encrypts data, KEK (KMS key) encrypts the DEK; the master key
  never leaves KMS. This is *the* concept to know cold.
- `GenerateDataKey` returns **both** a plaintext and an encrypted DEK; store the encrypted one
  with the data.
- Need **control over rotation, policy, cross-account, or deletion** → **customer-managed key**.
- KMS access requires the **key policy** to allow the principal — IAM alone is not enough.
- **Automatic rotation** keeps the same ARN and requires no re-encryption. It applies to
  symmetric `AWS_KMS` customer-managed keys; imported symmetric material supports on-demand,
  not automatic, rotation.
- **Multi-Region keys** share material and rotation state, but policies, grants, aliases, and
  tags remain Regional and must be governed separately.
- **Sharing an encrypted AMI** requires a **customer-managed key** (add the account to the key
  policy + grant launch permission); AMIs encrypted with the AWS-managed key **can't** be shared.
- **S3 replication of SSE-KMS objects** must be explicitly enabled, needs a **destination-region
  key**, and the replication role needs decrypt-on-source + encrypt-on-destination. SSE-C objects
  are supported but follow the normal S3 replication path rather than the KMS-specific path.
- RDS and EBS encryption is set **at creation**; flip it on an existing resource by restoring
  from an encrypted snapshot/copy.
- **CloudHSM** = single-tenant dedicated HSMs and customer-operated availability; **KMS** =
  multi-tenant, managed, default choice.

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
- ❌ Treating a multi-Region key as one global policy — every replica has an independent policy,
  grant set, alias set, and ARN.
- ❌ Centralizing every key into one account without checking service support, quotas, and the
  organization-wide failure dependency that creates.
- ❌ Putting sensitive values in the encryption context — it is authenticated but appears in
  plaintext in CloudTrail.
- ❌ Scheduling key deletion without an inventory and tested recovery path — KMS cannot discover
  every ciphertext that still depends on a key.
- ❌ Trying to share an AMI encrypted with the **AWS-managed key** — it can't be shared; you need
  a **customer-managed key** with the target account in the key policy.
- ❌ Expecting SSE-KMS objects to replicate automatically — replication of KMS-encrypted objects
  must be **turned on explicitly** and the replication role granted access to **both** keys.

💡 Hands-on walkthroughs of these flows live in
[KMS Encryption Examples](../18_practical_examples/18_kms_encryption_examples.md).

---

**Next**: [02_secrets_manager_and_acm.md — Secrets Manager, Parameter Store & ACM](02_secrets_manager_and_acm.md)
