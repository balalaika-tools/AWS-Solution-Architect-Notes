# Amazon S3 — The Object Storage Model

> **Who this is for**: Engineers who know block (EBS) and file (EFS/FSx) storage and now need the
> object-storage foundation. Read [01_ebs.md](01_ebs.md) for the block-vs-file-vs-object table first.
> S3 underpins half of AWS — backups, data lakes, static sites, logs, snapshots — so this file nails
> the model and access basics, and the next file ([04](04_s3_storage_classes_and_management.md))
> covers storage classes, lifecycle, replication, and security features.

---

## 1. What Object Storage Is

**Amazon S3 (Simple Storage Service)** stores data as **objects** in **buckets**, accessed over an
**HTTP REST API** — not as a disk and not as a mounted filesystem.

An **object** is: the **data** (the bytes), a unique **key** (its name within the bucket), and
**metadata** (content type, user tags, system fields). You write a whole object (`PUT`) and read a
whole object (`GET`); there is **no partial in-place edit** — to change one byte you re-upload the
object (a new version).

```
        ┌──────────────── Bucket: "acme-reports" (Region: us-east-1) ──────────────┐
        │  key = "2026/q1/financials.pdf"   →  [ data + metadata ]                 │
        │  key = "2026/q1/summary.csv"      →  [ data + metadata ]                 │
        │  key = "logos/header.png"         →  [ data + metadata ]                 │
        └──────────────────────────────────────────────────────────────────────────┘
   Access:  GET https://acme-reports.s3.us-east-1.amazonaws.com/2026/q1/financials.pdf
```

> **Key insight**: S3 is not a filesystem. There are no real folders — just keys with `/` in them.
> The "folder" you see in the console is a UI convenience over a **flat namespace**.

---

## 2. Buckets, Keys, Prefixes

- **Bucket** — the top-level container, **created in and tied to one Region**. In the traditional
  shared global namespace, its DNS name must be unique across accounts in the AWS partition. AWS
  also supports account regional namespaces in supported Regions, where an account reserves its own
  predictable names. Data never leaves the chosen Region unless you transfer or replicate it.
- **Key** — the full name of an object within the bucket, e.g. `2026/q1/financials.pdf`.
- **Prefix** — the leading part of a key up to a delimiter, e.g. `2026/q1/`. Prefixes are how you
  *organize* and *filter* objects (and how lifecycle/replication rules target subsets). They look
  like folders but are just string prefixes on a flat namespace.

| Concept | Scope | Example |
|---------|-------|---------|
| Bucket | Name governed by the chosen namespace; Region-scoped storage | `acme-reports` |
| Key | Unique within a bucket | `2026/q1/financials.pdf` |
| Prefix | A key namespace for grouping/filtering | `2026/q1/` |

⚠️ Namespace scope applies to the **bucket name**, not the data location. Whether the bucket uses the
shared global namespace or an account regional namespace, its data still lives in the **Region** you
chose. Don't confuse naming with global storage.

---

## 3. Durability, Availability, and Limits

| Property | Value | Meaning |
|----------|-------|---------|
| **Durability** | **11 nines (99.999999999%)** | Across all storage classes (objects redundantly stored across multiple devices/AZs in a Region). Effectively you will not lose an object. |
| **Availability** | ~99.99% (S3 Standard) | Designed availability for retrieval; varies by storage class. |
| **Max object size** | **50 TB** in standard AWS Regions | A single object; AWS GovCloud (US) retains a lower limit. |
| **Single PUT limit** | **5 GB** | Above this you **must** use multipart upload. |
| **Multipart recommended** | **> 100 MB** | AWS recommends multipart for anything over ~100 MB. |
| **Min object size** | 0 bytes | Empty objects are allowed. |
| **General purpose buckets/account** | Default **10,000**, adjustable through Service Quotas | Object count and total bucket size are unlimited; very large bucket quotas can add per-bucket cost. |

> **Rule**: **11 nines of durability** is the number to memorize — it's the headline reliability
> figure and a frequent distractor when other services quote lower numbers. Availability (~99.99%)
> is a *different* metric and is lower.

### Multipart upload

Split a large object into parts, upload them in **parallel** (faster, resilient — retry a failed part
only), then S3 assembles them. Required above 5 GB, recommended above ~100 MB.

💡 Pair multipart with **Transfer Acceleration** (covered in file 04) and a **lifecycle rule to
abort incomplete multipart uploads** so half-finished uploads don't silently accrue storage cost.

