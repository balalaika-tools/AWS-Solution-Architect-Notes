# KMS Encryption in Practice — EBS, S3, RDS, Secrets & Key Policies

> **Who this is for**: SAA-C03 candidates who understand the *theory* of envelope encryption
> and KMS keys from [Encryption & KMS](../13_security_services/01_encryption_and_kms.md) but want
> to see exactly *how you turn it on* for each service — and which gotchas the exam tests
> (can't encrypt an existing RDS in place; cross-account needs key policy **and** IAM; can't
> share a snapshot encrypted with the default key).

---

## 1. The Mental Model: Envelope Encryption

KMS almost never encrypts your bulk data directly. Instead it issues a **data key** that
encrypts the data; the data key itself is encrypted by the **KMS key** (older material calls
this a CMK) and stored *next to* the ciphertext. This is **envelope encryption**.

```
  GenerateDataKey(KMS key)                                     decrypt path
  ───────────────────                                          ────────────
  KMS ──▶ returns TWO things:                       ciphertext + encrypted data key
          1. Plaintext data key  ──encrypts──▶ DATA        │
          2. Encrypted data key  ──stored beside ciphertext│  send encrypted data key to KMS
                                                            ▼  KMS decrypts it with the KMS key
          (plaintext key wiped from memory ASAP)     plaintext data key ──decrypts──▶ DATA
```

