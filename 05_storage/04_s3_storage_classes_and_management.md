# S3 Storage Classes, Lifecycle & Management

> **Who this is for**: Engineers who understand the S3 object model
> ([03_s3_fundamentals.md](03_s3_fundamentals.md)) and now need the cost/retrieval trade-offs and the
> data-management features the exam tests hardest. The storage-class comparison table here is the
> single most exam-relevant table in the whole storage section.

---

## 1. Storage Classes

Every class delivers the same **11 nines durability** (One Zone classes are the exception on
*availability/AZ-resilience*, not durability of stored copies). They differ on **cost, retrieval
time, retrieval fee, availability, and minimum storage duration**.

| Class | Access pattern | Retrieval time | Retrieval fee | Min duration | AZs | Relative cost |
|-------|----------------|----------------|---------------|--------------|-----|---------------|
| **S3 Standard** | Frequent | Instant (ms) | None | None | ≥3 | $$$$ (highest storage) |
| **S3 Intelligent-Tiering** | Unknown / changing | Instant | None | None | ≥3 | Auto-tiers; small monitoring fee/object |
| **S3 Standard-IA** | Infrequent, rapid when needed | Instant (ms) | Per-GB fee | **30 days** | ≥3 | $$$ |
| **S3 One Zone-IA** | Infrequent, reproducible | Instant (ms) | Per-GB fee | **30 days** | **1** | $$ (≈20% less than Standard-IA) |
| **Glacier Instant Retrieval** | Archive, but ms access | **Instant (ms)** | Per-GB fee | **90 days** | ≥3 | $ |
| **Glacier Flexible Retrieval** | Archive, occasional | **minutes to hours** (Expedited 1–5 min / Standard 3–5 h / Bulk 5–12 h) | Per-GB + request | **90 days** | ≥3 | ¢ |
| **Glacier Deep Archive** | Long-term archive, rare | **12 h (Standard) / 48 h (Bulk)** | Per-GB + request | **180 days** | ≥3 | ¢ (cheapest) |

> **Key insight**: Move down the table as access gets rarer. The trade is always the same: **lower
> storage cost ↔ longer retrieval time + retrieval fee + longer minimum-duration commitment**. Deleting
> an object before its minimum duration still bills you for the full minimum.

### Picking a class (exam decision tree)

```
Access pattern?
├─ Frequent ─────────────────────────────► S3 Standard
├─ Unknown / shifting ───────────────────► Intelligent-Tiering (let S3 decide)
├─ Infrequent, need it fast & resilient ─► Standard-IA
├─ Infrequent, can recreate if AZ lost ──► One Zone-IA  (cheaper, single AZ)
├─ Archive, need ms retrieval ───────────► Glacier Instant Retrieval
├─ Archive, minutes–hours OK ────────────► Glacier Flexible Retrieval
└─ Archive, 12–48h OK, cheapest ─────────► Glacier Deep Archive
```

✅ "Access pattern is unpredictable / want automatic cost optimization with no retrieval fees" →
**Intelligent-Tiering**.
✅ "Long-term compliance archive, retrieval in ~12 h acceptable, lowest cost" → **Deep Archive**.
⚠️ **One Zone-IA stores in a single AZ** — if that AZ is lost, the data is gone. Only use it for data
you can recreate (e.g., a secondary copy of replaceable thumbnails).

---

## 2. Lifecycle Policies

**Lifecycle rules** automate moving objects between classes and expiring them, by **prefix** or
**tag**. Two action types:

- **Transition** — move objects to a cheaper class after *N* days (e.g., Standard → Standard-IA at 30
  days → Glacier Flexible at 90 → Deep Archive at 365).
- **Expiration** — delete objects (or old versions / delete markers / **incomplete multipart uploads**)
  after *N* days.

```
Day 0  ─► S3 Standard
Day 30 ─► Standard-IA            (transition)
Day 90 ─► Glacier Flexible       (transition)
Day 365─► Glacier Deep Archive   (transition)
Day 2555► delete                 (expiration)
```

💡 Always add a rule to **abort incomplete multipart uploads** (e.g., after 7 days) — orphaned parts
otherwise accrue silent storage cost.

---

## 3. Versioning & MFA Delete

**Versioning** keeps every version of an object in the same key. Enabling it protects against
accidental overwrite and delete.

- Once enabled, it can only be **suspended**, never fully turned off (existing versions remain).
- A **DELETE** doesn't remove data — it adds a **delete marker**; the prior version is recoverable.
- A `GET` returns the latest version unless you request a specific version ID.

**MFA Delete** adds a requirement that **MFA be presented to permanently delete a version or to change
the versioning state**. It can only be enabled by the **bucket owner (root account)** via CLI/API.

⚠️ Versioning **increases storage cost** (you store every version) — pair it with a lifecycle rule to
expire old *noncurrent* versions.

---

## 4. Replication (CRR & SRR)

S3 can asynchronously copy objects to another bucket:

- **CRR (Cross-Region Replication)** — different Region. Use for DR, lower-latency access in another
  geography, compliance.
- **SRR (Same-Region Replication)** — same Region, different bucket. Use for log aggregation, prod→test
  data sync, cross-account access.

**Requirements / behavior:**

- **Versioning must be enabled on both source and destination buckets.**
- Replication is **asynchronous** and applies to objects **created/changed after** it's enabled (use
  **S3 Batch Replication** to backfill existing objects).
- An IAM role grants S3 permission to replicate.
- **Not chained/transitive**: if A→B and B→C, A's objects are *not* automatically copied to C.
- Delete markers can optionally replicate; deletes of specific versions are **not** replicated.

---

## 5. Encryption

S3 encrypts new objects with **SSE-S3 by default**. Options:

