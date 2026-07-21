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
| **ALB** | An Application Load Balancer | Dynamic content from EC2/containers behind it. Can be **public** or **private** — keep it private with **VPC origins** (§6). |
| **Custom HTTP origin** | Any HTTP server (EC2, on-prem, even a non-AWS host) | Must be reachable over HTTP/HTTPS. A private EC2/ALB inside your VPC is reachable via **VPC origins** (§6). |

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

## 6. Reaching Private Origins: VPC Origins (Private ALB / Private EC2)

§5 keeps a **private S3 bucket** reachable only through CloudFront. But what about **dynamic** origins — an ALB or EC2 serving an app? The same goal applies: you don't want the load balancer or instances exposed to the internet at all, only reachable via CloudFront (so WAF, Shield, TLS, and signed URLs can't be bypassed by hitting the origin directly).

Historically CloudFront could only reach an origin over the **public internet**, which forced your ALB/EC2 to have a public IP and an internet-facing listener. **VPC origins** removes that requirement.

**VPC origins** let a CloudFront distribution send traffic to a resource that lives in a **private subnet** — an **internal ALB**, an NLB, or an EC2 instance — with **no public IP and no internet-facing exposure**. CloudFront establishes a managed, private path into your VPC; the origin's security group only needs to allow the VPC origin, not `0.0.0.0/0`.

```
   Internet                         Your VPC (private subnets)
   ────────                         ──────────────────────────
   Viewer                    ┌────────────────────────────────┐
     │                       │                                │
     ▼                       │   ┌───────────────┐   ┌──────┐ │
  ┌──────────┐   VPC origin  │   │ Internal ALB  │──▶│ EC2  │ │
  │CloudFront │══════════════╪══▶│ (no public IP)│   │/ ECS │ │
  │  (+ WAF)  │  private path │   └───────────────┘   └──────┘ │
  └──────────┘               │     SG: allow VPC origin only  │
                             └────────────────────────────────┘
   No internet-facing listener on the ALB — origin is unreachable except via CloudFront.
```

> **Key insight**: VPC origins are the modern answer to *"front a private ALB/EC2 with CloudFront without making them public."* The origin sits in a private subnet; CloudFront is the only way in.

**Before VPC origins (legacy pattern, still tested and valid):** if the ALB *must* be internet-facing, you lock it down so only CloudFront can use it:

| Technique | How it works |
|-----------|-------------|
| **Managed prefix list** | Restrict the ALB security group to allow inbound only from **`com.amazonaws.global.cloudfront.origin-facing`** (AWS-managed prefix list of CloudFront edge IP ranges). Blocks random internet clients. |
| **Secret custom header** | CloudFront adds a secret header (e.g. `X-Origin-Verify: <token>`) to every origin request; the ALB (via a **WAF** rule or listener rule) rejects requests missing it. Stops anyone who discovers the ALB DNS name from bypassing CloudFront. |

💡 Use **both** together for the legacy pattern: the prefix list narrows *who* can connect (CloudFront IPs), the secret header proves the request *came through your distribution* (since other tenants share those same edge IPs).

| | **VPC origins** (modern) | **Prefix list + secret header** (legacy) |
|---|---------------------------|-------------------------------------------|
| Origin exposure | Fully **private** (no public IP) | Public/internet-facing ALB, but filtered |
| Bypass risk | None — no public endpoint exists | Low if header check enforced; ALB DNS is still public |
| Setup | Enable VPC origin, point distribution at private ALB/EC2 | Manage prefix list on SG **+** WAF/listener header rule |
| Recommendation | ✅ Preferred for new designs | Use when an internet-facing ALB is required anyway |

⚠️ Don't confuse this with **OAC** (§5): OAC authenticates CloudFront to **S3 (and Lambda function URLs)** via SigV4. For **ALB/EC2**, the equivalent "only CloudFront can reach me" guarantees come from **VPC origins** or the **prefix-list + secret-header** pattern — not OAC.

---

## 7. Restricting Who Can View: Signed URLs, Signed Cookies, Geo-Restriction

CloudFront can restrict the *audience* of content (premium media, paid downloads, regional licensing):

| Control | What it does | Use case |
|---------|-------------|----------|
| **Signed URL** | A time-limited, signed link to **one specific file** | Individual paid file / download link. |
| **Signed cookies** | Grants access to **multiple files** without changing each URL | Streaming a whole catalog / subscriber area. |
| **Geo-restriction** | Allow/deny by **viewer country** (CloudFront-native, or via WAF for finer rules) | Content licensing, legal/compliance blocking by country. |

💡 Signed URL = one file; signed cookies = many files. Both rely on a key pair / key group you configure.

---

## 8. Edge Compute: CloudFront Functions vs Lambda@Edge

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

## 9. Price Classes, WAF & Shield

- **Price Classes** trade coverage for cost: **All** (every edge, lowest latency, highest cost) → **200** (excludes the most expensive regions) → **100** (US/Canada/Europe only, cheapest). Choose based on where your users actually are.
- **AWS WAF** attaches to a CloudFront distribution to filter requests at the edge (SQL injection, rate limiting, geo/IP rules) before they reach the origin.
- **AWS Shield Standard** is automatic and free on CloudFront (DDoS protection at the edge); **Shield Advanced** adds enhanced protection and cost protection. Absorbing DDoS at the edge keeps it away from your origin.

---

## 10. When to Use CloudFront vs Plain S3 Static Website Hosting

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

## 11. Production Delivery: Resilience, Policy, and Operations

### Multiple origins and origin groups

A distribution can send different paths to different origins and can place a
primary and secondary origin in an **origin group**. On configured origin
failure status codes or connection failures, CloudFront can retry eligible
requests against the secondary. This helps with origin outages; it does not
replicate the content or database. Both origins must contain compatible data,
configuration, certificates, and authorization.

