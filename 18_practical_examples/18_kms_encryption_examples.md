# KMS Encryption in Practice — EBS, S3, RDS, Secrets & Key Policies

> **Who this is for**: SAA-C03 candidates who understand the *theory* of envelope encryption
> and CMKs from [Encryption & KMS](../13_security_services/01_encryption_and_kms.md) but want
> to see exactly *how you turn it on* for each service — and which gotchas the exam tests
> (can't encrypt an existing RDS in place; cross-account needs key policy **and** IAM; can't
> share a snapshot encrypted with the default key).

---

## 1. The Mental Model: Envelope Encryption

KMS almost never encrypts your bulk data directly. Instead it issues a **data key** that
encrypts the data; the data key itself is encrypted by the **CMK** (customer master key /
KMS key) and stored *next to* the ciphertext. This is **envelope encryption**.

```
  GenerateDataKey(CMK)                                         decrypt path
  ───────────────────                                          ────────────
  KMS ──▶ returns TWO things:                       ciphertext + encrypted data key
          1. Plaintext data key  ──encrypts──▶ DATA        │
          2. Encrypted data key  ──stored beside ciphertext│  send encrypted data key to KMS
                                                            ▼  KMS decrypts it with the CMK
          (plaintext key wiped from memory ASAP)     plaintext data key ──decrypts──▶ DATA
```

