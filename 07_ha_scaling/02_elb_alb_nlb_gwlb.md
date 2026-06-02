# The ELB Family: ALB, NLB & GWLB

> **Who this is for**: Engineers prepping for SAA-C03 who know *why* load balancers exist and now need to pick the right AWS one. Networking-light readers welcome — TLS termination, SNI, and GENEVE are explained inline. Assumes you've read [01_load_balancing_concepts.md](01_load_balancing_concepts.md) (listeners, target groups, L4 vs L7).

---

## 1. What "ELB" Means

**Elastic Load Balancing (ELB)** is the AWS umbrella service. Under it live **four** load-balancer types. One is legacy; three are current and the heart of the exam.

```
                  Elastic Load Balancing (ELB)
                              │
      ┌───────────┬───────────────────────┬──────────────┐
      ▼           ▼                       ▼              ▼
   Classic     Application             Network        Gateway
   (CLB)         (ALB)                  (NLB)          (GWLB)
  LEGACY       Layer 7                Layer 4         Layer 3
  avoid        HTTP/HTTPS            TCP/UDP/TLS      appliances
```

> **Key insight**: AWS split the old "do everything" Classic Load Balancer into purpose-built products. ALB owns smart HTTP routing, NLB owns raw TCP/UDP performance, GWLB owns transparent traffic inspection. Knowing which job each does is most of the exam.

All of them are **regional, managed, multi-AZ** services — you don't run or patch the underlying boxes.

---

## 2. Classic Load Balancer (CLB) — Legacy

The original (2009-era) ELB. It straddles Layer 4 and a limited Layer 7 but does neither well by modern standards.

- Supports HTTP/HTTPS and raw TCP, but **no host/path routing**, no target groups (uses a flat pool of instances), no IP or Lambda targets.
- Works with the older **EC2-Classic** networking model.

> **Rule**: For any new design, **never choose CLB**. If a question offers CLB alongside ALB/NLB for a greenfield workload, CLB is a distractor. AWS recommends migrating CLBs to ALB or NLB. The only time CLB is the "right" answer is a legacy-migration question that explicitly mentions an existing Classic Load Balancer or EC2-Classic.

⚠️ CLB is the only ELB type that supports a **single certificate** with no SNI and uses pre-VPC concepts. Treat it as historical context only.

---

## 3. Application Load Balancer (ALB) — Layer 7

The ALB operates at **Layer 7**: it parses each HTTP/HTTPS request and routes on its contents. This is the default choice for web applications and microservices.

```
   HTTPS request: GET app.example.com/api/orders
                          │
   ┌────────────── ALB (HTTPS:443) ───────────────┐
   │  Host = api.example.com  ──▶ TG "api"        │
   │  Path = /images/*        ──▶ TG "static"     │
   │  Header X-Beta = true    ──▶ TG "canary"     │
   │  default                 ──▶ TG "web"        │
   └──────────────────────────────────────────────┘
```

**What the ALB can do that lower-layer LBs cannot:**

| Capability | What it gives you |
|------------|-------------------|
| **Host-based routing** | Route by `Host:` header — many domains on one ALB |
| **Path-based routing** | `/api/*` → one fleet, `/static/*` → another |
| **Header / method / query routing** | Canary releases, A/B, mobile vs web splits |
| **Target groups** | Instance, **IP**, or **Lambda** targets |
| **WebSockets & HTTP/2 / gRPC** | Long-lived bidirectional connections |
| **Redirects** | HTTP→HTTPS redirect at the LB, no app code |
| **Fixed responses** | Return a canned 503/maintenance page directly |
| **Authentication** | Built-in OIDC / Amazon Cognito auth before traffic reaches targets |
| **Sticky sessions** | Cookie-based (`AWSALB` or app cookie) |

💡 The ALB's **authentication integration** is exam-worthy: you can offload login (Cognito user pools or any OIDC provider) to the ALB so unauthenticated requests never reach your instances.

✅ Use an ALB for: web apps, REST APIs, microservices behind path/host routing, containers (ECS/Fargate), serverless behind HTTP (Lambda targets).

❌ Don't use an ALB for: non-HTTP protocols (raw TCP/UDP databases, game servers), workloads needing a fixed/static IP, or millions of requests/sec at the lowest possible latency.

---

## 4. Network Load Balancer (NLB) — Layer 4

