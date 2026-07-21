# Global Accelerator: Anycast Networking

> **Who this is for**: Engineers who've read [CloudFront](03_cloudfront.md) and now need to improve latency and failover for traffic that a CDN can't cache — APIs, gaming, IoT, VoIP, raw TCP/UDP. We start with the anycast concept (no AWS) before AWS Global Accelerator, then compare it head-to-head with CloudFront.

---

## 1. Prerequisite: Anycast IPs and the AWS Backbone

Normally an IP address lives in **one place** (this is **unicast** — one IP, one host). **Anycast** flips that: the *same* IP address is advertised from **many locations** at once, and the internet's routing automatically sends each user to the **nearest** one. Users in Tokyo and London can hit the *identical* IP and land on different, nearby entry points.

The second piece is the **AWS global backbone**: AWS's own private, high-capacity fiber network connecting its Regions and edge locations. Traffic on the backbone avoids the congestion, variable latency, and extra hops of the public internet.

```
   Public internet path              AWS backbone path
   ─────────────────                 ─────────────────
   user → ISP → many hops →          user → NEAREST edge (anycast IP)
   peering → congestion →                 → AWS private backbone (fast,
   eventually → app Region                  stable) → app Region
   (slow, jittery, unpredictable)         (low latency, low jitter)
```

> **Key insight**: Global Accelerator combines both ideas. It gives you **anycast IPs at the edge** so users enter the AWS network at the closest point, then carries their traffic over the **private backbone** to your application Region — instead of riding the open internet the whole way.

---

## 2. AWS Global Accelerator

**Global Accelerator (GA)** is a networking service that improves availability and performance for your applications by routing user traffic over the AWS backbone to the optimal healthy endpoint.

Key properties:

- **Two static anycast IPs** — GA gives you **2 fixed IP addresses** (from AWS's pool or your own BYOIP). They never change for the life of the accelerator. Clients/firewalls can whitelist them permanently. (Contrast: an ALB's DNS name resolves to changing IPs.)
- **Edge entry + backbone transit** — users hit the nearest edge location via the anycast IPs, then travel the private backbone to the chosen Region.
- **Routes to the nearest *healthy* endpoint** — endpoints are ALBs, NLBs, EC2 instances, or Elastic IPs grouped into **endpoint groups** per Region. GA health-checks them and routes around failures.
- **Fast regional failover** — because the IPs are static and failover happens at the network layer (no DNS/TTL involved), recovery is **seconds**, not minutes. This is the big advantage over DNS failover.
- **Works for TCP and UDP — any protocol, not just HTTP** — gaming, VoIP, IoT, MQTT, financial protocols. **No caching** is involved; GA is a fast, smart network path, not a content cache.
- **Traffic dials & weights** — shift percentages of traffic between endpoint groups (Regions) for blue-green or gradual migration.

```
        ┌──────────────────────────────────────────┐
        │ 2 static anycast IPs (e.g. 75.x / 99.x)  │
        └───────────────┬──────────────────────────┘
   user ───────────────►│ nearest AWS edge location
                        │
                        ▼  AWS private backbone
        ┌───────────────────────────┐   ┌───────────────────────────┐
        │ Endpoint group us-east-1  │   │ Endpoint group eu-west-1  │
        │  ALB / NLB / EC2 / EIP    │   │  ALB / NLB / EC2 / EIP    │
        │  (health-checked)         │   │  (health-checked)         │
        └───────────────────────────┘   └───────────────────────────┘
              ▲ routes to NEAREST HEALTHY group; fails over in seconds
```

💡 Because failover is at the IP/network layer, there is **no client-side DNS cache to wait on** — this is precisely the limitation of Route 53 failover (TTL) that GA sidesteps.

---

## 3. CloudFront vs Global Accelerator

They both use AWS edge locations, which is why they're confused. But they solve **different** problems.

| | **CloudFront** | **Global Accelerator** |
|---|----------------|-------------------------|
| Primary purpose | **Cache & deliver content** at the edge | **Route & accelerate** traffic over the backbone |
| Caches content? | ✅ Yes (its whole point) | ❌ No — proxies/forwards, never caches |
| Protocols | **HTTP/HTTPS only** | **Any TCP/UDP** (HTTP, gaming, VoIP, IoT, MQTT…) |
| Entry address | A distribution domain (`*.cloudfront.net`), fronted by DNS | **2 static anycast IPs** (whitelistable, never change) |
| Failover speed | Origin failover, but content/edge-centric | **Seconds**, at the network layer (no DNS TTL wait) |
| Edge compute | CloudFront Functions / Lambda@Edge | None |
| Typical use | Websites, static assets, media, APIs you can cache | Non-HTTP apps, gaming/VoIP/IoT, multi-Region failover, fixed-IP requirements |
| Security add-ons | WAF, Shield, signed URLs, geo-restriction | Shield (DDoS); WAF only via a downstream ALB |