Use S3 replication or an application-specific data strategy before declaring a
second origin ready. Test failure with cached and uncached objects: a healthy
cache can hide an origin failure, while dynamic requests reveal it immediately.
Know which HTTP methods are eligible for origin failover before relying on it
for writes.

### Separate cache policy from origin-request policy

Two policy decisions are related but different:

- A **cache policy** chooses which headers, cookies, and query strings become
  part of the cache key, plus TTL behavior. Every extra cache-key dimension can
  fragment the cache and lower the hit ratio.
- An **origin request policy** chooses additional viewer values forwarded to the
  origin without necessarily putting them in the cache key.

Forward only what the origin requires. For example, an API may need an
`Authorization` header and no caching, while versioned static assets need a small
stable cache key and long TTL. Including all cookies and headers "just in case"
raises origin load and cost and can accidentally mix personalized responses if
the key is wrong.

### Safer distribution changes

**CloudFront continuous deployment** pairs a staging distribution with the
primary and sends a small percentage of production traffic, or selected requests,
to the staging configuration. Compare errors, latency, cache hit rate, and
origin load before promoting. Keep content versions immutable so rollback can
restore both distribution configuration and a known-good asset set without a
global invalidation race.

Standard logs provide durable request history. **Real-time logs** stream sampled
or full request records through Kinesis Data Streams for fast operational
detection, at added logging and stream cost. Use CloudWatch metrics and alarms
for aggregate health, logs for diagnosis, and synthetic canaries for the viewer
path.

### Rotate signed-access keys

For signed URLs and cookies, use trusted key groups and keep private keys outside
CloudFront. Add the new public key, begin signing with the new private key, wait
for old signed objects/cookies to expire, then remove the old public key. Removing
it first immediately invalidates active access. Use short validity periods for
sensitive content and monitor rejected requests during rotation.

### Balance security and cost

- Attach WAF where HTTP filtering, rate control, or managed rules are required;
  tune rules in count mode before blocking to avoid a global false positive.
- Use OAC or VPC origins so users cannot bypass CloudFront and its WAF. Validate
  origin policies after every origin change.
- Raise cache hit ratio before buying more origin capacity. Compression,
  versioned assets, appropriate TTLs, and a small cache key reduce transfer and
  request cost.
- Choose a price class only when excluding some edge locations meets the latency
  objective. Include invalidations, real-time logs, WAF requests, origin fetches,
  and inter-Region origin transfer in the cost model.

---

## Key Exam Points

- CloudFront caches at **edge locations** to cut latency and **offload the origin**; even misses are faster via the AWS backbone.
- Origins: **S3 (REST, lock with OAC)**, **S3 static website endpoint (HTTP, no OAC)**, **ALB**, **custom HTTP**. One distribution, multiple origins routed by **cache behaviors** (path patterns).
- **ACM cert for CloudFront must be in `us-east-1`** (global service). ALB certs go in the ALB's Region.
- **OAC is current; OAI is legacy.** OAC supports SSE-KMS; OAI doesn't. Use OAC for private S3 origins.
- **VPC origins** front a **private ALB/EC2** (no public IP) — the modern way to keep dynamic origins off the internet behind CloudFront. Legacy alternative for an internet-facing ALB: restrict its SG to the **`cloudfront.origin-facing` managed prefix list** + verify a **secret custom header** (via WAF). OAC is for S3, *not* ALB/EC2.
- **Signed URL = one file; signed cookies = many files.** **Geo-restriction** allows/blocks by country.
- **CloudFront Functions** = lightweight JS, viewer-side, huge scale, cheap. **Lambda@Edge** = richer compute, all 4 triggers, network access.
- **Price classes** (All/200/100) trade edge coverage for cost. **WAF + Shield** protect at the edge.
- Custom-domain HTTPS / global / private-bucket static site → **CloudFront in front of S3**, not S3 website alone.
- **Origin groups** can fail eligible reads to a secondary origin, but data and dependency replication remain your responsibility.
- Cache policies define cache keys; origin-request policies can forward extra values without fragmenting the cache.
- Use a staging distribution for controlled changes and overlap trusted public keys during signed-access rotation.

---

## Common Mistakes

- ❌ Creating the ACM cert in the origin's Region for a CloudFront distribution. It **must** be `us-east-1`.
- ❌ Using OAI for a new setup or with SSE-KMS objects. Use **OAC**.
- ❌ Trying to apply OAC to the S3 **static website** endpoint. OAC needs the S3 **REST** endpoint.
- ❌ Relying on invalidations for routine deploys. Prefer **versioned filenames**; invalidations cost money and race with caches.
- ❌ Expecting S3 static website hosting to serve HTTPS on a custom domain. It can't — put CloudFront in front.
- ❌ Picking Lambda@Edge for a simple header rewrite. **CloudFront Functions** is cheaper and faster for that.
- ❌ Trying to use **OAC** to lock down an **ALB/EC2** origin. OAC is for S3 (and Lambda URLs); for private ALB/EC2 use **VPC origins**, or the prefix-list + secret-header pattern for an internet-facing ALB.
- ❌ Making an ALB public just to put CloudFront in front of it. A **private/internal ALB** works directly via **VPC origins** — no public IP needed.
- ❌ Forwarding every cookie/header/query string and destroying cache efficiency, or caching personalized responses under an incomplete key.
- ❌ Assuming origin failover makes writes or state multi-Region without application-level replication and conflict control.
- ❌ Deleting an old signing key before new signatures are deployed and old signed access expires.

---

**Next**: [04_global_accelerator.md — Global Accelerator: Anycast Networking](04_global_accelerator.md)