The NLB operates at **Layer 4**: it forwards TCP, UDP, and TLS connections without understanding their contents. It's built for extreme performance and protocol flexibility.

```
   TCP / UDP / TLS connection
            │
   ┌──────── NLB (TCP:443) ─────────┐
   │  one STATIC IP per AZ          │
   │  (or attach an Elastic IP)     │
   │  forwards connection, preserves│
   │  the client's SOURCE IP        │
   └────────────────┬───────────────┘
                    ▼
              target group (instance / IP / ALB)
```

**NLB's distinguishing features:**

| Feature | Detail |
|---------|--------|
| **Static IP per AZ** | NLB gives a fixed IP in each AZ; you can attach your own **Elastic IP** |
| **Ultra-low latency** | No HTTP parsing; minimal added latency |
| **Millions of req/s** | Scales to extreme throughput with no pre-warming |
| **Protocols** | TCP, UDP, TCP_UDP, and **TLS** |
| **Source IP visibility** | Often preserves the client IP, but behavior depends on target type/protocol/settings; Proxy Protocol v2 is available when targets need source details |
| **TLS termination** | Can terminate TLS at the NLB (offload from targets) |
| **Targets** | Instance, IP, or **an ALB** (NLB→ALB for static IP + L7 routing) |

> **Key insight**: Choose NLB when a question mentions any of: **static IP / Elastic IP**, **TCP or UDP**, **extreme performance / millions of requests**, **lowest latency**, or **preserving the client source IP** (e.g., for an IP allow-list on the backend).

✅ Use an NLB for: high-throughput TCP/UDP services, game servers, IoT, financial systems needing low latency, anything needing a whitelisted static IP, or fronting an ALB to give it a stable IP.

❌ Don't use an NLB when you need to route on URL path/host or do HTTP redirects/auth — it can't read HTTP. That's ALB territory.

⚠️ NLB stickiness is by **source IP**; there are no cookies at Layer 4.

---

## 5. Gateway Load Balancer (GWLB) — Layer 3

The GWLB is the odd one out. It exists to insert **third-party virtual appliances** — firewalls, intrusion detection/prevention (IDS/IPS), deep packet inspection — **transparently into the traffic path**, then scale and load-balance that fleet of appliances.

```
   Internet ──┐
              ▼
        ┌──────────────┐   GENEVE-encapsulated traffic
        │    GWLB       │ ─────────────────────────────▶ ┌─────────────┐
        │ (Layer 3)     │ ◀───────────────────────────── │  Firewall   │
        └──────┬────────┘    inspected traffic returns   │ appliances  │
               │                                         │ (scaled)    │
               ▼                                         └─────────────┘
        application VPC
```

- Operates at **Layer 3** (IP packets). It's a "bump in the wire" — traffic is routed *through* the appliance fleet for inspection, then on to its destination.
- Uses the **GENEVE protocol on port 6081** to encapsulate traffic to and from the appliances, preserving the original packet.
- Pairs with a **Gateway Load Balancer Endpoint (GWLBe)** so other VPCs can send traffic through the inspection fleet via routing.
- Provides one entry point and load-balances across many appliance instances, scaling them like any other target group.

> **Rule**: GWLB is the answer **only** when the question is about deploying/scaling **virtual network appliances** (third-party firewalls, IDS/IPS, packet inspection). The keyword is **GENEVE** or "transparent inspection of all traffic." For ordinary application traffic, it's never the answer.

---

## 6. Network Placement, ENIs, and Security Groups

Load balancers are VPC resources that create managed network interfaces in the
subnets/AZs you enable. Targets should run in AZs enabled on the load balancer,
or they might not receive traffic/health checks the way you expect.

| Load balancer | Subnet placement | Security group behavior |
|---------------|------------------|-------------------------|
| **Internet-facing ALB** | Public subnets in 2+ AZs | Has SGs. Allow client ports inbound; allow target and health-check ports outbound. |
| **Internal ALB** | Private subnets in 2+ AZs | Has SGs. Usually allow from app/client SGs or VPC CIDRs. |
| **NLB** | Public or private subnets, one static address per enabled AZ | Can have SGs **only if associated at creation**. If created without SGs, you cannot add them later. Target SGs can reference the NLB SG. |
| **GWLB** | Subnets for appliance traffic path | You cannot associate an SG with a GWLB. Put SGs on appliance targets and use route tables/NACLs. |

