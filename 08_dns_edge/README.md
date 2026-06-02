# DNS & Edge

> How names become IP addresses, and how AWS delivers content close to users. Starts with vendor-neutral DNS theory, then covers Route 53 (AWS DNS + registrar + health checks), CloudFront (the CDN), and Global Accelerator (anycast networking).

[![AWS](https://img.shields.io/badge/AWS-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/)
[![Route 53](https://img.shields.io/badge/Route%2053-DNS-FF9900.svg?logo=amazonaws&logoColor=white)](https://docs.aws.amazon.com/route53/)
[![CloudFront](https://img.shields.io/badge/CloudFront-CDN-FF9900.svg?logo=amazonaws&logoColor=white)](https://docs.aws.amazon.com/cloudfront/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_dns_fundamentals.md](01_dns_fundamentals.md) | DNS theory (no AWS) | What DNS is and why, full resolution flow (stub → recursive → root → TLD → authoritative), record types, TTL/caching, FQDN, zones, registrar vs hosting, CNAME vs A, the apex problem. |
| [02_route_53.md](02_route_53.md) | Route 53 | Registrar + authoritative DNS + health checks, public vs private hosted zones, Alias records, all seven routing policies, health checks & DNS failover, TTL behavior. |
| [03_cloudfront.md](03_cloudfront.md) | CloudFront (CDN) | What a CDN is, distributions, origins, edge caching, TTL/invalidation, OAC vs OAI, ACM in us-east-1, signed URLs/cookies, geo-restriction, CloudFront Functions vs Lambda@Edge, WAF/Shield. |
| [04_global_accelerator.md](04_global_accelerator.md) | Global Accelerator | Anycast IPs, the AWS backbone, static IP failover, any-protocol support, and a CloudFront vs Global Accelerator comparison. |

---

## Reading Order

1. **DNS Fundamentals** — the vocabulary (records, TTL, zones, resolution flow) every later file assumes. Do not skip if networking is weak.
2. **Route 53** — how AWS implements DNS and adds health-based routing.
3. **CloudFront** — caching content at the edge to cut latency and offload origins.
4. **Global Accelerator** — improving latency/failover for *any* protocol via the AWS backbone.

---

## Prerequisites

- [Networking](../03_networking/README.md) — IP addresses, ports, and how packets get routed; DNS resolves names to the IPs networking uses.
- [HA & Scaling](../07_ha_scaling/README.md) — load balancers are common Route 53 record targets and CloudFront origins.
