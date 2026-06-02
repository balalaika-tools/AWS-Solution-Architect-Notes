# CloudFront: Content Delivery at the Edge

> **Who this is for**: Engineers who understand DNS ([01](01_dns_fundamentals.md)) and Route 53 ([02](02_route_53.md)) and now want to cut latency for global users. We start with what a CDN *is* (no AWS) before diving into CloudFront. No CDN background assumed.

---

## 1. What a CDN Is and Why Edge Caching Cuts Latency

Imagine your servers (the **origin**) live in `us-east-1` (Virginia). A user in Sydney requesting an image has to send packets across the Pacific and back — every request. That round trip is dominated by **physics**: distance × speed of light, plus every router hop in between. You can't make light faster.

A **CDN (Content Delivery Network)** is a globally distributed fleet of cache servers (**edge locations / Points of Presence, PoPs**) placed close to users. The first time someone near an edge requests content, the edge fetches it from the origin and **caches a copy**. Every subsequent nearby user is served from the edge — a short trip — and never touches the origin.

```
  WITHOUT a CDN                          WITH a CDN (CloudFront)
  ─────────────                          ───────────────────────
  Sydney user                            Sydney user
      │  ~300ms round trip                   │  ~10ms to nearby edge
      │  across the Pacific                  ▼
      ▼                              ┌──────────────────┐
  ┌──────────┐                       │ Sydney edge (PoP)│── cached? serve it
  │  Origin  │                       └────────┬─────────┘
  │ us-east-1│                                │ cache MISS only:
  └──────────┘                                ▼  fetch once from origin
                                       ┌──────────┐
                                       │  Origin  │
                                       │ us-east-1│
                                       └──────────┘
```

> **Key insight**: A CDN wins two ways. (1) **Lower latency** — content is physically closer. (2) **Origin offload** — most requests are served from cache, so your origin handles a fraction of the traffic (cheaper, more resilient, absorbs spikes). Even cache *misses* are faster, because CloudFront edges reach the origin over AWS's optimized backbone, not the open internet.

**CloudFront** is AWS's CDN, with 600+ edge locations and regional edge caches worldwide.

---

## 2. Distributions and Origins

A **distribution** is the CloudFront configuration unit — it gets a domain name like `d111abc.cloudfront.net` (which you typically front with a Route 53 **Alias** record, e.g. `cdn.example.com`). A distribution defines one or more **origins** and how to cache/route to them.

An **origin** is where CloudFront fetches content on a cache miss:

| Origin type | What it is | Note |
|-------------|-----------|------|
| **S3 bucket (REST)** | Private S3 bucket via the S3 API | Lock it down with **OAC** so only CloudFront can read it (§5). |
| **S3 static website endpoint** | S3's `*.s3-website-*` endpoint | Use when you need S3 website features (index/error docs, redirects). **Cannot** use OAC — it's a public HTTP endpoint, not the S3 API. |
| **ALB** | An Application Load Balancer | Dynamic content from EC2/containers behind it. |
| **Custom HTTP origin** | Any HTTP server (EC2, on-prem, even a non-AWS host) | Must be reachable over HTTP/HTTPS. |

A single distribution can have **multiple origins** and route to them by URL path using **cache behaviors** (§3) — e.g. `/static/*` → S3, `/api/*` → ALB.

---

## 3. Cache Behaviors, TTL, and Invalidation

**Cache behaviors** are ordered rules matching URL **path patterns** (`/images/*`, `/api/*`, default `*`). Each behavior controls: which origin, allowed HTTP methods, whether to forward query strings/headers/cookies, viewer protocol policy (e.g. **redirect HTTP→HTTPS**), and caching/TTL.

**TTL controls how long an edge keeps an object before re-validating with the origin** — same TTL concept as DNS, different layer:

- Origin can send `Cache-Control: max-age=...` / `Expires` headers; CloudFront respects them within Min/Max bounds you set.
- You also set **Minimum / Default / Maximum TTL** on the behavior.