Common web-tier SG chain:

```
ALB SG:      inbound 443 from clients, outbound app/health ports to target SG
Target SG:   inbound app/health ports from ALB SG
```

For private ECS/Fargate tasks behind an ALB, the target group type is usually
`ip`, because each task has its own ENI and private IP.

For the broader resource map, see
[ENIs, Security Groups & Service Networking](../03_networking/07_enis_security_groups_and_service_networking.md).

---

## 7. The Big Comparison Table

| | **Classic (CLB)** | **Application (ALB)** | **Network (NLB)** | **Gateway (GWLB)** |
|---|---|---|---|---|
| **OSI layer** | 4 & basic 7 | **7** | **4** | **3** |
| **Protocols** | HTTP, HTTPS, TCP, SSL | HTTP, HTTPS, gRPC, WebSocket | TCP, UDP, TCP_UDP, TLS | IP / GENEVE (6081) |
| **Routing on path/host** | ❌ | ✅ | ❌ | ❌ |
| **Target types** | EC2 instances only | Instance, IP, **Lambda** | Instance, IP, **ALB** | Appliance instances/IPs |
| **Static IP / Elastic IP** | ❌ | ❌ (DNS name only) | ✅ **yes** | via endpoint |
| **TLS/SSL termination** | ✅ | ✅ | ✅ | n/a (pass-through) |
| **SNI (multi-cert)** | ❌ | ✅ | ✅ | n/a |
| **Client source info** | client IP via headers | via `X-Forwarded-For` header | Native or Proxy Protocol v2 depending on target/protocol/settings | ✅ |
| **WebSockets / HTTP/2** | ❌ | ✅ | (TCP passthrough) | n/a |
| **Performance** | moderate | high | **extreme (M req/s, µs latency)** | scales with appliances |
| **Use case** | legacy only | web apps, APIs, microservices, containers | TCP/UDP, low latency, static IP | firewalls / IDS/IPS appliances |
| **Greenfield?** | ❌ avoid | ✅ | ✅ | ✅ (appliance use only) |

---

## 8. Cross-Zone Load Balancing

By default, each LB node lives in one AZ and (historically) only distributed traffic to targets in its **own** AZ. **Cross-zone load balancing** lets every LB node distribute evenly across targets in **all** AZs.

```
   WITHOUT cross-zone:               WITH cross-zone:
   AZ-a node ─▶ only AZ-a targets    AZ-a node ─▶ targets in AZ-a + AZ-b
   AZ-b node ─▶ only AZ-b targets    AZ-b node ─▶ targets in AZ-a + AZ-b

   Risk: uneven if AZs have          Result: even 1/N split across
   different target counts           all healthy targets
```

