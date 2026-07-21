# S3 Advanced Features — Performance, Analytics & Access

> **Who this is for**: Engineers who already know the S3 object model
> ([03_s3_fundamentals.md](03_s3_fundamentals.md)) and the storage classes, lifecycle, and security
> features ([04_s3_storage_classes_and_management.md](04_s3_storage_classes_and_management.md)), and now
> need the remaining exam-tested S3 features: how to make S3 **fast**, how to **analyze** usage to drive
> cost decisions, how to run **bulk operations**, and the access-layer features (**CORS, access logs,
> Access Points**, plus legacy recognition for **S3 Select** and **Object Lambda**). These show up as
> standalone SAA-C03 questions even though each is small.

---

## 1. S3 Performance

S3 scales automatically, but you get the most throughput by parallelizing and by understanding the
per-prefix limits.

**Request rate (per prefix):**

- **≥ 3,500** PUT / COPY / POST / DELETE per second, **per prefix**.
- **≥ 5,500** GET / HEAD per second, **per prefix**.
- There is **no limit to the number of prefixes** — spread keys across many prefixes to multiply
  throughput. Four prefixes read in parallel ≈ 22,000 GET/s.

```
bucket/
├─ folder1/...  ─► 5,500 GET/s
├─ folder2/...  ─► 5,500 GET/s   total ≈ 22,000 GET/s
├─ folder3/...  ─► 5,500 GET/s
└─ folder4/...  ─► 5,500 GET/s
```

**Multipart upload** — split a large object into parts, upload them **in parallel**, S3 reassembles:

- **Recommended for files > 100 MB**, **required for files > 5 GB** (50 TB max object in standard Regions).
- Improves throughput and resilience (retry a single failed part, not the whole upload).
- Pair with a **lifecycle rule to abort incomplete multipart uploads** (orphaned parts cost money).

**Byte-range fetches** — request specific byte ranges of an object:

- **Parallelize downloads** by fetching ranges concurrently (speed up large GETs).
- **Retrieve only part** of an object (e.g., just the file header) without downloading the whole thing.

**S3 Transfer Acceleration** — route uploads through the nearest **CloudFront edge**, then over AWS's
private backbone to the bucket. Best for **long-distance** transfers of large objects. (Covered also in
[04 §8](04_s3_storage_classes_and_management.md).) Compatible with multipart upload.

💡 Exam cue: "users worldwide upload large files, slow over the public internet" → **Transfer
Acceleration** (+ multipart). "Speed up downloading a large object" → **byte-range fetches**.

---

## 2. S3 Storage Class Analysis (S3 Analytics)

**Storage Class Analysis** observes access patterns and **recommends when to transition objects from
Standard → Standard-IA**. It's how you decide the right *N days* for a lifecycle transition instead of
guessing.

- Produces a daily-updated report (CSV export available); recommendations take **24–48 h to start**.
- **Standard and Standard-IA only** — it does **not** give recommendations for One Zone-IA or Glacier.
- Feeds directly into building a **lifecycle policy** (see [04 §2](04_s3_storage_classes_and_management.md)).

✅ "How do I know the right day to move objects to Infrequent Access?" → **S3 Analytics / Storage Class
Analysis**.

---

## 3. S3 Storage Lens

**S3 Storage Lens** gives **organization-wide visibility** into object storage usage and activity —
across **multiple accounts, Regions, and buckets** (via AWS Organizations). It's the only S3 tool that
aggregates the whole org in one dashboard.

- **Default dashboard** is pre-configured; you can build custom ones scoped to accounts/Regions/buckets.
- Surfaces optimization opportunities: buckets with no lifecycle rules, incomplete multipart uploads,
  non-current version bloat, encryption/versioning status, cost-efficiency metrics.

| Tier | Cost | Metrics | Retention | Granularity |
|------|------|---------|-----------|-------------|
| **Free** | Free | Aggregated usage, cost-optimization, data-protection, access-management, performance, and event metrics | 14 days | Bucket-level |
| **Advanced** | Paid | Usage + **activity** metrics, CloudWatch publishing, **prefix-level** aggregation | 15 months | Down to prefix |

### Organization dashboard operating model

1. In the Organizations management account, enable trusted access for S3 Storage Lens.
2. Register a storage/security member account as the **delegated administrator**. That account can
   create organization dashboards without making day-to-day operators use the management account.
3. Create configurations that include the organization and narrow them by OU, account, Region,
   bucket, or prefix where different owners need different views.