To remove cached content *before* its TTL expires, you have two tools:

| Technique | How | When |
|-----------|-----|------|
| **Invalidation** | Tell CloudFront to purge paths (`/index.html`, `/*`) | Emergency / occasional purge. First 1,000 paths/month free, then charged. |
| **Versioned filenames** | Change the object name on deploy (`app.v2.js`) | ✅ Preferred. New name = guaranteed cache miss, no invalidation cost, old version still cached for in-flight clients. |

💡 ✅ Best practice is **versioned object names** over invalidations: cheaper, instant, and avoids the "I invalidated but users still see old content" race.

---

## 4. HTTPS and ACM (the us-east-1 Rule)

CloudFront serves over HTTPS. To use **your own domain** (`cdn.example.com`) with a trusted certificate, you attach an **ACM (AWS Certificate Manager)** certificate.

> **Rule**: The ACM certificate for a CloudFront distribution **must be created/imported in `us-east-1`**, regardless of where your origin or users are. CloudFront is a global service whose control plane lives in `us-east-1`. This is one of the most-tested CloudFront facts.

(Contrast: an ACM cert for an **ALB** must be in the ALB's *own* Region. The us-east-1 rule is specific to CloudFront.)

CloudFront supports **SNI** for free (multiple certs/domains on shared IPs) and can enforce a minimum TLS version. The default `*.cloudfront.net` name already has a valid cert; you only need ACM for custom domains.

---

## 5. Restricting Origin Access: OAC vs OAI

For a **private S3 bucket** origin, you want users to reach content *only* through CloudFront (so you can enforce HTTPS, WAF, signed URLs) and never hit the bucket directly. Two mechanisms:

| | **OAC (Origin Access Control)** | **OAI (Origin Access Identity)** |
|---|----------------------------------|-----------------------------------|
| Status | ✅ **Current**, AWS-recommended | ⚠️ **Legacy** |
| Auth | SigV4 signed requests from CloudFront | A special CloudFront principal in the bucket policy |
| Supports SSE-KMS encrypted objects | ✅ Yes | ❌ No (a key reason OAC exists) |
| All Regions / newer S3 features | ✅ Yes | ❌ Limited |
| Recommendation | Use OAC for new setups | Only for existing/legacy distributions |

The bucket policy is then locked to allow reads **only** from the distribution; direct bucket URLs return Access Denied.

⚠️ OAC/OAI work with the **S3 REST origin**, not the S3 **static website** endpoint (that endpoint is public HTTP and can't be SigV4-restricted). If you need a private bucket behind CloudFront, use the REST endpoint + OAC.

---

## 6. Restricting Who Can View: Signed URLs, Signed Cookies, Geo-Restriction

CloudFront can restrict the *audience* of content (premium media, paid downloads, regional licensing):

| Control | What it does | Use case |
|---------|-------------|----------|
| **Signed URL** | A time-limited, signed link to **one specific file** | Individual paid file / download link. |
| **Signed cookies** | Grants access to **multiple files** without changing each URL | Streaming a whole catalog / subscriber area. |
| **Geo-restriction** | Allow/deny by **viewer country** (CloudFront-native, or via WAF for finer rules) | Content licensing, legal/compliance blocking by country. |

💡 Signed URL = one file; signed cookies = many files. Both rely on a key pair / key group you configure.

---

## 7. Edge Compute: CloudFront Functions vs Lambda@Edge

You can run code at the edge to customize requests/responses (redirects, header manipulation, auth checks, A/B routing) without a round trip to the origin.

| | **CloudFront Functions** | **Lambda@Edge** |
|---|---------------------------|------------------|
| Runtime | JavaScript (lightweight) | Node.js / Python |
| Triggers | Viewer request / viewer response only | Viewer + **origin** request/response (all 4) |
| Scale & latency | Millions/sec, sub-ms, runs at **edge locations** | Lower scale, ms latency, runs at **regional edge caches** |
| Capabilities | Header/URL manipulation, redirects, simple auth — **no network/disk** | Network calls, larger compute, access to request body |
| Cost | Much cheaper | More expensive |

> **Rule**: Reach for **CloudFront Functions** for lightweight, high-volume header/URL/redirect logic. Use **Lambda@Edge** when you need richer compute, the origin-side triggers, or network/disk access.

---

## 8. Price Classes, WAF & Shield

- **Price Classes** trade coverage for cost: **All** (every edge, lowest latency, highest cost) → **200** (excludes the most expensive regions) → **100** (US/Canada/Europe only, cheapest). Choose based on where your users actually are.
- **AWS WAF** attaches to a CloudFront distribution to filter requests at the edge (SQL injection, rate limiting, geo/IP rules) before they reach the origin.
- **AWS Shield Standard** is automatic and free on CloudFront (DDoS protection at the edge); **Shield Advanced** adds enhanced protection and cost protection. Absorbing DDoS at the edge keeps it away from your origin.

---

## 9. When to Use CloudFront vs Plain S3 Static Website Hosting

| Need | S3 static website hosting alone | CloudFront (+ S3 origin) |
|------|--------------------------------|---------------------------|
| Global low latency | ❌ Served from one Region | ✅ Cached at edges worldwide |
| **HTTPS on a custom domain** | ❌ S3 website endpoint is HTTP only | ✅ via ACM (us-east-1) |
| Private bucket | ❌ Must be public | ✅ Private + OAC |
| WAF / Shield / signed URLs / geo-restriction | ❌ | ✅ |
| Simplicity / cost for a tiny internal/regional site | ✅ Simplest | More config |

> **Key insight**: If the exam mentions **HTTPS on a custom domain for a static site**, or **global users**, or a **private bucket**, the answer is **CloudFront in front of S3** — not S3 website hosting alone. The S3 static website endpoint cannot serve HTTPS on your own domain.

See the full trade-off walkthrough: **[S3 static site vs CloudFront + S3](../18_practical_examples/10_s3_static_site_vs_cloudfront.md)**.

---

## Key Exam Points

- CloudFront caches at **edge locations** to cut latency and **offload the origin**; even misses are faster via the AWS backbone.
- Origins: **S3 (REST, lock with OAC)**, **S3 static website endpoint (HTTP, no OAC)**, **ALB**, **custom HTTP**. One distribution, multiple origins routed by **cache behaviors** (path patterns).
- **ACM cert for CloudFront must be in `us-east-1`** (global service). ALB certs go in the ALB's Region.
- **OAC is current; OAI is legacy.** OAC supports SSE-KMS; OAI doesn't. Use OAC for private S3 origins.
- **Signed URL = one file; signed cookies = many files.** **Geo-restriction** allows/blocks by country.
- **CloudFront Functions** = lightweight JS, viewer-side, huge scale, cheap. **Lambda@Edge** = richer compute, all 4 triggers, network access.
- **Price classes** (All/200/100) trade edge coverage for cost. **WAF + Shield** protect at the edge.
- Custom-domain HTTPS / global / private-bucket static site → **CloudFront in front of S3**, not S3 website alone.

---

## Common Mistakes

- ❌ Creating the ACM cert in the origin's Region for a CloudFront distribution. It **must** be `us-east-1`.
- ❌ Using OAI for a new setup or with SSE-KMS objects. Use **OAC**.
- ❌ Trying to apply OAC to the S3 **static website** endpoint. OAC needs the S3 **REST** endpoint.
- ❌ Relying on invalidations for routine deploys. Prefer **versioned filenames**; invalidations cost money and race with caches.
- ❌ Expecting S3 static website hosting to serve HTTPS on a custom domain. It can't — put CloudFront in front.
- ❌ Picking Lambda@Edge for a simple header rewrite. **CloudFront Functions** is cheaper and faster for that.

---

**Next**: [04_global_accelerator.md — Global Accelerator: Anycast Networking](04_global_accelerator.md)
