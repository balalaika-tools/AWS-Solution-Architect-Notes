# Route 53 Resolver: The VPC Resolver, Endpoints, Rules, DNS Firewall & Query Logging

> **Who this is for**: Engineers who know Route 53 the *authoritative* service ([02_route_53.md](02_route_53.md)) and now need the *other* Route 53 — the **recursive resolver** that your EC2 instances actually query. This file is the conceptual home for the **`.2` Amazon resolver**, **inbound/outbound endpoints**, **resolver rules**, **DNS Firewall**, and **query logging**. The two hands-on walkthroughs ([hybrid DNS](../18_practical_examples/09_route53_resolver_hybrid_dns.md), [peering DNS](../18_practical_examples/06_vpc_peering_dns_resolution.md)) apply what's here.
>
> Prerequisites: [01_dns_fundamentals.md](01_dns_fundamentals.md) (recursive vs authoritative, the resolution flow) and [02_route_53.md](02_route_53.md) (hosted zones, public vs private).

---

## 1. Authoritative ≠ Resolver — Two Different "Route 53"s

The single most important distinction, and a classic exam trip-up. AWS reuses the "Route 53" brand for **two unrelated jobs**:

| | **Route 53 (authoritative DNS)** | **Route 53 Resolver** |
|---|----------------------------------|------------------------|
| Role in DNS | **Authoritative** name server — *holds* your zone's records | **Recursive resolver** — *looks up* answers on behalf of clients |
| Who uses it | The whole internet resolving `example.com` | **Your EC2 instances** inside a VPC |
| Covered in | [02_route_53.md](02_route_53.md) | **this file** |
| Address | A global, distributed NS fleet | The **`.2` address** inside each VPC |

Recall from [01_dns_fundamentals.md](01_dns_fundamentals.md §3): a **recursive resolver** is the workhorse that chases referrals (root → TLD → authoritative) and caches answers; the **authoritative** server is the source of truth. In a VPC, the recursive resolver your instances use **is** the Route 53 Resolver.

> **Key insight**: When your app calls `8.8.8.8` from a laptop, Google runs the recursive resolver. When an EC2 instance resolves a name, **the Route 53 Resolver (the `.2` address)** is that recursive resolver — and it's on by default, no setup.

---

## 2. The `.2` Amazon-Provided Resolver (AmazonProvidedDNS)

Every VPC ships with a built-in recursive resolver — historically called **AmazonProvidedDNS**, now the **Route 53 Resolver**. It lives at two addresses:

- **`VPC_CIDR_base + 2`** — e.g. in `10.0.0.0/16` the resolver is `10.0.0.2`. (AWS reserves the first four IPs of every subnet; `.2` is the resolver.)
- **`169.254.169.253`** — a fixed **link-local** address that works regardless of the VPC CIDR (handy when you don't want to compute base+2).

```
   EC2 (10.0.1.50)
        │  "resolve api.example.com / db.internal / s3.amazonaws.com"
        ▼
   ┌──────────────────────────────┐
   │ Route 53 Resolver  (the .2)  │   = 10.0.0.2  or  169.254.169.253
   │  • recurses public names     │── public internet names ──► authoritative NS
   │  • answers VPC private hosts │── *.ec2.internal / *.compute.internal
   │  • answers associated PHZs   │── db.internal  (Private Hosted Zone)
   └──────────────────────────────┘
```

What the `.2` resolver answers, for free, with zero configuration:

1. **Public DNS** — recurses normal internet names (`s3.amazonaws.com`, `example.com`).
2. **Internal EC2 hostnames** — `ip-10-0-1-50.ec2.internal` (us-east-1) / `*.compute.internal` (other Regions).
3. **Private Hosted Zones** associated with the VPC — `db.internal` (see [02_route_53.md §2](02_route_53.md)).

> **Rule**: For the `.2` resolver to work, the VPC needs **`enableDnsSupport = true`** (the resolver itself) and, for instances to *get* resolvable hostnames, **`enableDnsHostnames = true`**. `enableDnsHostnames` defaults to **false** in custom VPCs — a frequent "names won't resolve" cause (detailed in [06_vpc_peering_dns_resolution.md](../18_practical_examples/06_vpc_peering_dns_resolution.md)).

⚠️ The `.2` resolver is reachable **only from inside its own VPC**. It can't be queried from on-prem, and it doesn't natively know about on-prem zones. Bridging that gap is what **endpoints** (§3) are for.

---

## 3. Resolver Endpoints — Crossing the VPC Boundary

The `.2` resolver is sealed inside the VPC. **Resolver endpoints** are ENIs (with private IPs, in 2+ AZs) that let DNS cross between on-prem and AWS over VPN/Direct Connect. There are two directions:

| Endpoint | Direction | Lets… | Mental model |
|----------|-----------|-------|--------------|
| **Inbound** | on-prem → AWS | on-prem DNS resolve **AWS** private names (PHZs) | "a door *in* to the `.2` resolver" |
| **Outbound** | AWS → on-prem | EC2 resolve **on-prem** names | "a door *out*, paired with forwarding rules" |

```
  ON-PREM                          AWS VPC
  corp DNS ──forward "internal"──► [INBOUND endpoint ENIs] ──► .2 resolver / PHZ
  corp DNS ◄──forward "corp.local"─ [OUTBOUND endpoint ENIs] ◄── .2 resolver (via rule)
```

⚠️ These are **DNS resolver endpoints** — *not* VPC interface/gateway endpoints (those give private connectivity to S3, DynamoDB, or AWS APIs; see [VPC Peering & Endpoints](../03_networking/05_vpc_peering_and_endpoints.md)). Same word, different feature.

> The full build (CLI, security groups, BIND forwarder config, both directions, troubleshooting) lives in **[09_route53_resolver_hybrid_dns.md](../18_practical_examples/09_route53_resolver_hybrid_dns.md)**. This section is just the concept.

---

## 4. Resolver Rules — Which Domains Get Forwarded Where

A **resolver rule** tells the `.2` resolver what to do with a query for a given domain. An **outbound endpoint** without rules does nothing; the rule is what says "for `corp.local`, forward to these on-prem DNS IPs." Rules are then **associated with VPCs**, and can be **shared across accounts via AWS RAM**.

| Rule type | What it does | When you see it |
|-----------|-------------|------------------|
| **Forward** | For domain X, **forward** queries to specified target DNS IPs (e.g. on-prem) via an outbound endpoint. | The hybrid-DNS workhorse: `corp.local → 192.168.1.53`. |
| **System** | **Override** a more general forward rule for a subdomain — forces it back to the default Resolver behavior (don't forward). | Carve an exception out of a broad forward rule (e.g. forward `corp.local` but keep `aws.corp.local` resolving via AWS). |
| **Recursive** (autodefined / default) | The Resolver's built-in behavior for everything not matched by a rule — recurse public names, answer PHZs and internal hostnames. | The default; you don't create these. |

> **Key insight — most specific wins**: rules match the **most specific** domain. A **System** rule on `aws.corp.local` beats a **Forward** rule on `corp.local`. Use System rules to punch holes in a broad forward.

⚠️ **Never forward the root (`.`)** or a domain the Resolver should answer (a PHZ domain) out to on-prem — you black-hole all resolution. Forward **only** the specific external zones.

---

## 5. Route 53 Resolver DNS Firewall — Filtering Outbound DNS

**DNS Firewall** filters the domains your VPC's resources are allowed to *resolve*. It inspects queries flowing through the `.2` resolver and applies allow/block decisions from **domain lists**. This is a **security control**, not a routing feature.

```
   EC2 ──"resolve evil-c2.malware.example"──► .2 Resolver
                                                  │
                                       ┌──────────▼───────────┐
                                       │  DNS Firewall rule   │
                                       │  group (ordered)     │
                                       │   match domain list? │
                                       └──────────┬───────────┘
                                  BLOCK ◄─────────┤  (NODATA / NXDOMAIN / custom)
                                  ALERT  (log only, allow)
                                  ALLOW  (explicit pass)
```

Core pieces:

- **Domain lists** — the names to match. Either **your own** lists (e.g. block `*.coinhive.com`) or **AWS Managed Domain Lists** (curated feeds of known malware, botnet C2, and crypto-mining domains).
- **Rule group** — an **ordered** set of rules, each pairing a domain list with an action. Lower priority number evaluates first; first match wins. The rule group is then **associated with one or more VPCs**.
- **Actions**: **`ALLOW`** (explicit pass), **`ALERT`** (allow but log — great for a monitor-first rollout), **`BLOCK`** (with response `NODATA`, `NXDOMAIN`, or a custom `CNAME` override).

**Fail-open vs fail-closed**: the VPC's DNS Firewall config has a **fail mode**. **Fail-open** keeps resolving if the firewall can't be evaluated (availability over security); **fail-closed** blocks (security over availability). Know this exists.

> **Exam trigger**: "**prevent instances from resolving malicious domains / stop DNS-based data exfiltration**" → **Route 53 Resolver DNS Firewall**. It is *not* a security group, NACL, or WAF job — those don't see DNS query *names*. WAF inspects HTTP; DNS Firewall inspects DNS.

💡 Pair DNS Firewall with **GuardDuty**, which independently flags suspicious DNS lookups (e.g. to known-bad domains) from VPC DNS logs — detection (GuardDuty) + prevention (DNS Firewall).

---

## 6. Resolver Query Logging — Audit Every Lookup

**Resolver query logging** records the DNS queries originating from a VPC: the name queried, type, response, and which instance asked. You attach a **query log config** to a VPC and send logs to one of three destinations:

| Destination | Use case |
|-------------|----------|
| **CloudWatch Logs** | Search/alarm on query patterns in near-real-time. |
| **S3** | Cheap long-term retention / archival for audit. |
| **Amazon Data Firehose** | Stream to a SIEM or data lake for analysis. |

> **Exam trigger**: "**audit / get visibility into what domains your instances are looking up**" → **Resolver query logging**. It answers *"what was resolved"*; DNS Firewall (§5) *controls* what's allowed. They complement each other.

⚠️ Query logging captures queries the `.2` Resolver handles. It is **not** VPC Flow Logs (which record IP traffic, not DNS names) — a common mix-up.

---

## 7. Putting It Together — Which Feature Solves What

| You need… | Feature | Notes |
|-----------|---------|-------|
| EC2 to resolve public + internal names | **`.2` Resolver** (default) | Nothing to configure beyond VPC DNS attributes. |
| On-prem to resolve AWS PHZ names | **Inbound endpoint** | on-prem forwards to the endpoint ENIs (§3). |
| EC2 to resolve on-prem names | **Outbound endpoint + Forward rule** | rule associated with the VPC; RAM-shareable (§4). |
| Stop instances resolving malicious/blocked domains | **DNS Firewall** | managed or custom domain lists; ALERT→BLOCK rollout (§5). |
| Audit what domains instances resolve | **Query logging** | to CloudWatch / S3 / Data Firehose (§6). |
| Private name across peered VPCs | **PHZ association** (not a resolver feature) | see [06_vpc_peering_dns_resolution.md](../18_practical_examples/06_vpc_peering_dns_resolution.md). |

---

## Key Exam Points

- ✅ **Route 53 Resolver = the recursive resolver inside a VPC**, at **`VPC_CIDR_base + 2`** and the link-local **`169.254.169.253`**. Distinct from Route 53 *authoritative* DNS.
- ✅ The `.2` resolver answers **public names, internal EC2 hostnames, and associated Private Hosted Zones** for free; needs **`enableDnsSupport`** (+ **`enableDnsHostnames`** for hostnames).
- ✅ **Inbound endpoint** = on-prem → AWS (resolve PHZs); **Outbound endpoint + Forward rule** = AWS → on-prem. Endpoints are **ENIs in 2+ AZs**; rules are **VPC-associated and RAM-shareable**.
- ✅ **Resolver rule types**: **Forward** (send a domain to specific DNS IPs), **System** (override/exempt a subdomain back to default), **Recursive** (the default). Most specific match wins.
- ✅ **DNS Firewall** filters which domains a VPC can resolve via **domain lists** (custom or **AWS Managed**) + **rule groups** with **ALLOW/ALERT/BLOCK**. The answer to "block malicious domain resolution / DNS exfiltration."
- ✅ **Resolver query logging** → **CloudWatch / S3 / Data Firehose** for auditing *what* was resolved (≠ VPC Flow Logs).
- ⚠️ Never forward `.` or a PHZ domain outbound — it black-holes resolution.

---

## Common Mistakes

- ❌ Confusing **Route 53 authoritative** (holds records, serves the internet) with the **Resolver** (the `.2`, recurses for your instances). Different services, same brand.
- ❌ Thinking **resolver endpoints** are the same as **VPC interface/gateway endpoints**. One forwards DNS; the other gives private connectivity to AWS services.
- ❌ Reaching for **security groups / NACLs / WAF** to block domain resolution. They don't inspect DNS query names — that's **DNS Firewall**.
- ❌ Using **VPC Flow Logs** to see what domains were queried. Flow Logs show IPs/ports, not names — use **Resolver query logging**.
- ❌ Creating an outbound endpoint but **no forward rule**, or a rule **not associated** with the VPC — then wondering why nothing forwards.
- ❌ Forwarding the root domain `.` outbound, black-holing all of the VPC's DNS.

---

**Next**: [../09_containers/01_container_fundamentals.md — Container Fundamentals](../09_containers/01_container_fundamentals.md)