---

## 4. Consistency Model

S3 provides **strong read-after-write consistency** for all operations — `PUT`s of new objects,
overwrites (`PUT` of an existing key), and `DELETE`s. A `GET`, `LIST`, or metadata read immediately
reflects the latest write, with **no eventual-consistency window**.

⚠️ **Exam currency check**: older material describes S3 as *eventually consistent* for overwrites and
deletes. That changed in **December 2020** — S3 is now **strongly consistent** by default at no extra
cost. Trust the strong-consistency answer.

---

## 5. Access Control Basics

By default a bucket and its objects are **private** — only the owning account can access them. You
grant access through layered controls:

| Mechanism | Granularity | Use when |
|-----------|-------------|----------|
| **Block Public Access** | Account / bucket master switch | **On by default**; the safety net that overrides any policy/ACL that would expose data publicly |
| **IAM policies** | Identity-based (attached to a user/role) | "*What can this principal do across S3?*" — central, in your account |
| **Bucket policies** | Resource-based (JSON on the bucket) | "*Who can access this bucket?*" — cross-account access, public read for a website, enforcing TLS/encryption |
| **ACLs** (Access Control Lists) | Object/bucket, coarse | **Legacy** — AWS recommends keeping them disabled (Object Ownership = bucket owner enforced) |

```
        Request to s3://acme-reports/key
                │
                ▼
   ┌────────────────────────┐  if blocked here, request is DENIED
   │ Block Public Access    │  (master safety switch)
   └───────────┬────────────┘
               ▼
   ┌────────────────────────┐
   │ IAM + Bucket Policy +  │  evaluated together; explicit DENY always wins
   │ ACL (legacy)           │
   └────────────────────────┘
```

> **Key insight**: **Block Public Access is ON by default** and is the recommended posture. To make
> objects public (e.g., a static website), you must first turn off the relevant Block Public Access
> setting *and* grant read via a **bucket policy**. An explicit **deny** anywhere always wins.

✅ Cross-account or "make this bucket readable by the public/CloudFront" → **bucket policy**.
✅ "Control what my users/roles can do" → **IAM policy**.
❌ Reaching for **ACLs** — AWS now steers you to disable ACLs and use policies; ACLs are legacy.

---

## 6. Organization-Scale Ownership and Governance

A bucket is owned by one AWS account even when producers and consumers live in many accounts. At
scale, decide who owns the data, policies, encryption keys, and bill before granting access.

| Ownership pattern | Use when | Main trade-off |
|-------------------|----------|----------------|
| **Workload-owned buckets** | Teams need independent lifecycle, performance, and cost boundaries | Simple accountability, but organization guardrails and inventory must cover many buckets/accounts |
| **Central data account** | A platform/security team owns a shared lake, archive, or log store | Strong separation and centralized controls, but shared-bucket request cost and consumer access need explicit allocation and policy design |

For a central bucket, grant workload roles or S3 Access Points access to only their prefixes. Keep
the bucket and KMS key administration in the data account; do not solve cross-account writes by
handing every producer broad bucket-administration permissions.

### Object Ownership: make the bucket owner authoritative

Use **S3 Object Ownership — Bucket owner enforced**, the default for new buckets. It disables ACLs,
and the bucket-owning account owns every new and existing object, including objects uploaded by
another account. Access is then governed with IAM, bucket, access point, VPC endpoint, SCP, and RCP
policies instead of per-object ACLs.

An uploader that sends an unsupported ACL header to a bucket-owner-enforced bucket gets an error.
When migrating an older bucket, first translate required ACL grants into policies and test writers
that might still send ACLs.

### Access-point strategy

Create an **S3 Access Point** per stable application or consumer boundary when one shared bucket
policy becomes difficult to reason about. Each access point has its own name, policy, and either an
internet or VPC network origin. Scope it to the consumer's prefix and permissions; use a VPC-only
access point when requests must enter through a VPC endpoint.

The access point policy and underlying bucket policy are both evaluated. An access point does not
hide or disable the bucket's other access paths, so make the bucket policy delegate to approved
access points and deny unintended direct access when that is the design.

### Organization-wide public-access baseline

Turn on all four **Block Public Access** settings at the bucket and account levels. For an AWS
Organization, an **Amazon S3 policy in AWS Organizations** can enforce the complete Block Public
Access configuration at the root, OU, or account level. Member accounts inherit it and cannot weaken
the effective organization setting; S3 applies the most restrictive applicable configuration.

