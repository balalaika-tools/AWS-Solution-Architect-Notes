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
        ┌──────────────── Bucket: "acme-reports" (region: us-east-1) ──────────────┐
        │  key = "2026/q1/financials.pdf"   →  [ data + metadata ]                  │
        │  key = "2026/q1/summary.csv"      →  [ data + metadata ]                  │
        │  key = "logos/header.png"         →  [ data + metadata ]                  │
        └──────────────────────────────────────────────────────────────────────────┘
   Access:  GET https://acme-reports.s3.us-east-1.amazonaws.com/2026/q1/financials.pdf
```

> **Key insight**: S3 is not a filesystem. There are no real folders — just keys with `/` in them.
> The "folder" you see in the console is a UI convenience over a **flat namespace**.

---

## 2. Buckets, Keys, Prefixes

- **Bucket** — the top-level container. Bucket **names are globally unique across all of AWS** (DNS-
  addressable), but a bucket is **created in and tied to one Region**. Data never leaves that Region
  unless you replicate it.
- **Key** — the full name of an object within the bucket, e.g. `2026/q1/financials.pdf`.
- **Prefix** — the leading part of a key up to a delimiter, e.g. `2026/q1/`. Prefixes are how you
  *organize* and *filter* objects (and how lifecycle/replication rules target subsets). They look
  like folders but are just string prefixes on a flat namespace.

| Concept | Scope | Example |
|---------|-------|---------|
| Bucket | Globally unique name, Region-scoped storage | `acme-reports` |
| Key | Unique within a bucket | `2026/q1/financials.pdf` |
| Prefix | A key namespace for grouping/filtering | `2026/q1/` |

⚠️ "Globally unique" applies to the **bucket name**, not the data location. The data still physically
lives in the **single Region** you chose. Don't confuse global naming with global storage.

---

## 3. Durability, Availability, and Limits

| Property | Value | Meaning |
|----------|-------|---------|
| **Durability** | **11 nines (99.999999999%)** | Across all storage classes (objects redundantly stored across multiple devices/AZs in a Region). Effectively you will not lose an object. |
| **Availability** | ~99.99% (S3 Standard) | Designed availability for retrieval; varies by storage class. |
| **Max object size** | **5 TB** | A single object. |
| **Single PUT limit** | **5 GB** | Above this you **must** use multipart upload. |
| **Multipart recommended** | **> 100 MB** | AWS recommends multipart for anything over ~100 MB. |
| **Min object size** | 0 bytes | Empty objects are allowed. |
| **Buckets per account** | Soft limit (default 100, raisable to 1,000) | Objects per bucket: unlimited. |

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
   │ Block Public Access     │  (master safety switch)
   └───────────┬────────────┘
               ▼
   ┌────────────────────────┐
   │ IAM + Bucket Policy +   │  evaluated together; explicit DENY always wins
   │ ACL (legacy)            │
   └────────────────────────┘
```

> **Key insight**: **Block Public Access is ON by default** and is the recommended posture. To make
> objects public (e.g., a static website), you must first turn off the relevant Block Public Access
> setting *and* grant read via a **bucket policy**. An explicit **deny** anywhere always wins.

✅ Cross-account or "make this bucket readable by the public/CloudFront" → **bucket policy**.
✅ "Control what my users/roles can do" → **IAM policy**.
❌ Reaching for **ACLs** — AWS now steers you to disable ACLs and use policies; ACLs are legacy.

---

## 6. Bucket vs Object — Quick Contrast

| | **Bucket** | **Object** |
|---|---|---|
| What | Container, Region-scoped, globally unique name | The stored data + key + metadata |
| Limit | Default 100/account (raisable) | Up to 5 TB each; unlimited count |
| Policy attach point | Bucket policy, BPA, versioning, replication, lifecycle | Tags, metadata, storage class, version ID |
| Addressed by | DNS name | Key within the bucket |

---

## Key Exam Points

- S3 is **object storage** over an **HTTP API** — not a disk (EBS) and not a mountable filesystem
  (EFS/FSx). No partial edits; you replace whole objects.
- **Bucket names are globally unique; storage is Region-scoped.** Data stays in its Region unless replicated.
- **11 nines (99.999999999%) durability**; **max object size 5 TB**; **multipart required above 5 GB**,
  recommended above ~100 MB.
- S3 is now **strongly read-after-write consistent** for all operations (since Dec 2020) — ignore old
  "eventually consistent" claims.
- Access layers: **Block Public Access (on by default)** → **IAM + bucket policy + (legacy) ACL**.
  **Bucket policy** for cross-account/public; **IAM policy** for your principals; **explicit deny wins**.

---

## Common Mistakes

- ❌ Treating S3 like a filesystem — there are no real folders, only keys/prefixes in a flat namespace.
- ❌ Thinking "globally unique name" means the data is global. The data lives in one Region.
- ❌ Quoting S3 as eventually consistent — it's strongly consistent now.
- ❌ Trying to upload a 10 GB object in a single PUT — anything over 5 GB **must** use multipart.
- ❌ Using ACLs for cross-account access — use a **bucket policy**; AWS recommends disabling ACLs.
- ❌ Forgetting that **Block Public Access** can override a permissive bucket policy and silently keep
  data private.

---

**Next**: [04_s3_storage_classes_and_management.md — S3 Storage Classes, Lifecycle & Management](04_s3_storage_classes_and_management.md)
