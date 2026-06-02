# Route 53: AWS DNS, Registrar & Health Checks

> **Who this is for**: Engineers who have read [01_dns_fundamentals.md](01_dns_fundamentals.md) and understand records, zones, TTL, the resolution flow, and the apex/CNAME problem. Now we map all of that onto AWS's DNS service. If "authoritative name server," "zone apex," and "TTL" don't ring a bell, go back one file.

---

## 1. Where Route 53 Fits the DNS Model

Recall from [01_dns_fundamentals.md](01_dns_fundamentals.md) that "registrar" and "authoritative DNS host" are two separate jobs. **Route 53 does both, plus a third thing (health checks) that ordinary DNS providers don't.** The name "53" is the DNS port.

| Job | What Route 53 does |
|-----|--------------------|
| **Domain registrar** | Register/transfer domains (`.com`, `.org`, etc.) and pay the yearly fee. Optional — you can register elsewhere. |
| **Authoritative DNS** | Host your zone and answer queries. This is the **hosted zone** (§2). When you register through Route 53, it auto-creates the hosted zone and wires the `NS` records for you. |
| **Health checking** | Probe endpoints and route DNS answers based on health — true DNS-level failover (§6). This is the differentiator. |

> **Key insight**: Route 53 is a globally distributed *authoritative* name server with a 100% availability SLA. It is **not** a recursive resolver for your apps (that's the Route 53 *Resolver*, a separate feature — see the hybrid DNS example linked at the bottom). When the world resolves `example.com`, Route 53 is the authoritative server giving the answer.

---

## 2. Hosted Zones: Public vs Private

A **hosted zone** is Route 53's name for a zone (§4 of the fundamentals file) — a container for the records of one domain. You create records inside it. There are two kinds:

| | **Public hosted zone** | **Private hosted zone** |
|---|------------------------|--------------------------|
| Answers for | The **public internet** | Only **inside associated VPC(s)** |
| Resolved by | Anyone on the internet | Only resources in the linked VPC(s) (via the VPC's `.2` resolver) |
| Typical use | Your real website / public services | Internal service names (`db.internal`, `api.corp.local`) |
| Requirement | — | The VPC needs `enableDnsHostnames` and `enableDnsSupport` = true |

You can have a **public and a private zone with the same name** (split-horizon DNS): internal clients get an internal IP, the internet gets a public IP. The private zone wins for queries originating inside the associated VPC.

💡 Every hosted zone has a monthly cost (~$0.50/zone) plus per-query charges. **Alias queries to AWS resources are free** (§4).

---

## 3. Records in Route 53

Inside a hosted zone you create the same record types from [01_dns_fundamentals.md](01_dns_fundamentals.md) — `A`, `AAAA`, `CNAME`, `MX`, `TXT`, `NS`, `SOA`, etc. Route 53 auto-creates the `NS` and `SOA` records when you make the zone; you fill in the rest. Each record additionally carries a **routing policy** (§5) and an optional **health check** (§6) — those two are the AWS-specific superpowers.

---

## 4. Alias Records (the Apex Fix, and It's Free)

Remember the apex problem: you **can't** put a `CNAME` on `example.com`. Route 53's answer is the **Alias record**.

An **Alias record** is a Route 53-specific extension (not standard DNS) that points a name at a *specific AWS resource* — and Route 53 resolves it internally to the resource's current IP(s), returning a plain `A`/`AAAA` answer to the client.

```
  CNAME (subdomain only):
    www.example.com ──CNAME──► myalb-123.us-east-1.elb.amazonaws.com
                                (client does a SECOND lookup; billed query)

  ALIAS (works at apex):
    example.com ──ALIAS──► [ALB]   Route 53 resolves internally and
                                   returns A 52.x.x.x directly to client
                                   (one lookup; FREE if target is AWS)
```

| | **CNAME** | **Alias** |
|---|-----------|-----------|
| Standard DNS? | Yes | No — Route 53 only |
| Works at zone apex (`example.com`)? | ❌ No | ✅ Yes |
| Points to | Any DNS name | Specific AWS resources (see below) + other Route 53 records |
| Query cost | Charged | **Free** (when pointing to AWS resources) |
| Client-visible answer | A `CNAME` (extra hop) | An `A`/`AAAA` (no extra hop) |
| Health-check awareness | No | Yes — can evaluate target health |

**Alias targets** you can point at: CloudFront distributions, ALB/NLB/CLB, S3 static website buckets, API Gateway, VPC interface endpoints, Global Accelerator, Elastic Beanstalk environments, and other records in the *same* hosted zone.

> **Rule**: To point a **naked/apex domain** at an AWS load balancer, CloudFront, or S3 website → use an **Alias**. For a subdomain pointing at a non-AWS name → a `CNAME` is fine. On the exam, "point the root domain `example.com` at the ALB" almost always means **Alias record**.

⚠️ You cannot Alias to an arbitrary external host (e.g. a third-party SaaS hostname) — Alias targets must be AWS resources or same-zone records. For external targets at a subdomain, use `CNAME`.

---

## 5. Routing Policies

A **routing policy** decides *which* record value Route 53 returns when multiple records share the same name. This is where Route 53 goes far beyond a normal DNS host.

| Policy | What it does | Use case / exam trigger |
|--------|-------------|--------------------------|
| **Simple** | One record, one (or more) values; returns it. If multiple values, returns all in random order. No health checks. | A single resource; the default. |
| **Weighted** | Splits traffic by assigned weights (e.g. 90/10). | A/B testing, **canary/blue-green** rollouts, gradually shifting traffic. |
| **Latency** | Returns the record for the AWS **Region** giving the user the lowest network latency. | Multi-Region apps where you want users on the *fastest* Region. ("Lowest latency" = this.) |
| **Failover** | Active/passive: returns primary while healthy, switches to secondary when the primary's health check fails. | DR / active-passive HA. ("Failover to a backup site" = this.) |
| **Geolocation** | Routes by the **user's physical location** (continent/country/state). | Compliance/localization — serve EU users from EU, block/redirect by geography, language-specific content. |
| **Geoproximity** | Routes by distance between user and resource, with an adjustable **bias** to expand/shrink a Region's traffic zone. | Shift traffic toward/away from a Region by a tunable radius. Requires Route 53 Traffic Flow. |
| **Multi-Value Answer** | Returns up to 8 healthy records at random; client picks one. **Health-checked** (unlike Simple). | Cheap client-side load balancing / improving availability — *not* a substitute for a real load balancer. |

> **Key insight — Latency vs Geolocation** (a favorite trap): **Latency** optimizes for *speed* (which Region is fastest for this user, measured by network performance). **Geolocation** routes by *where the user is* for *compliance/content* reasons. A user physically near a Region isn't guaranteed to have the lowest latency to it, so the two can route differently.

💡 **Geoproximity vs Geolocation**: Geo**location** = "what country are you in?" Geo**proximity** = "how far are you, and I can bias the boundaries." Geoproximity has the tunable **bias**; geolocation does not.

---

## 6. Health Checks and DNS Failover

A **health check** in Route 53 periodically probes a target and marks it healthy/unhealthy. Records can then route only to healthy targets — this is **DNS-level failover** (the third Route 53 job from §1). Three types:

| Type | Monitors | Notes |
|------|----------|-------|
| **Endpoint** | An IP or domain over HTTP/HTTPS/TCP | Checkers in multiple AWS Regions probe it; you set interval (10s/30s) and failure threshold. Can match a string in the response body. |
| **Calculated** | The combined status of **other health checks** | "Healthy if ≥ N of these child checks are healthy." Build composite logic. |
| **CloudWatch alarm** | A **CloudWatch alarm** state | Lets you health-check things with no public endpoint (e.g. private resources, DynamoDB throttling, queue depth) via a metric. |

How failover works end to end:

```
   Client asks: example.com?
            │
            ▼
   ┌───────────────────────────┐
   │ Route 53 (Failover policy)│
   │  PRIMARY  (us-east-1 ALB) │── health check ──► [HTTP 200 ✅] → return PRIMARY
   │  SECONDARY(us-west-2 ALB) │
   └───────────────────────────┘
            │  (primary health check now FAILS ❌)
            ▼
   Route 53 stops returning primary, returns SECONDARY instead.
   But clients holding a cached answer keep using it until TTL expires.
```

⚠️ **TTL gates failover speed.** Route 53 detects failure in seconds, but a client/resolver that cached the old answer won't re-query until the TTL expires. **Use a low TTL (e.g. 60s) on failover records** so the switch is felt quickly. This directly applies the TTL trade-off from the fundamentals file.

💡 Alias records can be set to **"Evaluate Target Health"** — Route 53 automatically uses the underlying AWS resource's health (e.g. ALB target health) without you wiring a separate health check.

---

## 7. TTL Behavior in Route 53

- You set TTL **per record** (e.g. 300s) for normal `A`/`CNAME`/etc. records.
- **Alias records to AWS resources do not let you set a TTL** — Route 53 uses the target's own default (and the answer is free). You manage cutover by changing the Alias target, not the TTL.
- For anything involved in **failover or planned migration**, choose a **low TTL** so changes propagate fast (at the cost of more queries). Lower it *before* a migration, as taught in §6 of the fundamentals file.

---

## 8. Worked Examples (Cross-Links)

These hands-on scenarios apply everything above:

- **[Public hosted zone](../18_practical_examples/07_route53_public_hosted_zone.md)** — register/host a public domain, wire records to an ALB via Alias.
- **[Private hosted zone](../18_practical_examples/08_route53_private_hosted_zone.md)** — internal-only names resolved inside a VPC (split-horizon DNS).
- **[Route 53 Resolver / hybrid DNS](../18_practical_examples/09_route53_resolver_hybrid_dns.md)** — inbound/outbound resolver endpoints so on-prem and AWS resolve each other's names.

---

## Key Exam Points

- Route 53 = **registrar + authoritative DNS + health checks**, with a **100% availability SLA**. It is not your apps' recursive resolver (that's Route 53 *Resolver*).
- **Public hosted zone** = internet-facing; **private hosted zone** = resolves only inside associated VPC(s); both can share a name (split-horizon).
- **Alias** records are free, work at the **apex**, return `A`/`AAAA`, and point at AWS resources (CloudFront, ALB/NLB, S3 website, API GW, GA). Apex → Alias; external subdomain → CNAME.
- Routing policies: **Simple, Weighted (canary/blue-green), Latency (fastest Region), Failover (active-passive DR), Geolocation (by location/compliance), Geoproximity (distance + bias), Multi-Value (up to 8 healthy, client-side LB).**
- Health checks: **endpoint, calculated, CloudWatch alarm**. CloudWatch-alarm type checks resources with no public endpoint.
- **Low TTL on failover records** so DNS failover is felt quickly; cached answers ignore the health check until TTL expires.

---

## Common Mistakes

- ❌ Using a `CNAME` for the apex domain. Use an **Alias** — this is *the* Route 53 exam pattern.
- ❌ Confusing **Latency** (fastest Region by network performance) with **Geolocation** (route by where the user physically is, for compliance/content).
- ❌ Thinking **Multi-Value Answer** replaces a load balancer. It's lightweight, health-checked client-side distribution, not a real LB.
- ❌ Expecting instant failover with a high TTL. Lower the TTL; clients cache old answers until it expires.
- ❌ Trying to Alias to a third-party hostname. Alias targets must be AWS resources or same-zone records; use `CNAME` for external names.
- ❌ Forgetting a private hosted zone needs `enableDnsSupport` + `enableDnsHostnames` on the VPC, or names won't resolve.

---

**Next**: [03_cloudfront.md — CloudFront: Content Delivery at the Edge](03_cloudfront.md)