4. Export daily metrics to a controlled S3 bucket for long-term Athena/BI analysis. Use Advanced
   metrics and CloudWatch publishing only where activity alerts or prefix-level investigation justify
   the additional charge.

Storage Lens is an aggregate, daily governance view. It can find fleet-wide version bloat, incomplete
multipart uploads, missing encryption/versioning, and rapid growth; it is not a per-request audit log
or a real-time incident detector. Use CloudTrail data events, server access logs, and CloudWatch S3
request metrics for those jobs.

💡 Don't confuse the three "analysis" tools: **Storage Class Analysis** = transition recommendations for
*one bucket*; **Storage Lens** = *org-wide* usage/activity dashboards; **S3 Inventory** = a flat *list*
of objects + metadata (feeds Batch Operations, below).

---

## 4. S3 Batch Operations

**S3 Batch Operations** performs a single action on **billions of existing objects** with one request —
S3 handles retries, progress tracking, and a completion report.

Supported operations:

- **Copy** objects between buckets.
- **Set object tags** or **ACLs**.
- **Restore** objects from Glacier Flexible / Deep Archive.
- **Invoke a Lambda function** to run custom logic on each object.
- Apply **Object Lock** retention or legal hold.

How it works:

```
S3 Inventory (or your CSV)  ─►  manifest (list of objects)
            │
            ▼
  S3 Batch Operations job  ──►  applies the action to every object  ──►  completion report
```

- The object list comes from an **S3 Inventory** report or a CSV manifest you supply.
- Use **Athena** to filter the inventory first, then feed the result as the manifest.
- **S3 Select** can filter inside one object, but it is **not available to new customers**. Treat it
  as a legacy/existing-account feature, not the modern answer for new designs.

✅ "Encrypt / re-tag / copy millions of *existing* objects in place" → **S3 Batch Operations** (lifecycle
and replication only act on *new/changed* objects).

### Inventory-to-remediation workflow

Use this pattern when a Storage Lens dashboard or compliance control identifies a large existing
population that violates policy:

1. Configure **S3 Inventory** daily or weekly with the needed fields—for example version ID,
   encryption status, replication status, storage class, tags, and Object Lock retention. Inventory
   is a scheduled alternative to synchronous `LIST` calls and does not consume the bucket's request
   rate.
2. Deliver CSV, ORC, or Parquet reports to a dedicated, encrypted inventory bucket. For a
   cross-account or SSE-KMS destination, grant the S3 service access in both the destination bucket
   and customer-managed KMS key policies.
3. Query the report with Athena to select only noncompliant objects and versions. Preserve version
   IDs in the manifest when remediation must target a specific version.
4. Run a small **Batch Operations** canary using a least-privilege job role, review the completion
   report, then expand the job. Batch Operations invokes the underlying API, so its permissions,
   KMS requirements, retention restrictions, request charges, and failure modes still apply.
5. Re-run Inventory or the compliance control to prove the desired state rather than treating a
   submitted job as success.

Use built-in operations for copy, restore, retention/legal hold, or replacing tags. Use Lambda only
when the transformation is genuinely custom. The tag operation **replaces** the tag set rather than
merging with existing tags, so include tags that must be retained.

---

## 5. S3 CORS (Cross-Origin Resource Sharing)

An **origin** is a scheme + host + port (e.g., `https://www.example.com`). **CORS** is a **browser**
security control: a web page loaded from origin A may only request resources from origin B if B
explicitly allows it via CORS headers.

If a static site hosted in one S3 bucket (or any web origin) fetches files (fonts, images, JSON) from a
**different** S3 bucket, that second bucket must have a **CORS configuration** allowing the first origin.

```
Browser (origin A)  ──preflight OPTIONS──►  S3 bucket (origin B)
                    ◄─Access-Control-Allow-Origin: A─
                    ──actual GET──────────►
```

- Configure CORS rules on the bucket: **AllowedOrigins**, **AllowedMethods** (GET/PUT/…), **AllowedHeaders**.
- The browser sends a **preflight `OPTIONS`** request; S3 answers with the allowed origin/methods.

✅ Exam cue: "web app gets a **CORS error** loading assets from an S3 bucket" → add a **CORS
configuration** on the bucket being requested (the *destination* origin), allowing the requesting origin.

---

## 6. S3 Server Access Logs