This is a safety baseline, not a substitute for least-privilege policies. Test legitimate public
website or data-distribution exceptions before attaching an organization policy because the
organization-level setting is all four controls together, not a per-setting customization.

### Observe request rate and cost, not just stored bytes

Storage capacity is only one S3 cost and performance dimension. Establish these views:

| Question | Tool | Caveat |
|----------|------|--------|
| Which accounts, buckets, and prefixes are growing or missing protection controls? | **S3 Storage Lens** organization dashboard | Aggregated trend/governance data, not a per-request audit trail |
| Is a hot bucket returning elevated `4xx`, `5xx`, or `503 Slow Down` responses? | Opt-in **CloudWatch S3 request metrics** plus application metrics | Request metrics add monitoring cost; clients still need retries with exponential backoff |
| Which principal accessed an object? | **CloudTrail data events** or S3 server access logs | Data events and log storage/querying add cost; enable them deliberately for required scopes |
| Why did the S3 bill change? | **Cost Explorer/CUR**, split by account, Region, operation/usage type, and allocation tags where applicable | A central shared bucket needs an internal allocation model based on request/log/application context |

Include request charges, data retrieval, lifecycle transitions, inter-Region/internet transfer,
replication, KMS requests, incomplete multipart uploads, and noncurrent versions in the model. A
request-rate problem can also become a cost problem: retry storms and millions of tiny-object calls
may cost more than the stored data.

---

## 7. Bucket vs Object — Quick Contrast

| | **Bucket** | **Object** |
|---|---|---|
| What | Container with a namespace-governed name; Region-scoped | The stored data + key + metadata |
| Limit | Default 10,000 general purpose buckets/account (adjustable) | Up to 50 TB each in standard Regions; unlimited count |
| Policy attach point | Bucket policy, BPA, versioning, replication, lifecycle | Tags, metadata, storage class, version ID |
| Addressed by | DNS name | Key within the bucket |

---

## Key Exam Points

- S3 is **object storage** over an **HTTP API** — not a disk (EBS) and not a mountable filesystem
  (EFS/FSx). No partial edits; you replace whole objects.
- Traditional bucket names are unique in the shared namespace; account regional namespaces provide
  account-reserved names in supported Regions. **Storage remains Region-scoped.**
- **11 nines (99.999999999%) durability**; **max object size 50 TB** in standard Regions;
  **multipart required above 5 GB**,
  recommended above ~100 MB.
- S3 is now **strongly read-after-write consistent** for all operations (since Dec 2020) — ignore old
  "eventually consistent" claims.
- Access layers: **Block Public Access (on by default)** → **IAM + bucket policy + (legacy) ACL**.
  **Bucket policy** for cross-account/public; **IAM policy** for your principals; **explicit deny wins**.
- At organization scale, choose data-account ownership, keep **Bucket owner enforced**, use
  per-consumer **Access Points**, and enforce Block Public Access through AWS Organizations.
- Watch both storage and activity: **Storage Lens** for fleet trends, **CloudWatch request metrics**
  for operational signals, and CloudTrail/logs plus CUR for attribution and cost analysis.

---

## Common Mistakes

- ❌ Treating S3 like a filesystem — there are no real folders, only keys/prefixes in a flat namespace.
- ❌ Thinking a bucket's naming scope means the data is global. The data lives in its selected Region.
- ❌ Quoting S3 as eventually consistent — it's strongly consistent now.
- ❌ Trying to upload a 10 GB object in a single `PUT` — anything over 5 GB **must** use multipart.
- ❌ Using ACLs for cross-account access — use a **bucket policy**; AWS recommends disabling ACLs.
- ❌ Forgetting that **Block Public Access** can override a permissive bucket policy and silently keep
  data private.
- ❌ Letting cross-account writers own objects through legacy ACLs. Use Bucket owner enforced and
  policy-based access for modern designs.
- ❌ Assuming an access point replaces the bucket policy or closes direct bucket access. Both policy
  layers apply, and unwanted paths must be denied explicitly.
- ❌ Monitoring only GB-month. Request volume, retrieval, transfer, replication, KMS, old versions,
  and incomplete multipart uploads can dominate cost.

---

**Next**: [04_s3_storage_classes_and_management.md — S3 Storage Classes, Lifecycle & Management](04_s3_storage_classes_and_management.md)
