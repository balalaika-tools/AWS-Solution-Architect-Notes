# DNS & Edge

> How names become IP addresses, and how AWS delivers content close to users. Starts with vendor-neutral DNS theory, then covers Route 53 (AWS DNS + registrar + health checks), CloudFront (the CDN), Global Accelerator (anycast networking), and the Route 53 Resolver (the recursive resolver inside your VPC, plus DNS Firewall).

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
| [05_route53_resolver.md](05_route53_resolver.md) | Route 53 Resolver | The `.2` VPC resolver (`base+2` / `169.254.169.253`), inbound/outbound endpoints, resolver rules (Forward/System), **DNS Firewall** (block malicious domains), and **query logging**. The recursive resolver, not authoritative DNS. |

---

## Reading Order

1. **DNS Fundamentals** — the vocabulary (records, TTL, zones, resolution flow) every later file assumes. Do not skip if networking is weak.
2. **Route 53** — how AWS implements DNS and adds health-based routing.
3. **CloudFront** — caching content at the edge to cut latency and offload origins.
4. **Global Accelerator** — improving latency/failover for *any* protocol via the AWS backbone.
5. **Route 53 Resolver** — the *other* Route 53: the recursive resolver inside your VPC, hybrid endpoints, DNS Firewall, and query logging. (Read right after Route 53 if you're focused on DNS rather than edge.)

---

## SAP-C02 Architecture Path

After the DNS prerequisite, follow the professional path by failure boundary:

1. [Route 53 multi-Region recovery](02_route_53.md#9-multi-region-traffic-recovery)
   — build representative health, control regional switches with ARC, and
   measure DNS caching as part of RTO.
2. [Organization-scale hybrid DNS](05_route53_resolver.md#8-organization-scale-hybrid-dns)
   — share regional rules, choose centralized or distributed endpoints, govern
   DNS Firewall/query logs, and eliminate loops and namespace ambiguity.
3. [CloudFront production delivery](03_cloudfront.md#11-production-delivery-resilience-policy-and-operations)
   — design origin failover, cache/origin policies, staged changes, signed access,
   edge security, and cost controls.
4. [Global Accelerator traffic engineering](04_global_accelerator.md#4-traffic-engineering-and-recovery)
   — manage endpoint weights and regional dials, select standard versus custom
   routing, and test connection-level failover and failback.

For each global design, state who owns the traffic switch, how the secondary is
declared ready, which cached or existing sessions remain on the old path, how
data writes are fenced, and what evidence proves the tested RTO.

---

## Prerequisites

- [Networking](../03_networking/README.md) — IP addresses, ports, and how packets get routed; DNS resolves names to the IPs networking uses.
- [HA & Scaling](../07_ha_scaling/README.md) — load balancers are common Route 53 record targets and CloudFront origins.