> **Key insight**: The CMK never leaves KMS (it's hardware-backed). Only the small **data
> key** travels, and it travels encrypted. To read the data you must be allowed to call
> `kms:Decrypt` on the CMK — that's the whole access-control story.

CMK types: **AWS-managed** (`aws/ebs`, `aws/s3`, `aws/rds` — free, auto-rotated, you can't
edit the policy) vs **customer-managed** (you control the key policy, rotation, and grants).

---

## 2. (a) Encrypt an EBS Volume with a Customer-Managed Key

```bash
# Create the CMK once
aws kms create-key --description "ebs-app-data" --tags TagKey=app,TagValue=web
# → KeyId 1234abcd-12ab-34cd-56ef-1234567890ab

# Create an encrypted volume using that CMK
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 100 --volume-type gp3 \
  --encrypted \
  --kms-key-id 1234abcd-12ab-34cd-56ef-1234567890ab
```

⚠️ You **cannot encrypt an existing unencrypted volume in place**. The path is: **snapshot →
copy snapshot with `--encrypted` → create volume from the encrypted snapshot**.

```bash
aws ec2 copy-snapshot \
  --source-region us-east-1 --source-snapshot-id snap-0unencrypted \
  --encrypted --kms-key-id 1234abcd-...   # encrypt during copy
```

💡 Set **EBS encryption by default** at the account+Region level so every new volume is
encrypted automatically — a common "ensure all volumes are encrypted" exam answer.

---

## 3. (b) S3 Default Encryption: SSE-KMS with a Bucket Key

Set bucket default encryption to **SSE-KMS** and enable the **S3 Bucket Key** to slash KMS
costs (one data key per bucket-key period instead of one `Decrypt`/`GenerateDataKey` call per
object).

```bash
aws s3api put-bucket-encryption \
  --bucket prod-customer-uploads \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:111122223333:key/1234abcd-..."
      },
      "BucketKeyEnabled": true
    }]
  }'
```

| S3 encryption option | Key owner | Key management | Cost / notes |
|----------------------|-----------|----------------|--------------|
| **SSE-S3** (`AES256`) | AWS | Fully AWS-managed | Free, no KMS audit trail |
| **SSE-KMS** (`aws:kms`) | You or AWS-managed CMK | KMS, CloudTrail-audited | KMS request charges (mitigate with **Bucket Key**) |
| **DSSE-KMS** | You | Double-layer KMS | Highest assurance |
| **SSE-C** | You (supply key per request) | You hold the key | AWS never stores the key |

> **Rule**: "encrypted **and** auditable / per-key access control / rotation" → **SSE-KMS**.
> "encrypted with zero management" → **SSE-S3**. "millions of objects, KMS bill too high" →
> SSE-KMS **+ Bucket Key**.

---

## 4. (c) RDS Encryption at Rest — Enable at Creation

RDS storage encryption (which also covers automated backups, read replicas, and snapshots)
**must be enabled when the instance is created**.

```bash
aws rds create-db-instance \
  --db-instance-identifier prod-orders \
  --engine postgres --db-instance-class db.r6g.large \
  --allocated-storage 100 \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:111122223333:key/1234abcd-...
```

❌ You **cannot encrypt an existing unencrypted RDS instance in place**, and you cannot create
an encrypted read replica of an unencrypted primary. The required workaround:

```
unencrypted RDS ──snapshot──▶ snapshot ──copy-snapshot --kms-key-id──▶ ENCRYPTED snapshot
                                                                              │ restore
                                                                              ▼
                                                                  new ENCRYPTED RDS instance
                                                       (then repoint app / cut over)
```

```bash
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier prod-orders-snap \
  --target-db-snapshot-identifier prod-orders-snap-enc \
  --kms-key-id arn:aws:kms:us-east-1:111122223333:key/1234abcd-...
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-orders-enc \
  --db-snapshot-identifier prod-orders-snap-enc
```

⚠️ The reverse is also true: you can't *remove* encryption from an encrypted instance. And
you can't change which CMK encrypts an existing instance except via this snapshot→copy dance.

---

## 5. (d) Envelope Encryption with `GenerateDataKey` (Secrets / App Data)

When your *own* app encrypts a blob (the pattern Secrets Manager uses under the hood), you
call `GenerateDataKey`, encrypt locally with the plaintext key, then discard it:

```bash
aws kms generate-data-key \
  --key-id arn:aws:kms:us-east-1:111122223333:key/1234abcd-... \
  --key-spec AES_256
# Returns: Plaintext (use to encrypt, then wipe) + CiphertextBlob (store with the data)
```

```python
import boto3, os
from cryptography.fernet import Fernet  # illustrative symmetric cipher

kms = boto3.client("kms")
resp = kms.generate_data_key(KeyId=CMK_ARN, KeySpec="AES_256")
plaintext_key, encrypted_key = resp["Plaintext"], resp["CiphertextBlob"]

ciphertext = encrypt(plaintext_key, b"super-secret-config")  # encrypt locally
del plaintext_key                                            # never persist it

store(blob=ciphertext, wrapped_key=encrypted_key)            # save both together

# To read later: KMS decrypts the wrapped key, then you decrypt the blob locally
plaintext_key = kms.decrypt(CiphertextBlob=encrypted_key)["Plaintext"]
secret = decrypt(plaintext_key, ciphertext)
```

💡 **Secrets Manager** does exactly this for you and adds **automatic rotation** (e.g., a
Lambda that rotates an RDS password every 30 days). **SSM Parameter Store SecureString** also
uses KMS but has no built-in rotation — the exam contrasts the two on rotation.

---

## 6. (e) A Sample Key Policy

The **key policy** is the *primary* access control for a CMK — IAM policies alone are not
enough unless the key policy delegates to IAM. This policy keeps account admin via IAM and
grants a specific role decrypt rights:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnableRootAndIAM",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111122223333:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowAppRoleToUseKey",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111122223333:role/web-app-role" },
      "Action": ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey", "kms:DescribeKey"],
      "Resource": "*"
    }
  ]
}
```

> **Rule**: The `root` statement doesn't grant the root *user* powers — it means
> "permissions for this key **can be delegated via IAM** within account `111122223333`."
> Without it, only principals named directly in the key policy can ever use the key.

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `AccessDenied` on `kms:Decrypt` despite an IAM policy allowing it | Key policy has no statement delegating to IAM | Add the `root`/IAM-enable statement *or* name the principal in the key policy |
| Cross-account role can't use the key | Cross-account needs **both** the key policy (grant the external account) **and** the caller's IAM policy | Add the external account/role to the key policy **and** allow `kms:*` actions in their IAM |
| Can't share an encrypted EBS/RDS snapshot with another account | It's encrypted with the **default `aws/ebs` / `aws/rds`** key (can't be shared) | Re-encrypt the snapshot with a **customer-managed CMK**, then share the CMK + snapshot |
| "Can't encrypt existing RDS / EBS in place" | Encryption must be set at creation | snapshot → **copy with `--kms-key-id`** → restore/create new |
| S3 KMS bill exploding | One KMS call per object | Enable **S3 Bucket Key** |
| `KMS.DisabledException` / scheduled deletion | CMK disabled or pending deletion | Re-enable; you **cannot decrypt** data while the key is disabled/deleted |

⚠️ **Cross-account KMS = key policy AND IAM**, always both. This is the single most-tested KMS
gotcha. ✅ To share an encrypted snapshot, it must use a **customer-managed** CMK (default
service keys are non-shareable), and you must share the CMK with the target account too.

---

## Key Exam Points

- **Envelope encryption**: CMK encrypts a **data key**; the data key encrypts the data. CMK
  never leaves KMS. `GenerateDataKey` returns plaintext + encrypted copies.
- **AWS-managed CMKs** (`aws/ebs`, `aws/s3`, `aws/rds`) = free, non-editable policy,
  **not shareable**. **Customer-managed CMKs** = your policy, rotation, grants, shareable.
- **EBS and RDS can't be encrypted in place** — snapshot → copy with a CMK → restore.
- **RDS encryption must be enabled at creation** and covers backups, replicas, snapshots.
- **S3**: SSE-KMS for auditable, controllable encryption; add **Bucket Key** to cut KMS cost;
  SSE-S3 for zero-management; SSE-C when you hold the key.
- **Key policy is primary**; IAM only works if the key policy delegates (the `root`
  statement). **Cross-account requires key policy AND IAM**.
- **Secrets Manager** = KMS-backed secrets **with rotation**; **Parameter Store SecureString**
  = KMS-backed but no built-in rotation.

---

**Next**: [19_disaster_recovery_strategies.md — The Four DR Strategies Compared](19_disaster_recovery_strategies.md)