> **Rule of thumb**:
> - **Cacheable web content / HTTP(S), global users** → **CloudFront**.
> - **Non-HTTP (TCP/UDP), need static IPs, or need seconds-fast multi-Region failover** → **Global Accelerator**.
> - You can even put **CloudFront in front for caching and GA for the dynamic/non-HTTP paths** — they're complementary, not mutually exclusive.

⚠️ Exam triggers for **Global Accelerator** specifically: "**two static IP addresses**," "**UDP / non-HTTP / gaming / VoIP / IoT**," "**fast regional failover without DNS changes**," "**whitelist a fixed IP**." Triggers for **CloudFront**: "**cache**," "**static website / media**," "**HTTPS for a static site**," "**reduce latency for downloads**."

---

## 4. Traffic Engineering and Recovery

A standard accelerator has endpoint groups, normally one per Region. Two knobs
serve different purposes:

- An endpoint **weight** changes the proportion sent to healthy endpoints within
  one endpoint group. Use it for canaries, blue/green targets, or unequal
  regional capacity.
- An endpoint-group **traffic dial** changes how much of that Region's normal
  traffic the group accepts. Use it to drain a Region or ramp it up gradually.

Health still wins over both knobs: do not send a canary to a target whose health
check cannot represent the application. Track connection errors, target health,
regional capacity, and application-level success during every shift.

**Client affinity** can keep connections from the same source IP on the same
endpoint where supported. It helps protocols that retain local state, but users
behind a shared NAT can appear as one client and a health event can still move
them. Prefer externalized state; affinity is not a consistency mechanism.

### Standard vs custom routing

| Accelerator | Behavior | Use it for |
|-------------|----------|------------|
| **Standard** | Chooses a healthy regional endpoint for each client flow | Public applications needing performance, static IPs, and health-based regional failover |
| **Custom routing** | Deterministically maps accelerator listener ports to specific EC2 destinations in VPC subnets | Large session-based systems, such as games, that assign users to a particular server/port |

Custom routing is not a more configurable standard accelerator. The application
controls destination assignment and must handle server availability, port
mappings, and session recovery.

### Test failover without guessing

Before a game day, confirm quotas and spare capacity in the recovery Region.
Drain or set weight to zero for maintenance, then simulate endpoint failure and
observe existing and new connections separately. UDP sessions and long-lived TCP
connections may need application retries even though new flows move quickly.
Test restoration and failback as carefully as failover to prevent overload or
split-brain writes.

**Bring Your Own IP (BYOIP)** can preserve an organization-owned address range
and advertise it through Global Accelerator where supported. Choose it for an
existing allow-list or address-ownership requirement, not merely to avoid DNS.
AWS-provided accelerator IPs already remain static for the accelerator.

Cost includes accelerator-hours and data transfer through Global Accelerator,
plus the regional endpoint stack. Compare it with Route 53 when DNS caching and
client reconnection meet the RTO, and with CloudFront when HTTP caching can avoid
origin traffic. GA earns its cost when static IPs, TCP/UDP support, or rapid
health-based flow placement is a business requirement.

---

## Key Exam Points

- **Anycast** = one IP advertised from many places; users reach the nearest. GA pairs this with the **AWS private backbone**.
- GA gives **2 static anycast IPs** that never change — ideal for IP whitelisting and as a stable front for changing backends.
- GA routes to the **nearest healthy endpoint** (ALB/NLB/EC2/EIP in endpoint groups) and fails over **in seconds** at the network layer — **no DNS TTL** to wait on.
- GA supports **any TCP/UDP protocol** and **does not cache**. CloudFront is **HTTP(S) only** and **does cache**.
- Pick **CloudFront** for cacheable HTTP content; pick **GA** for non-HTTP apps, static-IP needs, or fast multi-Region failover. They can be combined.
- Endpoint **weights** divide traffic within a Region; **traffic dials** ramp a regional endpoint group up or down.
- **Custom routing** provides deterministic port-to-EC2 mapping; standard accelerators provide health-based application routing.
- Test new and existing connections, regional headroom, and failback; fast network routing does not replicate state.

---

## Common Mistakes

- ❌ Thinking Global Accelerator caches content. It never does — it accelerates/routes; CloudFront caches.
- ❌ Choosing CloudFront for a UDP/TCP gaming or VoIP workload. CloudFront is HTTP/HTTPS only → use **GA**.
- ❌ Reaching for Route 53 failover when the requirement is **failover in seconds without DNS propagation**. That's a **GA** signal (static IPs, network-layer failover).
- ❌ Assuming GA gives you edge compute or WAF directly. No CloudFront Functions/Lambda@Edge; WAF only via a downstream ALB.
- ❌ Confusing the two just because both use AWS edge locations. **Caching + HTTP = CloudFront; routing + any protocol + static IPs = GA.**
- ❌ Treating client affinity as durable session storage, especially when many users share a NAT source address.
- ❌ Using GA because it sounds faster without comparing its processing cost with Route 53 or CloudFront against the actual RTO and caching need.

---

**Next**: [05_route53_resolver.md — Route 53 Resolver: VPC Resolver, Endpoints, DNS Firewall](05_route53_resolver.md)