**Server access logging** records **every request** made to a bucket (requester, bucket, time, action,
response, error code) and delivers log files to a **target bucket** for audit and analysis.

- The **target (logging) bucket must be in the same Region** as the source bucket.
- Logs can be analyzed with **Amazon Athena**.

⚠️ **Never set the logging bucket to be the same as (or logged by) the monitored bucket** — writing logs
generates more requests, which generate more logs → an **exponential logging loop** that grows storage
(and cost) without bound.

💡 For richer, near-real-time API-level logging of *who did what*, **CloudTrail data events** are the
alternative; server access logs are best-effort, lower-cost, request-level logs.

---

## 7. S3 Access Points

**Access Points** simplify access management for **shared datasets** in a large bucket. Instead of one
sprawling, hard-to-maintain bucket policy, you create **one access point per application/team**, each
with its **own DNS name** and **own access point policy** scoped to a prefix.

```
                    ┌─ Finance Access Point  (policy: R/W on /finance) ─► Finance app
S3 bucket ──────────┼─ Sales   Access Point  (policy: R   on /sales)   ─► Sales app
(prefixes inside)   └─ Analytics AP (VPC-only) (policy: R on /logs)     ─► VPC workload
```

- Each access point has a **network origin**: **internet** or **VPC-only** (reachable solely through a
  **VPC endpoint** — the access point policy plus an endpoint policy lock it to the VPC).
- Access is granted by the **access point policy** (and the bucket policy can delegate to access points).
- Scales access management horizontally — add an access point per consumer instead of editing one policy.

✅ "Many applications share one bucket and managing a single bucket policy has become unwieldy" →
**S3 Access Points** (one policy per app). "Restrict access to a dataset to only resources inside a VPC"
→ **VPC-only access point**.

---

## 8. S3 Access Grants and Private Connectivity

### Access Points vs Access Grants

Access Points create stable application endpoints and policies. **S3 Access Grants** solves a
different problem: granting many IAM or corporate-directory identities scoped access to buckets,
prefixes, or objects without creating a long-lived IAM role per person.

```
IAM / IAM Identity Center identity
              │ requests access
              ▼
       S3 Access Grants ──assumes registered-location role──► temporary scoped credentials
                                                                    │
                                                                    ▼
                                                            bucket / prefix / object
```

An Access Grants instance registers S3 locations and the IAM role that can reach each location. A
grant maps an IAM principal—or a user/group from IAM Identity Center—to a location and permission;
S3 Access Grants vends temporary credentials when access is authorized. It also supports
cross-account grants when both the Access Grants resource policy and the consumer's IAM policy allow
the request.

Use Access Grants for dynamic, identity-centric data access such as analytics users and corporate
groups. Use Access Points for application/network boundaries with stable resource policies. For
cataloged data-lake tables that require table/column permissions, use **Lake Formation** rather than
treating path-level S3 grants as a table authorization system.

### Network and cost choices

| Access path | Security boundary | Cost/constraint | Choose when |
|-------------|-------------------|-----------------|-------------|
| Regional S3 endpoint | IAM/bucket/access-point policies and TLS | No VPC endpoint hourly charge; clients use S3's service endpoint | Public IP addressing is acceptable and policy controls meet the requirement |
| **Gateway VPC endpoint** | Route tables + endpoint policy + S3 resource policies | No additional endpoint charge; same-Region VPC access only, not reachable from on-premises, peered VPCs, or Transit Gateway | Workloads inside a VPC need private S3 routing at the lowest cost |
| **Interface VPC endpoint** (PrivateLink) | Private endpoint ENIs/security groups + endpoint and S3 policies | Hourly per-AZ and data-processing charges; supports private IP access from on-premises over VPN/Direct Connect and other supported connected networks | The client needs private-IP connectivity that a gateway endpoint cannot provide |
| **VPC-only Access Point** | Access-point policy requires requests through a VPC endpoint | Access point simplifies policy but does not remove endpoint/transfer/request charges | A shared bucket needs a consumer-specific endpoint and VPC network boundary |

For hybrid access, use an S3 interface endpoint only for traffic that needs it. Where both are
required, route in-VPC traffic through the free gateway endpoint and on-premises traffic through the
paid interface endpoint. Private connectivity does not grant data access: the identity policy,
access-point/bucket policy, endpoint policy, KMS key policy, and organization guardrails must still
allow the request, and an explicit deny in any layer wins.

---

## 9. S3 Object Lambda (Legacy / Existing Customers)