| LB type | Cross-zone default | Data charge for cross-AZ? |
|---------|--------------------|----------------------------|
| **ALB** | **Always on** (can't disable at LB level) | **No** inter-AZ charge |
| **NLB** | **Off** by default (can enable) | **Yes** — enabling incurs inter-AZ data charges |
| **GWLB** | Off by default (can enable) | Yes if enabled |

⚠️ This is a classic exam trap: **ALB cross-zone is free and always on; NLB charges for it and it's off by default.** If targets are unevenly distributed across AZs behind an NLB and load looks lopsided, the fix is to enable cross-zone load balancing.

---

## 9. SSL/TLS Termination and SNI

**TLS termination** means the load balancer decrypts incoming HTTPS/TLS and (optionally) re-encrypts to the targets. The LB holds the certificate; targets can receive plain HTTP, saving them the crypto work.

```
   Client ══HTTPS/TLS══▶ LB ──HTTP──▶ targets        (TLS terminated at LB)
                         │
                    holds the cert
                    (often from ACM)
```

- Certificates come from **AWS Certificate Manager (ACM)** — free public certs, auto-renewed.
- The LB can also do **end-to-end encryption** by re-encrypting to targets (re-establishing TLS to the backend).

**SNI (Server Name Indication)** lets one listener serve **multiple certificates** for multiple domains on the same IP/port. The client tells the LB which hostname it wants during the TLS handshake, and the LB presents the matching cert.

| | TLS termination | SNI |
|---|---|---|
| **What** | LB decrypts TLS, offloads from targets | One listener, many certs/domains |
| **Supported by** | CLB, ALB, NLB | **ALB, NLB** (not CLB) |
| **Why it matters** | Targets don't manage certs | Host many HTTPS sites on one LB |

💡 SNI is why a single ALB can serve `shop.example.com` and `api.example.com` on the same `:443` listener, each with its own certificate.

---

## 10. Decision Guide: ALB vs NLB vs GWLB

```
   Are you inserting a firewall / IDS / IPS appliance?
   │
   ├─ YES ──────────────────────────────▶  GWLB
   │
   └─ NO
        │
        Is the traffic HTTP/HTTPS and do you need routing
        by path/host, redirects, auth, or Lambda targets?
        │
        ├─ YES ─────────────────────────▶  ALB
        │
        └─ NO  (raw TCP/UDP, static IP needed,
                extreme perf, preserve source IP)
                                          ──▶  NLB
```

| If the requirement is… | Choose |
|------------------------|--------|
| Route by URL path or hostname | **ALB** |
| HTTP→HTTPS redirect or canned response without app code | **ALB** |
| Cognito / OIDC authentication at the edge | **ALB** |
| Lambda as a target | **ALB** |
| A **static IP** or **Elastic IP** for an allow-list | **NLB** |
| TCP, UDP, or extreme throughput / lowest latency | **NLB** |
| Preserve the **client source IP** to the backend | **NLB** |
| Give an ALB a stable IP | **NLB in front of ALB** |
| Scale third-party firewall / inspection appliances (GENEVE) | **GWLB** |
| It's a brand-new design | **never CLB** |

See a full worked build of public ingress to a private fleet: [ALB → Private EC2](../18_practical_examples/03_alb_to_private_ec2.md).

---

## 11. Key Exam Points

- **ALB = Layer 7 / HTTP(S)**; routes on **path, host, header**; supports **Lambda** targets, redirects, fixed responses, and **Cognito/OIDC auth**.
- **NLB = Layer 4 / TCP-UDP-TLS**; **static IP & Elastic IP**, **millions of req/s, µs latency**, **preserves source IP**. Source-IP stickiness only.
- **GWLB = Layer 3**, for **virtual appliances** (firewalls/IDS/IPS), uses **GENEVE on port 6081**.
- **ALB has SGs; NLB can have SGs if attached at creation; GWLB cannot have SGs.**
- Target SGs should usually allow from the ALB/NLB SG, not from the whole internet.
- **CLB is legacy** — only correct in an explicit migration/EC2-Classic scenario.
- **ALB cross-zone: always on, free. NLB cross-zone: off by default, costs inter-AZ data.**
- **ACM** provides free auto-renewing certs for TLS termination; **SNI** (ALB/NLB) serves many certs on one listener.
- **NLB can target an ALB** to combine a static IP with Layer-7 routing.

---

## 12. Common Mistakes

- **❌ Choosing ALB when a static IP is required.** ALB only gives a DNS name; static/Elastic IP needs **NLB**.
- **❌ Choosing NLB for path/host routing.** Layer 4 can't read URLs — use **ALB**.
- **❌ Picking GWLB for normal app traffic.** GWLB is for inspection appliances only.
- **❌ Forgetting NLB cross-zone is off by default**, then being surprised by uneven load across AZs.
- **❌ Selecting CLB for a new build** because it "does both layers." It's legacy; pick ALB or NLB.
- **❌ Assuming targets see the client IP behind an ALB.** ALB inserts `X-Forwarded-For`; the connection's source is the ALB. NLB preserves the real source IP.
- **❌ Creating an NLB with no security group and expecting to add one later.** Associate SGs at creation if you need them.
- **❌ Opening target SGs to `0.0.0.0/0` behind a public ALB.** Allow inbound from the ALB SG.

---

## 13. Limits & Defaults

| Item | Value |
|------|-------|
| ALB cross-zone | On (always), free |
| NLB / GWLB cross-zone | Off by default; inter-AZ data charges when on |
| NLB static IP | One per enabled AZ; supports Elastic IP |
| NLB security groups | Must be associated at creation if you want to use them |
| ACM public certs | Free, auto-renewed |
| GWLB protocol/port | GENEVE / **6081** |
| ALB targets | Instance, IP, Lambda |
| NLB targets | Instance, IP, ALB |
| Subnets per LB | Must enable **≥2 AZs** for HA |

---

**Next**: [03_auto_scaling_groups.md — Auto Scaling Groups](03_auto_scaling_groups.md)
