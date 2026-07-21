# Hosting a Static Site: S3 Website Hosting vs S3 + CloudFront (OAC)

> **Who this is for**: Engineers prepping for SAA-C03 who need to host a static
> site (HTML/JS/CSS, SPA build output) and must choose between plain S3 website
> hosting and a private S3 bucket fronted by CloudFront. Assumes you know S3
> basics — see [S3 storage classes & management](../05_storage/04_s3_storage_classes_and_management.md)
> and [CloudFront](../08_dns_edge/03_cloudfront.md).

There are two canonical ways to serve a static site from S3. The exam loves the
distinction because only **one** of them gives you HTTPS on a custom apex domain.

---

## 1. The Two Approaches at a Glance

```
APPROACH A — S3 Static Website Hosting (public bucket)
──────────────────────────────────────────────────────
   Browser ──HTTP──▶  http://my-site.s3-website-us-east-1.amazonaws.com
                              │
                              ▼
                     ┌──────────────────┐
                     │  S3 bucket        │  Bucket policy: public-read
                     │  Website endpoint │  Index/Error documents set
                     │  HTTP only (:80)  │
                     └──────────────────┘


APPROACH B — Private S3 + CloudFront with OAC
──────────────────────────────────────────────────────
   Browser ──HTTPS──▶  https://www.example.com (Route 53 alias)
                              │
                              ▼
                     ┌──────────────────┐
                     │   CloudFront      │  ACM cert (HTTPS), caching,
                     │   distribution    │  custom domain, WAF-capable
                     └──────────────────┘
                              │  signed SigV4 request (OAC)
                              ▼
                     ┌──────────────────┐
                     │  S3 bucket        │  Block Public Access = ON
                     │  REST API endpoint│  Policy: only this distribution
                     │  (private)        │
                     └──────────────────┘
```

> **Key insight**: The S3 **website endpoint** (`*.s3-website-*.amazonaws.com`)
> speaks HTTP only — it has no TLS. The S3 **REST endpoint**
> (`*.s3.amazonaws.com`) supports HTTPS but doesn't do index-document /
> redirect rules. CloudFront is what bridges custom-domain HTTPS to a private bucket.

---

## 2. Approach A — S3 Static Website Hosting

Fast to stand up, good for internal tooling or HTTP-only demos.

```bash
# 1. Create the bucket (name must be globally unique; for website hosting the
#    bucket name traditionally matches the domain, e.g. www.example.com)
aws s3api create-bucket \
  --bucket www.example.com \
  --region us-east-1

# 2. Disable Block Public Access (required — website hosting needs public reads)
aws s3api put-public-access-block \
  --bucket www.example.com \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# 3. Enable static website hosting with index + error documents
aws s3 website s3://www.example.com/ \
  --index-document index.html \
  --error-document error.html
```