**S3 Object Lambda** uses a **Lambda function to transform an object as it's returned to the caller** —
the source object in S3 is never duplicated or modified. It's built **on top of an S3 Access Point**.

⚠️ **Current availability**: as of **November 7, 2025**, S3 Object Lambda is available only to
existing customers currently using it and select APN partners. AWS says existing users can continue,
but new designs should treat it as a legacy option.

```
Caller ─► S3 Object Lambda Access Point ─► Lambda (transforms) ─► S3 Access Point ─► bucket
                                            │
                                            └─ redact / convert / resize / enrich on the fly
```

Existing-customer use cases:

- **Redact PII** from objects for a non-production/analytics audience.
- **Convert formats** on the fly (e.g., XML ↔ JSON, or CSV ↔ Parquet for one consumer).
- **Resize / watermark images**, or **enrich** data from another source — all without storing a second copy.

✅ "Different consumers need different *views* of the same object without keeping multiple copies" →
**S3 Object Lambda** in older exam material. In a current greenfield design, prefer a clearer pattern:
CloudFront/Lambda-based transformation for web objects, API Gateway or Function URLs in front of
Lambda, client-side transformation, or a materialized transformed copy when that is simpler.

---

## Key Exam Points

- **Performance**: ≥3,500 write / ≥5,500 read requests **per prefix**, unlimited prefixes → spread keys
  to scale. **Multipart** recommended >100 MB, required >5 GB. **Byte-range fetches** parallelize/partial
  GETs. **Transfer Acceleration** for long-distance uploads.
- **Storage Class Analysis** recommends Standard → Standard-IA transition timing (Standard/Standard-IA
  only). **Storage Lens** = org-wide usage/activity dashboards. **S3 Inventory** = object list/metadata.
- Enable Storage Lens trusted access and use a **delegated administrator** for organization
  dashboards; export metrics for long-term analysis rather than operating from the management account.
- **Batch Operations** acts on **existing** objects (copy, tag, ACL, restore, invoke Lambda) using an
  Inventory/CSV manifest — the answer when lifecycle/replication (new-objects-only) won't do.
- **S3 Select** and **S3 Object Lambda** are legacy/existing-customer features for new AWS accounts:
  recognize them in older material, but avoid choosing them for greenfield architecture unless the
  question explicitly says they are already in use.
- **CORS**: configure on the **destination** bucket, allow the requesting **origin**; fixes browser CORS
  errors loading cross-origin assets.
- **Access logs**: target bucket same Region, **never** the monitored bucket (logging loop).
- **Access Points**: per-app policies + DNS names; can be **VPC-only**.
- **S3 Access Grants** vends temporary, prefix/object-scoped credentials to IAM or IAM Identity
  Center identities. It complements Access Points; Lake Formation remains the data-table permission layer.
- S3 gateway endpoints are private and have no added endpoint charge; interface endpoints support
  hybrid/private-IP access but add endpoint-hour and data-processing charges.

---

## Common Mistakes

- ❌ Using one prefix for a high-throughput workload, then hitting request limits — spread across prefixes.
- ❌ Forgetting multipart is **required** above 5 GB, or leaving incomplete parts to accrue cost.
- ❌ Expecting lifecycle or replication to transform/encrypt **existing** objects — that's **Batch Operations**.
- ❌ Confusing Storage Class **Analysis** (one bucket, transition advice) with **Storage Lens** (org-wide).
- ❌ Treating a Storage Lens organization dashboard as real-time auditing. It is aggregate governance
  data; use CloudTrail/logs/request metrics for request-level or operational detection.
- ❌ Running a Batch Operations job across billions of objects without version-aware Inventory,
  canary scope, least-privilege/KMS permissions, and a completion report.
- ❌ Adding CORS rules to the *requesting* origin instead of the **bucket being requested**.
- ❌ Pointing server access logs at the same bucket they monitor → runaway logging loop.
- ❌ Picking S3 Select or Object Lambda as a greenfield default. They are legacy/existing-customer
  features now; use Athena, CloudFront/Lambda, API Gateway/Lambda, or a transformed copy instead.
- ❌ Paying interface-endpoint processing charges for all in-VPC S3 traffic when a gateway endpoint
  can carry it, or assuming a private endpoint grants authorization by itself.

---

**Next**: [06_storage_gateway_and_transfer.md — Storage Gateway, DataSync, Transfer Family & Snow](06_storage_gateway_and_transfer.md)
