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
  encrypted `SecureString` values, generally at lower cost, but with **no built-in rotation**.

Both encrypt sensitive values with **KMS** and gate access with **IAM**.

---

## 2. Secrets Manager vs Parameter Store

| Feature | **Secrets Manager** | **SSM Parameter Store** |
|---------|---------------------|--------------------------|
| Primary purpose | Storing & **rotating** secrets | Config + parameters (secrets via `SecureString`) |
| **Automatic rotation** | ✅ Built in; service-managed for supported secrets or through **Lambda** | ❌ None (you'd build it yourself) |
| Database rotation | ✅ Managed strategies/templates for supported databases | ❌ |
| Encryption | KMS (always) | `SecureString` type uses KMS; `String` is plaintext |
| Cost | Per secret and per API call | Standard parameters have no additional storage charge; advanced parameters and higher throughput are paid |
| Versioning | ✅ Staged versions (`AWSCURRENT` / `AWSPREVIOUS`) | ✅ Version history per parameter |
| Cross-account / cross-region | ✅ Resource policy + replica secrets | Limited |
| Size limit | 64 KB | 4 KB standard / 8 KB advanced |
| Hierarchy / path queries | Flat (with tags) | ✅ Path hierarchy (`/app/prod/db/password`) `GetParametersByPath` |
| Generate random secret | ✅ `GetRandomPassword` | ❌ |

> **Rule of thumb for the exam**:
> - Needs **automatic rotation** of a credential (especially **RDS**) → **Secrets Manager**.
> - Just need to store config / a secret cheaply with **no rotation requirement** → **Parameter
>   Store `SecureString`** (usually lower cost).
> - "Most cost-effective way to store a secret" with no rotation mentioned → **Parameter Store**.
> - "Automatically rotate the DB password every 30 days" → **Secrets Manager**.

💡 Parameter Store can actually *reference* a Secrets Manager secret, so they compose — but for
the exam, treat the picker above as the decision.

---

## 3. Secrets Manager Deep Dive

**Automatic rotation** is the headline feature. Supported services can manage rotation directly;
other targets use a **Lambda rotation function** that Secrets Manager invokes on a schedule to
generate a new value and update both the secret and the target. The diagram shows the Lambda
path:

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

- Some supported database integrations can use managed rotation. Lambda-based rotation uses an
  AWS-provided template or your own function when the target needs custom logic.
- **Versioning via staging labels**: the new value is `AWSPENDING`, promoted to `AWSCURRENT` on
  success, and the old one becomes `AWSPREVIOUS`. These labels support diagnosis and recovery;
  they are not permission to promote an untested value blindly.
- Encrypted with KMS; access controlled by IAM identity policies **and** an optional resource
  policy for cross-account sharing.

✅ Best fit: production database credentials that compliance requires rotating, or any secret
where a leaked-then-rotated credential should auto-heal.

---

## 4. Production Secrets Architecture

A secret is a dependency with a lifecycle, not just an encrypted string. A production design
must decide who owns it, which applications may retrieve it, how it changes without downtime,
and how a secondary Region becomes independent during a failure.

### Multi-account ownership and authorization

Prefer a secret in the **workload account and Region that consumes it**. This keeps the KMS key,
rotation target, VPC connectivity, quota, and incident blast radius with the workload. Centralize
standards, monitoring, and inventory; centralize the secret itself only when several accounts
must intentionally consume the same credential or a security team must own its lifecycle.

Cross-account access has three layers:

1. A Secrets Manager **resource policy** allows a specific external role.
2. The caller's identity policy allows `secretsmanager:GetSecretValue` on that ARN.
3. If the secret uses a customer-managed KMS key, its key policy also permits the caller to
   decrypt through Secrets Manager. The AWS-managed `aws/secretsmanager` key cannot satisfy a
   cross-account design.

Use `PutResourcePolicy` with `BlockPublicPolicy: true` in deployment automation. Secrets Manager
uses automated policy analysis to reject a resource policy that grants broad public access. This
check evaluates the **secret resource policy**, not the caller's IAM policy or the KMS key policy,
so review all three. Grant named roles, not `Principal: "*"`, and constrain organization/account
boundaries where appropriate.

For a centrally shared secret, consumers become coupled to the same value and rotation event.
That is useful for a genuinely shared upstream credential, but it is a poor substitute for
separate application identities. Prefer one database user/API credential per workload where the
target supports it; attribution and revocation are then much cleaner.

### Replica secrets and Regional failover

Secrets Manager can replicate a primary secret to selected Regions. A replica receives encrypted
secret versions and metadata such as tags, resource policies, and rotation updates, but it uses a
KMS key in the replica Region and has a Region-specific ARN. The application should resolve the
local Regional ARN through configuration rather than hard-coding the primary endpoint.

Replication is not application failover by itself:

- Fields such as a database host remain whatever value the primary secret contains unless the
  design deliberately stores a Region-specific endpoint or discovers it separately.
- The replica's resource policy, the local KMS key, private endpoint, application role, and
  target service must all work in the recovery Region.
- Rotation occurs on the primary and propagates. During an extended primary-Region incident,
  use `StopReplicationToReplica` to promote the replica to an independent secret, then configure
  its own rotation and any new replicas. Promotion stops future synchronization; failback is a
  planned reconciliation, not an automatic merge.

Test both normal replication and promotion. Use `DescribeSecret`, CloudTrail, and replica status
to alarm on a stalled replica. Existing replicated versions may remain readable even while a new
version fails to propagate, so a green application health check alone does not prove replication
is current.

### Rotation without an outage

The four Lambda rotation steps are `createSecret`, `setSecret`, `testSecret`, and `finishSecret`.
They should be idempotent because Secrets Manager can retry a step. The function needs network
access to both Secrets Manager and the target database/service, permission to use the secret and
KMS key, and a secret JSON schema that matches its rotation code.

When rotation fails:

1. Inspect the rotation Lambda's CloudWatch Logs and the secret's staging labels.
2. Check timeout, security groups/NACLs, DNS, endpoint policies, KMS permissions, target
   authentication, and the secret's JSON fields.
3. Verify both `AWSCURRENT` and, when needed for recovery, `AWSPREVIOUS` directly against the
   target. Do not blindly promote a value that was never set or tested.
4. Fix the root cause and retry the documented rotation flow. Remove an orphaned `AWSPENDING`
   label only when the error state and recovery procedure call for it.

Avoid a single-user strategy if changing the current password breaks long-lived clients before
they refresh. Where supported, alternating users allow one identity to remain valid while the
other rotates. Applications must tolerate credential refresh and retry authentication after
fetching a fresh version.

### Retrieval, caching, and private networking

Fetch secrets at runtime with the AWS SDK or an AWS caching library and keep the plaintext only
in process memory. A cache lowers latency, API cost, and throttling risk; choose a refresh period
shorter than the rotation interval, add jitter, and invalidate/refetch after an authentication
failure. Caching does not weaken IAM on the initial fetch, but the application now owns protection
of that plaintext in memory and logs.

Private workloads can reach Secrets Manager without an internet gateway or NAT gateway through
an **interface VPC endpoint** (`com.amazonaws.<region>.secretsmanager`) with private DNS. Scope the
endpoint policy, security groups, secret policy, IAM policy, and KMS policy together. A rotation
Lambda in a VPC also needs this path (or another approved egress path) plus connectivity to its
database; creating the endpoint alone does not give it database network access.

---

## 5. ACM — AWS Certificate Manager

**TLS termination recap**: a client connects over HTTPS; somewhere the encrypted connection is
*terminated* (decrypted) and the server presents an **X.509 certificate** proving its identity.
That certificate must be issued by a trusted CA and kept renewed. ACM manages that lifecycle.

```
  client ──HTTPS (TLS)──► [ ALB / CloudFront / API GW ]  ──HTTP──► backend
                          ▲ presents ACM certificate          (re-encrypt
                          │ (TLS terminated here)              optional)
```

What ACM gives you:

- **Public certificates for integrated AWS services** — non-exportable public certificates used
  with supported integrations do not have a certificate charge. Exportable public certificates
  are a paid option.
- **Managed renewal for eligible ACM-issued certificates** — ACM revalidates and renews eligible
  public and private certificates. An exportable certificate remains eligible after it has been
  exported, but ACM cannot redeploy it to your server: you must detect renewal, export the new
  certificate/private key, and install them everywhere that uses them. Re-exporting after each
  renewal also keeps the exportable certificate eligible for its next managed renewal.
- **Domain validation** via DNS (Route 53 — preferred, hands-off) or email.

Certificate lifecycle depends on how the certificate entered ACM:

| Certificate | Private key export | Renewal responsibility |
|-------------|--------------------|------------------------|
| ACM public, non-exportable | ❌ | ACM manages renewal while the certificate remains eligible and integrated |
| ACM public, requested with export enabled | ✅, encrypted with a passphrase at export | ACM can renew it; **you** re-export and redeploy each renewed version |
| ACM-requested private certificate | ✅ | ACM can manage renewal when eligible; exported deployments still need re-export/redeploy automation |
| Imported public/private certificate | It was supplied by you | **You** renew and reimport it; ACM does not renew imported certificates |

Export eligibility is selected when requesting a public certificate and cannot be turned on for
an existing non-exportable certificate. For an on-instance workload, either terminate TLS on an
integrated service such as an ALB, or request an exportable certificate and build secure delivery
and redeployment of its private key. Never put the export passphrase or private key in logs,
source control, or an AMI.

### Where ACM certs attach

| Integration | Notes |
|-------------|-------|
| **Elastic Load Balancing (ALB/NLB)** | **Regional** cert — must be in the **same region** as the load balancer |
| **CloudFront** | Cert **must be in `us-east-1` (N. Virginia)** — global edge service |
| **API Gateway** | Edge-optimized endpoints → `us-east-1`; regional endpoints → that region |
| **EC2 / containers / on premises** | ACM cannot attach a managed certificate directly; use an **exportable** ACM-issued certificate or import one, then automate private-key delivery and renewed-certificate deployment |

> **Rule**: For **CloudFront**, the ACM certificate **must live in `us-east-1`**, regardless of
> where your origin is. This is one of the most frequently tested ACM facts. For **ALB**, the
> cert must be in the **same region** as the ALB.

ACM certificates are **Regional and account-bound resources**. A certificate ARN in one account
cannot simply be attached to an ALB in another account, and the certificate attached to a
Regional service must exist in that service's Region. Request/issue a certificate in every
required account and Region, or deliberately export and deploy it yourself. CloudFront is the
special case: its certificate must be in `us-east-1` in the distribution's account.

### AWS Private CA architecture

**AWS Private CA** is a separate paid service for private trust: internal TLS, mutual TLS, device
identity, and code-signing use cases whose certificates do not need public-browser trust. The
hard part is designing the trust hierarchy and protecting its issuing authority.

A typical organization uses:

```text
Dedicated PKI/security account
└── Offline or rarely used root CA
    ├── Production subordinate issuing CA
    │   ├── workload-account ACM private certificate
    │   └── workload-account ACM private certificate
    └── Non-production subordinate issuing CA
```

- Keep the root CA in a dedicated security account, minimize its use, and give root
  administration to a small role separate from certificate issuers. Use subordinate issuing CAs
  to contain compromise and policy differences. Private CA supports hierarchies up to five
  levels, but fewer levels are easier to operate and recover.
- Set CA path-length, validity, name constraints/templates, and issuance permissions so a
  workload cannot mint a new CA or arbitrary identities. Publish and monitor CRLs and/or OCSP as
  required by relying clients; a private certificate is not safely revocable just because its
  IAM permission was removed.
- Share a CA to organization accounts or OUs with **AWS RAM** and a resource-based permission.
  Consumer accounts can then request local ACM private certificates while the CA remains under
  central control. Issuance authorization and certificate lifecycle in the consuming account are
  separate controls.
- Private CAs are Regional. Build subordinate issuing CAs in the Regions that require local
  issuance/availability, and distribute the correct trust chain. Do not assume ACM or Private CA
  replicates a hierarchy across Regions.
- Prefer ACM-managed deployment where supported. Certificates issued directly with Private CA's
  `IssueCertificate` API are not ACM-managed and need your renewal/deployment automation.

Design recovery before issuing production identities: protect CA backups and configuration,
retain the chain, document replacement and revocation procedures, monitor issuance via
CloudTrail, and alert on CA state/policy changes. A compromised issuing CA may require replacing
every certificate below it, not merely rotating one private key.

---

## Key Exam Points

- **Automatic rotation** of credentials (especially databases) → **Secrets Manager**. Supported
  integrations can use managed strategies; other targets use an idempotent Lambda rotation
  workflow.
- **Cheapest** secret storage with **no rotation** requirement → **Parameter Store
  `SecureString`** is usually the intended answer.
- Both encrypt with **KMS** and authorize with **IAM**.
- Cross-account secret access needs the secret resource policy, caller IAM policy, and a usable
  customer-managed KMS key policy. Use `BlockPublicPolicy` to reject broad resource policies.
- Replica secrets copy versions but do not fail over the database or application. Promotion
  makes a replica independent and requires a planned reconciliation later.
- **ACM** manages eligible certificate renewal. Exportable certificates can be installed on your
  own servers, but renewed versions must be re-exported and redeployed; imported certificates are
  not renewed by ACM.
- **CloudFront certs must be in `us-east-1`**; ALB/NLB certs must be in the **same region** as
  the LB.
- Certificates are Regional and account-bound. Issue/request in each deployment account and
  Region, or own the security and automation of exporting and distributing them.
- A Private CA hierarchy normally has a protected root and constrained subordinate issuing CAs;
  AWS RAM can share a CA while consumers request local ACM private certificates.
- Parameter Store offers a **path hierarchy** and `GetParametersByPath`; Secrets Manager offers
  staged versions (`AWSCURRENT`/`AWSPENDING`/`AWSPREVIOUS`).

---

## Common Mistakes

- ❌ Choosing Secrets Manager when no rotation is required and cost is the deciding factor — the
  lower-cost **Parameter Store** is usually the intended answer.
- ❌ Putting a CloudFront ACM cert in the origin's region instead of **`us-east-1`**.
- ❌ Treating an exported and an imported certificate alike — ACM can renew an eligible
  ACM-issued exportable certificate, but you must re-export/redeploy it; imported certificates
  are not renewed.
- ❌ Storing secrets as a plaintext `String` parameter instead of `SecureString` — `String` is
  not encrypted.
- ❌ Assuming secret replication changes a database endpoint or performs application failover.
- ❌ Sharing a secret resource policy across accounts but forgetting that the KMS key and caller
  IAM policy must also authorize the read.
- ❌ Fetching a secret on every request — cache it with a bounded refresh period and refetch after
  authentication failure.
- ❌ Assuming an ACM certificate ARN can be attached across accounts or Regions.
- ❌ Sharing a root CA broadly instead of issuing through constrained subordinate CAs.
- ❌ Confusing **Secrets Manager rotation** (managed, Lambda-driven) with Parameter Store, which
  has **no** rotation at all.

---

**Next**: [03_threat_detection_services.md — WAF, Shield, GuardDuty, Inspector, Macie, Detective & Security Hub](03_threat_detection_services.md)