Attach a **public-read** bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::www.example.com/*"
  }]
}
```

Resulting endpoint (HTTP only):
`http://www.example.com.s3-website-us-east-1.amazonaws.com`

⚠️ You **cannot** put an ACM certificate on the S3 website endpoint. If a
question requires `https://` on the apex/www of a custom domain, plain website
hosting is the wrong answer.

---

## 3. Approach B — Private Bucket + CloudFront with OAC

This is the modern, exam-correct pattern for production static sites. **OAC**
(Origin Access Control) replaces the legacy **OAI** (Origin Access Identity) —
OAC supports SSE-KMS and all regions; if you see OAI in a "current best
practice" answer it's usually a distractor.

```bash
# 1. Bucket stays PRIVATE — leave Block Public Access fully ON
aws s3api create-bucket --bucket example-site-assets --region us-east-1
aws s3api put-public-access-block \
  --bucket example-site-assets \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# 2. Create an Origin Access Control
aws cloudfront create-origin-access-control \
  --origin-access-control-config \
  'Name=site-oac,SigningProtocol=sigv4,SigningBehavior=always,OriginAccessControlOriginType=s3'
```

Console steps for the distribution:

1. CloudFront → **Create distribution**.
2. **Origin domain**: pick the bucket's **REST** endpoint
   (`example-site-assets.s3.us-east-1.amazonaws.com`), *not* the website endpoint.
3. **Origin access**: choose **Origin access control settings (recommended)** →
   select `site-oac`.
4. **Viewer protocol policy**: **Redirect HTTP to HTTPS**.
5. **Default root object**: `index.html`.
6. **Alternate domain names (CNAMEs)**: `www.example.com`.
7. **Custom SSL certificate**: select an ACM cert in **us-east-1** (CloudFront
   only reads certs from us-east-1, regardless of bucket region).

Bucket policy that allows **only this distribution** (CloudFront prints this for
you to copy):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipal",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::example-site-assets/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::111122223333:distribution/E1ABCDEF2GHIJK"
      }
    }
  }]
}
```

Route 53 **alias** record (A/AAAA) pointing the domain at the distribution:

```hcl
resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.primary.zone_id
  name    = "www.example.com"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.site.domain_name
    zone_id                = aws_cloudfront_distribution.site.hosted_zone_id  # always Z2FDTNDATAQYW2
    evaluate_target_health = false
  }
}
```

💡 An **alias** record (vs a CNAME) lets you point the **apex** (`example.com`,
not just `www`) at CloudFront, and AWS doesn't charge for alias queries to AWS
resources. A CNAME cannot live at the zone apex.

---

## 4. Make the CloudFront Pattern Production-Ready

The private origin is only the foundation. A public production distribution
also needs edge security, observable deployments, and a recovery path.

### Edge security and DNS

- Attach a **CloudFront-scope AWS WAF web ACL** to the distribution (its control
  plane is in `us-east-1`). Start managed rule groups in count mode, review
  false positives, then enforce. Add rate-based rules for abusive clients and
  protect the WAF logging destination from general access.
- Keep both Route 53 `A` and `AAAA` alias records when IPv6 is enabled. The ACM
  certificate for viewer HTTPS must cover every alternate domain name and be in
  `us-east-1`; validate renewal records and alarm before expiry.
- Attach a **response headers policy** to every relevant cache behavior. A
  security baseline normally includes HSTS, `Content-Security-Policy`,
  `X-Content-Type-Options`, `X-Frame-Options` or CSP `frame-ancestors`, and a
  suitable `Referrer-Policy`. Test CSP in report-only mode before enforcement.
- Keep the origin bucket private with Block Public Access and an OAC policy
  restricted by the exact distribution ARN. If objects use SSE-KMS, the KMS key
  policy must also allow the CloudFront distribution through OAC.

Enable CloudFront standard access logs to a protected logging destination and
WAF logs for security decisions. Define retention/lifecycle rules, remove or
restrict sensitive query strings/cookies, and alert on elevated 4xx/5xx rates,
origin latency, low cache-hit ratio, and WAF blocks. Real-time logs are useful
for incident response but cost more and should be enabled only where the lower
latency is required.

### Deploy and roll back without cache surprises

Use immutable, content-hashed asset names such as
`app.4f8c2a.js`. Upload those assets first with a long `Cache-Control` lifetime,
then upload `index.html` and any service worker with a short/no-cache policy.
This avoids invalidating the whole distribution on each release.

For a release that must replace a stable path, invalidate only the small set of
mutable entry points:

```bash
aws cloudfront create-invalidation \
  --distribution-id E1ABCDEF2GHIJK \
  --paths /index.html /service-worker.js
```

Store each release under an immutable prefix or retain the previous hashed
artifacts. Roll back by publishing the previous entry document/manifest and
invalidating that entry point. S3 Versioning protects against accidental
overwrite or deletion, but its object version IDs do **not** by themselves
change a CloudFront cache key. Test the deployed hostname, headers, asset
integrity, and a cache miss before declaring the rollout complete.

### Origin and Regional failure

For a Regional recovery requirement, keep a versioned secondary bucket in a
second Region with S3 Cross-Region Replication, grant the same distribution OAC
access to both buckets, and configure them as a CloudFront **origin group**.
Choose failover status codes and shorter origin connection attempts/timeouts
from measured behavior. CloudFront origin failover applies only to
`GET`, `HEAD`, and `OPTIONS`, which fits a read-only static site.

Replication is asynchronous, so this design can serve an older release or miss
a newly published object during a sudden failure. Track replication status and
lag, use S3 Replication Time Control if its SLA is required, and do not make the
secondary live until a release is complete there. Run an origin-failure drill
and a failback drill; edge caches can make an untested secondary look healthy.
For less critical sites, S3's in-Region durability plus versioning and a tested
restore may be a better cost tradeoff than a continuously replicated origin.

### Cost controls

The recurring bill includes CloudFront requests/data transfer, WAF requests and
rules, log ingestion/storage, invalidations beyond the free allowance, and
possibly replication and Origin Shield. Improve cache hit ratio with a cache
policy that forwards only required headers/cookies/query strings, enable
compression, lifecycle old logs/noncurrent S3 versions, and choose a CloudFront
price class that matches the user footprint. Review cost and cache metrics
together; an overly short TTL can erase both the latency and cost benefit of the
CDN.

---

## 5. Comparison Table

| Capability                  | A: S3 Website Hosting        | B: S3 (private) + CloudFront + OAC |
|-----------------------------|------------------------------|-------------------------------------|
| HTTPS                       | ❌ HTTP only                 | ✅ via ACM cert (us-east-1)         |
| Custom apex domain (HTTPS)  | ❌                           | ✅ Route 53 alias                   |
| Edge caching / low latency  | ❌ single region             | ✅ global edge locations            |
| Bucket exposure             | ⚠️ public-read               | ✅ fully private (Block Public ON)  |
| WAF / geo-restriction       | ❌                           | ✅                                  |
| SPA routing / error pages   | ✅ index/error docs          | ✅ custom error responses → index   |
| Cost (low traffic)          | ✅ cheapest (S3 only)        | ⚠️ + CloudFront request/transfer    |
| Setup effort                | ✅ minimal                   | ⚠️ more moving parts                |
| Exam "correct" for prod     | ❌                           | ✅                                  |

---

## 6. When to Choose Each

✅ **Choose A (website hosting)** when: HTTP is acceptable, it's an internal/dev
artifact, you need S3's redirect rules, and you have no custom-domain HTTPS
requirement.

✅ **Choose B (CloudFront + OAC)** when: you need HTTPS on a custom domain,
global low latency, a private bucket, WAF, or geo-restriction — i.e. essentially
every production public website. This is the default exam answer.

⚠️ **Trap**: "Serve a static website over HTTPS on a custom domain at lowest
cost." The lowest-cost option that *meets the HTTPS requirement* is still
CloudFront + private S3 — plain website hosting fails the HTTPS requirement
outright, so cheaper-but-wrong.

---

## 7. Troubleshooting

**CloudFront returns 403 AccessDenied for every object**

- The bucket policy doesn't reference the distribution, or `AWS:SourceArn`
  doesn't match the real distribution ARN. Re-copy the policy CloudFront
  generates.
- You pointed the CloudFront origin at the **website endpoint** instead of the
  **REST endpoint** while using OAC. OAC signs SigV4 requests that only the REST
  endpoint understands. Switch the origin to `*.s3.<region>.amazonaws.com`.
- Block Public Access being **ON** is *expected and correct* with OAC — it is
  not the cause.

**CloudFront 403 only on directory paths like `/about/`**

- No **default root object** equivalent for subpaths. For SPAs, add a custom
  error response: HTTP 403/404 → response code 200 → response page `/index.html`.

**`https://...s3-website-...amazonaws.com` won't load over HTTPS**

- Correct behavior — the website endpoint is HTTP-only. Use Approach B for HTTPS.

**ACM certificate doesn't appear in the CloudFront cert dropdown**

- The cert isn't in **us-east-1**. CloudFront only uses ACM certs from
  us-east-1, even if your bucket and users are elsewhere. Re-request it there.

**Old content keeps serving after deploy**

- CloudFront cached it for the TTL. Run
  `aws cloudfront create-invalidation --distribution-id E1ABCDEF2GHIJK --paths "/*"`
  for an emergency, or normally version asset filenames and invalidate only
  mutable entry documents.

**The secondary origin works, but serves a mixed or older release**

- Cross-Region Replication had not completed before deployment, or the HTML
  entry point and hashed assets were published in the wrong order. Gate release
  promotion on replication status and upload immutable assets before HTML.

---

## Key Exam Points

- S3 **website endpoint = HTTP only**; S3 **REST endpoint = HTTPS** but no index docs.
- Custom-domain **HTTPS** for a static site ⇒ **CloudFront + ACM**, full stop.
- **ACM cert for CloudFront must be in us-east-1.**
- Modern S3↔CloudFront access = **OAC** (not OAI); keep **Block Public Access ON**.
- Apex domain → CloudFront uses a **Route 53 alias** record (CNAME can't sit at apex).
- A 403 from CloudFront+OAC almost always = bucket policy / wrong origin endpoint.
- Production adds **WAF, response-header policies, access/WAF logs, immutable
  asset versioning, targeted invalidations, alarms, and a tested rollback**.
- Multi-Region static-site recovery can use replicated S3 buckets plus a
  CloudFront origin group, but replication is asynchronous and failover is only
  for `GET`/`HEAD`/`OPTIONS` requests.

---

**Next**: [11_rds_private_access.md — RDS in Private Subnets, Accessed from the App Tier](11_rds_private_access.md)