> **Key insight**: The KMS key never leaves KMS (it's hardware-backed). Only the small **data
> key** travels, and it travels encrypted. To read the data you must be allowed to call
> `kms:Decrypt` on the KMS key — that's the whole access-control story.

KMS key types: **AWS-managed** (`aws/ebs`, `aws/s3`, `aws/rds` — no monthly key charge,
AWS-rotated, you can't
edit the policy) vs **customer-managed** (you control the key policy, rotation, and grants).

---

## 2. (a) Encrypt an EBS Volume with a Customer-Managed Key

```bash
# Create the customer-managed key once
aws kms create-key --description "ebs-app-data" --tags TagKey=app,TagValue=web
# → KeyId 1234abcd-12ab-34cd-56ef-1234567890ab

# Create an encrypted volume using that KMS key
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
| **SSE-KMS** (`aws:kms`) | You or an AWS-managed KMS key | KMS, CloudTrail-audited | KMS request charges (mitigate with **Bucket Key**) |
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
you can't change which KMS key encrypts an existing instance except via this snapshot→copy dance.

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
import os

import boto3

kms = boto3.client("kms")
KEY_ARN = os.environ["KMS_KEY_ARN"]
resp = kms.generate_data_key(KeyId=KEY_ARN, KeySpec="AES_256")
plaintext_key, encrypted_key = resp["Plaintext"], resp["CiphertextBlob"]

# Pseudocode below: use a vetted authenticated-encryption library and durable store.
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

The **key policy** is the *primary* access control for a customer-managed key — IAM policies alone are not
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

> **Rule**: The account-principal (`...:root`) statement gives the account control of the key and
> enables permissions to be delegated through IAM; it does **not** automatically allow every IAM
> principal. Protect the root credentials and grant operational access to named roles.

---

## 7. Cross-Account Encrypted Data Sharing

Sharing the S3 object or snapshot is not enough. The consumer needs permission to the **data
resource** and to the **KMS key**, and its own identity policy must allow both. The following
same-Region example shares `s3://shared-data/reports/*` from account `111122223333` with
`ReportReader` in account `444455556666` in the same AWS Organization.

### 7a. Resource policy in the data-owner account

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowConsumerRoleReadReports",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::444455556666:role/ReportReader"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::shared-data/reports/*",
    "Condition": {
      "StringEquals": {
        "aws:PrincipalOrgID": "o-exampleorgid"
      }
    }
  }]
}
```

The organization condition prevents this statement from continuing to work if the consumer
account leaves the organization. It is defense in depth, not a replacement for the named role.
Add `s3:ListBucket` on the bucket ARN only if the client actually lists keys, with an `s3:prefix`
condition for `reports/*`.

### 7b. Key policy in the key-owner account

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnableOwnerAccountIAM",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::111122223333:root"},
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowReportReaderThroughS3Only",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::444455556666:role/ReportReader"
      },
      "Action": ["kms:Decrypt", "kms:DescribeKey"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.us-east-1.amazonaws.com",
          "kms:CallerAccount": "444455556666"
        },
        "StringLike": {
          "kms:EncryptionContext:aws:s3:arn": "arn:aws:s3:::shared-data/reports/*"
        }
      }
    }
  ]
}
```

This key-policy example assumes ordinary per-object SSE-KMS encryption context. With **S3 Bucket
Keys**, S3 uses the bucket ARN rather than the object ARN in the KMS encryption context, so adjust
the condition and enforce the `reports/*` boundary in the bucket policy. Encryption context is
authenticated and appears in CloudTrail in plaintext; use identifiers, never secret values.

### 7c. Identity policy on the consumer role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::shared-data/reports/*"
    },
    {
      "Effect": "Allow",
      "Action": ["kms:Decrypt", "kms:DescribeKey"],
      "Resource": "arn:aws:kms:us-east-1:111122223333:key/1234abcd-..."
    }
  ]
}
```

All three policies are required. An SCP, permissions boundary, session policy, VPC endpoint
policy, or explicit deny can still block the request. Use the full external key ARN; another
account's alias is not a portable identifier and consoles do not always list external keys.

For EBS/RDS snapshots, replace the bucket policy with the service's snapshot/restore sharing
permission and verify that the target service supports the cross-account workflow. The source
snapshot must use a customer-managed key. A robust consumer copies the snapshot and re-encrypts it
under a key it owns, removing the long-term dependency on the producer's key and deletion policy.

---

## 8. Grants and AWS Service Roles

Use key policies for durable ownership boundaries and **grants** for narrow delegation created as
part of a resource lifecycle. A grant names a grantee, operations, and optional encryption-context
constraints; it can be retired/revoked without rewriting the key policy.

```bash
aws kms create-grant \
  --key-id arn:aws:kms:us-east-1:111122223333:key/1234abcd-... \
  --grantee-principal arn:aws:iam::111122223333:role/tenant-worker \
  --operations Decrypt GenerateDataKey \
  --constraints 'EncryptionContextSubset={tenant-id=tenant-123}'
```

KMS grants are eventually consistent. Pass the returned grant token when the grantee must use a
new grant immediately, record the grant ID with the resource that owns it, and revoke/retire it on
resource deletion. Restrict who can call `CreateGrant`; otherwise a principal can create a durable
path around the intended application policy.

AWS integrations do not all call KMS under the same principal:

| Integration | Principal/path to authorize |
|-------------|-----------------------------|
| S3 request using SSE-KMS | The caller is authorized and KMS use is constrained with `kms:ViaService` and S3 encryption context |
| S3 replication | The configured replication IAM role needs source decrypt and destination encrypt permissions, with both key policies if cross-account |
| EC2 launch from an encrypted EBS snapshot | The launch identity and EC2/EBS grant workflow must be allowed; sharing the snapshot alone is insufficient |
| Auto Scaling encrypted EBS launch | The ASG **service-linked role** needs key use and permission to create an AWS-resource grant |

For an ASG service-linked role, the key policy normally contains one statement for
`kms:Encrypt`, `kms:Decrypt`, `kms:ReEncrypt*`, `kms:GenerateDataKey*`, and `kms:DescribeKey`, and a
separate `kms:CreateGrant`, `kms:ListGrants`, and `kms:RevokeGrant` statement constrained by:

```json
"Condition": {
  "Bool": {"kms:GrantIsForAWSResource": true}
}
```

Name the exact service-linked role ARN, including a custom suffix if the ASG uses one. Do not add a
broad service principal because “Auto Scaling needs KMS.” In a cross-account key design, follow
the service's documented grant process; some integrations support an external key and others
require a key in the resource's account/Region.

---

## 9. Multi-Region Keys, Rotation, and Recovery

A multi-Region key is not one global authorization object. The primary and replicas share key ID,
key material, and rotation state, but each replica has a separate Regional ARN, policy, grants,
aliases, and tags. A grant on the primary does not work in the recovery Region. The AWS service
must support the multi-Region/cross-Region flow, and the data itself still needs replication.

For a Regional recovery:

1. Pre-create the replica and its local key/resource/IAM policies and service grants.
2. Configure the workload with the **local replica ARN**, not a cross-Region KMS endpoint.
3. Replicate or restore the ciphertext and verify its encryption context is unchanged/expected.
4. Test decrypt under the recovery application role and under the actual service integration.
5. Control `ReplicateKey`/`UpdatePrimaryRegion`; promotion changes administration, not whether a
   healthy replica can perform cryptographic operations.

Automatic rotation of an `AWS_KMS`-origin symmetric customer-managed key retains old backing
material and keeps the ARN, so old ciphertext still decrypts. It is not a reason to delete the old
key after a manual alias migration. Keep the old key enabled until every dependent object,
snapshot, backup, message, and application blob has been re-encrypted or expired.

Before deletion, disable the key and observe use/failures, inventory dependent ciphertext, test
restores, and then use the 7–30 day pending-deletion window. Cancel during that window if a missed
dependency appears; after deletion the key is unrecoverable. A multi-Region primary cannot finish
deletion while replicas remain. Imported material, CloudHSM custom stores, and external key stores
also require a separately tested material/store availability and backup plan.

### Audit evidence

Keep organization CloudTrail KMS management events in a protected archive and do not exclude KMS
events without an explicit cost/evidence decision. Preserve both caller-account and key-owner
events for cross-account use. Alert on `PutKeyPolicy`, `CreateGrant`, `RevokeGrant`, `DisableKey`,
`ScheduleKeyDeletion`, imported-material changes, and multi-Region replication/promotion. The event
should let an investigator identify principal, key ARN, account/Region, `viaService`, source
resource, encryption context, grant ID, and success/error code. Because context is plaintext,
never put sensitive content in it.

---

## 10. Troubleshooting Access Denied

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `AccessDenied` on `kms:Decrypt` despite an IAM policy allowing it | Key policy has no statement delegating to IAM | Add the `root`/IAM-enable statement *or* name the principal in the key policy |
| Cross-account role can't use the key | Cross-account needs **both** the key policy (grant the external account) **and** the caller's IAM policy | Add the external account/role to the key policy **and** allow `kms:*` actions in their IAM |
| Can't share an encrypted EBS/RDS snapshot with another account | It's encrypted with the **default `aws/ebs` / `aws/rds`** key (can't be shared) | Re-encrypt the snapshot with a **customer-managed key**, then share the key + snapshot |
| "Can't encrypt existing RDS / EBS in place" | Encryption must be set at creation | snapshot → **copy with `--kms-key-id`** → restore/create new |
| S3 KMS bill exploding | One KMS call per object | Enable **S3 Bucket Key** |
| `KMS.DisabledException` / scheduled deletion | KMS key disabled or pending deletion | Re-enable; you **cannot decrypt** data while the key is disabled/deleted |

Use this order instead of repeatedly adding `kms:*`:

1. Confirm the ciphertext/resource uses the expected key ARN and Region and the key is enabled.
2. Inspect the caller's IAM policy, permissions boundary, session policy, and applicable SCP for
   the exact KMS and data-resource actions.
3. Inspect the **key policy** for that principal/account. Cross-account IAM cannot grant access the
   owner key policy did not allow.
4. Inspect the S3/snapshot/secret/resource policy and any VPC endpoint policy.
5. Check whether the call is direct or through a service. A direct `aws kms decrypt` test will fail
   a deliberate `kms:ViaService` condition even when S3 `GetObject` is correctly authorized.
6. Check encryption-context spelling/value, grant constraints, service role or service-linked role,
   and whether a just-created grant needs its token.
7. Correlate CloudTrail in both accounts. An IAM `AccessDenied` before KMS is called may leave no
   KMS event; a KMS denial identifies the evaluated key/request context.

Use IAM Policy Simulator/Access Analyzer to find identity/resource-policy mistakes, but validate
with the real service integration because simulators cannot reproduce every service grant and
encryption-context path.

⚠️ **Cross-account KMS = key policy AND IAM**, always both. This is the single most-tested KMS
gotcha. ✅ To share an encrypted snapshot, it must use a **customer-managed KMS key** (default
service keys are non-shareable), and you must share the KMS key with the target account too.

---

## Key Exam Points

- **Envelope encryption**: the KMS key encrypts a **data key**; the data key encrypts the data. The KMS key
  never leaves KMS. `GenerateDataKey` returns plaintext + encrypted copies.
- **AWS-managed KMS keys** (`aws/ebs`, `aws/s3`, `aws/rds`) have non-editable policies and are
  **not shareable**. **Customer-managed keys** give you policy, rotation, grants, and supported
  cross-account sharing.
- **EBS and RDS can't be encrypted in place** — snapshot → copy with a customer-managed key → restore.
- **RDS encryption must be enabled at creation** and covers backups, replicas, snapshots.
- **S3**: SSE-KMS for auditable, controllable encryption; add **Bucket Key** to cut KMS cost;
  SSE-S3 for zero-management; SSE-C when you hold the key.
- **Key policy is primary**; IAM only works if the key policy delegates (the `root`
  statement). **Cross-account requires key policy AND IAM**.
- Cross-account encrypted data also needs the data resource policy/share permission. Constrain
  service use with `kms:ViaService` and encryption context; authorize exact service roles and
  service-linked-role grant creation where required.
- Multi-Region replicas share material/rotation but not policies, grants, aliases, tags, or ARNs.
  Pre-test the local role and actual service path.
- Disable/inventory/test before deletion, and preserve caller/key-owner CloudTrail evidence for
  access and policy/grant changes.
- **Secrets Manager** = KMS-backed secrets **with rotation**; **Parameter Store SecureString**
  = KMS-backed but no built-in rotation.

---

**Next**: [19_disaster_recovery_strategies.md — The Four DR Strategies Compared](19_disaster_recovery_strategies.md)