| Type | Key managed by | Key control | Use when |
|------|----------------|-------------|----------|
| **SSE-S3** | AWS (AES-256) | None — fully managed | Default; simplest at-rest encryption |
| **SSE-KMS** | AWS KMS | You (key policies, **audit via CloudTrail**, rotation) | Need control/audit over keys; envelope encryption |
| **SSE-C** | **You provide the key** per request | You hold and supply the key; AWS never stores it | You must own/manage keys yourself |
| **DSSE-KMS** | AWS KMS, **double layer** | You | Regulatory requirement for two independent layers of encryption |

- All use **server-side encryption**; you can also do **client-side encryption** before upload.
- Enforce encryption in transit (TLS) and at rest via a **bucket policy** (`aws:SecureTransport`,
  deny unencrypted `PUT`s).

💡 **SSE-KMS** is the exam answer when you need **audit logging of key usage** and **rotation/control**;
**SSE-C** when "the customer must supply and manage the encryption keys themselves."

⚠️ SSE-KMS `PUT`/`GET` calls count against **KMS request quotas** — very high-throughput buckets can
hit KMS throttling. **S3 Bucket Keys** reduce KMS calls and cost.

---

## 6. Presigned URLs

A **presigned URL** grants **temporary, time-limited** access to a specific object (GET to download or
PUT to upload) using the signer's permissions — without making the object public.

- Generated via SDK/CLI; expires after a set duration.
- Classic use: let a logged-in user download a private file, or upload directly to S3 from a browser.

✅ "Give a user temporary access to a private object without changing bucket permissions" → presigned URL.

---

## 7. Event Notifications

S3 can emit events (`s3:ObjectCreated:*`, `s3:ObjectRemoved:*`, etc.) to trigger downstream processing:

- Targets: **SNS**, **SQS**, **Lambda**, and **EventBridge** (EventBridge adds richer filtering, more
  targets, and replay).
- Classic pattern: upload an image → event → Lambda generates a thumbnail.

```
PUT object ─► S3 event ─► (SNS | SQS | Lambda | EventBridge) ─► processing
```

---

## 8. Transfer Acceleration & Requester Pays

- **Transfer Acceleration** — uploads/downloads route through the nearest **CloudFront edge** then over
  AWS's backbone to the bucket, speeding **long-distance** transfers. Use it when users far from the
  bucket's Region upload large objects.
- **Requester Pays** — the **requester** (not the bucket owner) pays for data transfer and request
  costs. Used for sharing large datasets without absorbing egress cost. Requesters must be authenticated.

---

## 9. Object Lock & Glacier Vault Lock (WORM)

For **WORM (Write Once, Read Many)** compliance — data that must not be altered or deleted for a period:

- **S3 Object Lock** — prevents object versions from being deleted/overwritten. Modes: **Governance**
  (privileged users can override) and **Compliance** (no one, not even root, can delete until the
  retention period expires). Plus **Legal Hold** (indefinite, until removed). Requires versioning.
- **S3 Glacier Vault Lock** — applies a lockable, immutable policy to a Glacier *vault* for WORM/compliance.

✅ "Records must be immutable for 7 years for regulatory compliance, no one can delete" → **Object Lock
in Compliance mode** (or Glacier Vault Lock).

---

## 10. Static Website Hosting

S3 can serve a **static website** (HTML/CSS/JS/images) directly:

- Enable static website hosting, set an index and error document.
- Requires the objects to be publicly readable (disable Block Public Access for the bucket + a bucket
  policy granting `s3:GetObject`) **or** front it with **CloudFront** + Origin Access Control (OAC) to
  keep the bucket private and add HTTPS/caching.
- S3 static sites serve **HTTP only**; to get **HTTPS + caching + a custom domain**, put **CloudFront**
  in front.

See the full walkthrough: [../18_practical_examples/10_s3_static_site_vs_cloudfront.md](../18_practical_examples/10_s3_static_site_vs_cloudfront.md).

---

## Key Exam Points

- Memorize the **storage-class trade-off**: lower storage cost ↔ longer retrieval + retrieval fee +
  longer min-duration (Standard-IA/One Zone-IA = 30 d; Glacier classes = 90 d; Deep Archive = 180 d).
- **Intelligent-Tiering** = unknown/changing access, auto-optimizes, **no retrieval fees**. **One
  Zone-IA** = single AZ, only for reproducible data. **Deep Archive** = cheapest, ~12 h retrieval.
- **Replication requires versioning on both buckets**, is async, isn't retroactive (use Batch
  Replication), and isn't transitive.
- Encryption: **SSE-S3** (default), **SSE-KMS** (audit/control), **SSE-C** (you supply keys),
  **DSSE-KMS** (double-layer).
- **Presigned URL** = temporary access to a private object. **Object Lock (Compliance)** = WORM/
  immutable records.
- **Transfer Acceleration** speeds long-distance uploads via edge locations; **Requester Pays** shifts
  cost to the downloader.

---

## Common Mistakes

- ❌ Using One Zone-IA for critical data — a single AZ loss destroys it.
- ❌ Enabling replication without versioning on **both** buckets — it won't work.
- ❌ Expecting replication to copy **existing** objects — only new/changed ones (use Batch Replication).
- ❌ Deleting IA/Glacier objects early and being surprised by the **minimum-duration charge**.
- ❌ Choosing SSE-C when the requirement is "audit key usage" — that's SSE-KMS (CloudTrail logs KMS).
- ❌ Serving an HTTPS static site straight from S3 — S3 website endpoints are HTTP; use CloudFront for HTTPS.

---

**Next**: [05_s3_advanced_features.md — CORS, Access Logs, Access Points, Storage Lens, Batch Operations & Performance](05_s3_advanced_features.md)
