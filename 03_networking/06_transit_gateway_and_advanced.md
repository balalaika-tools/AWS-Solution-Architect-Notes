# Transit Gateway & Advanced Networking

> **Who this is for**: Engineers who understand [VPC peering and endpoints](05_vpc_peering_and_endpoints.md)
> and hit the wall of peering's non-transitivity and mesh explosion. This file covers scaling
> connectivity (Transit Gateway), observing traffic (Flow Logs), and the remaining advanced
> topics the exam touches: IPv6, the default VPC, BYOIP, and longest-prefix routing.

---

## 1. The Problem: Peering Doesn't Scale

From file 05, VPC peering is **non-transitive** — every pair of VPCs that must communicate needs its own peering connection. Connecting N VPCs in a full mesh requires **N(N−1)/2** connections, and each one needs route table edits on both sides.

```
Full-mesh peering of 5 VPCs = 10 connections, and it gets worse fast:

        A ──── B
        │ ╲  ╱ │
        │  ╳   │      4 VPCs → 6 connections
        │ ╱  ╲ │      5 VPCs → 10
        D ──── C      10 VPCs → 45  (unmanageable)
```

This mesh becomes unmanageable past a handful of VPCs. **Transit Gateway** solves it.

---

## 2. Transit Gateway (TGW) — Hub-and-Spoke, Transitive Routing

A **Transit Gateway** is a regional network hub. You **attach** VPCs (and VPN / Direct Connect connections) to it, and the TGW routes between all of them. Instead of a mesh, you get a **star**:

```
            ┌──────────────────────────┐
            │   Transit Gateway (hub)  │
            └───┬───────┬───────┬──────┘
        attach  │       │       │  attach
          ┌─────▼─┐  ┌──▼───┐ ┌─▼─────┐
          │ VPC A │  │ VPC B│ │ VPC C │       + on-prem via VPN / Direct Connect
          └───────┘  └──────┘ └───────┘
       transitive: A↔B↔C all reachable through the hub
```

| | VPC Peering | Transit Gateway |
|---|-------------|-----------------|
| Topology | Point-to-point mesh | **Hub-and-spoke** |
| Routing | **Non-transitive** | **Transitive** (A↔B↔C through the hub) |
| Connections for N VPCs | N(N−1)/2 | **N attachments** (one each to the hub) |
| On-prem (VPN/DX) integration | Per-VPC | **Centralized** at the hub |
| Cost | No hourly fee (data transfer only) | Hourly per-attachment + per-GB data |
| Scale | A few VPCs | Hundreds / thousands |
| Routing control | Route tables per VPC | **TGW route tables** for segmentation |

> **Rule**: Few VPCs, simple → **peering**. Many VPCs, transitive routing, centralized hybrid connectivity → **Transit Gateway**. "Replace the mesh" or "connect hundreds of VPCs and on-prem centrally" is the TGW trigger phrase.

⚠️ TGW still requires **non-overlapping CIDRs** across attached VPCs for normal routing — the ambiguous-address rule from the primer never goes away. TGW route tables let you **segment** (e.g., keep prod and dev VPCs from talking) by controlling which attachments share a route table.

---

## 3. VPC Flow Logs — Seeing the Traffic

You can't see packets inside a VPC by default. **VPC Flow Logs** capture **metadata** about IP traffic — source/destination IP and port, protocol, bytes, and **ACCEPT or REJECT** — and publish it to **CloudWatch Logs**, **S3**, or **Kinesis Data Firehose**.

- Captured at three levels: **VPC**, **subnet**, or **ENI** (instance interface).
- Flow logs record **metadata only — not packet contents/payloads**. (For payloads you'd use Traffic Mirroring, out of scope.)
- The **ACCEPT/REJECT** field tells you whether the SG/NACL allowed the flow — invaluable for debugging "why can't these two talk."

```
2 123456789 eni-abc 10.0.1.5 10.0.2.9 51000 3306 6 12 1500 ... ACCEPT OK
2 123456789 eni-abc 203.0.113.7 10.0.1.5 40500 22  6  4  240 ... REJECT OK
                                                              └────┴── blocked by SG/NACL
```

💡 Exam use cases for Flow Logs: **troubleshooting** connectivity (which rule dropped traffic), **security forensics**, and **compliance auditing**. If a question asks how to *see whether traffic was accepted or rejected*, the answer is Flow Logs.

---

## 4. IPv6 in a VPC (Brief)

- A VPC is **always** IPv4 (you can't disable it), but you can **add** an IPv6 CIDR block (a `/56` from Amazon's pool) and dual-stack your subnets.
- IPv6 addresses in AWS are **all globally routable** — there is **no NAT for IPv6**.
- To allow IPv6 **outbound-only** for private instances, use an **Egress-Only Internet Gateway** (file 03), the IPv6 analog of a NAT Gateway.
- NACLs and security groups need **separate IPv6 rules** — an IPv4 `0.0.0.0/0` rule does **not** cover IPv6 `::/0`. This is a common oversight.

⚠️ `0.0.0.0/0` (IPv4) and `::/0` (IPv6) are different default routes/rules. Dual-stack subnets need both covered.

---

## 5. The Default VPC

Every account gets a **default VPC** per Region so you can launch instances immediately:

- CIDR `172.31.0.0/16`, with a **default public subnet in each AZ**, an **IGW pre-attached**, and a route to it. Instances launched here get a **public IP automatically**.
- ✅ Convenient for quick tests; ❌ **not** following least-exposure best practice for production.
- If deleted, you can recreate it, but production designs should use a **purpose-built custom VPC** (file 02).

---

## 6. Bring Your Own IP (BYOIP) — Teaser

**BYOIP** lets you bring your **own public IPv4/IPv6 address ranges** into AWS and use them for resources (e.g., Elastic IPs). Useful when customers have **allow-listed your existing IPs**, you have IP **reputation** to preserve, or for licensing tied to IP. You import the range and prove ownership via a signed authorization (ROA). Just know it exists and *why* (keep existing IPs) for the exam.

---

## 7. Route Priority — Longest-Prefix Match (Recap + AWS Specifics)

When multiple routes match a destination, the router chooses the **most specific** — the **longest prefix** (most network bits). This is the same rule from the [primer](01_networking_primer.md#5️⃣-routing-and-route-tables), and it governs VPC route tables too.

```
Route table:
  10.0.0.0/16  → local
  10.0.1.0/24  → peering-conn      ← more specific, wins for 10.0.1.x
  0.0.0.0/0    → igw               ← least specific, catch-all default

Packet to 10.0.1.50 → matches /24 (peering)  ✅ (beats /16 and 0/0)
Packet to 10.0.9.50 → matches /16 (local)    ✅
Packet to 8.8.8.8   → matches 0/0 (igw)       ✅
```

AWS tiebreaker order when prefixes are equal-length: the **local route always wins first** (it's immutable), then AWS prefers more-specific static routes, then propagated routes by source priority (Direct Connect > VPN, etc.). For the exam, **longest-prefix-match + "local always wins"** is the essential takeaway.

> **Rule**: More specific route always beats less specific. The immutable `local` route can never be overridden — you cannot route around intra-VPC traffic.

---

## 8. Hybrid Networking — Where This Goes Next

Transit Gateway is also the centralized attachment point for connecting AWS to **on-premises** networks. The two mechanisms — **Site-to-Site VPN** (encrypted tunnel over the internet) and **Direct Connect** (dedicated private fiber) — are covered in depth in **[Hybrid Networking](../14_hybrid_migration_dr/01_hybrid_networking.md)**. Everything in this section (CIDR planning, route tables, TGW, non-overlapping ranges) is the prerequisite for that.

---

## 9. Key Exam Points

- **Transit Gateway** = regional **hub-and-spoke** with **transitive routing**; replaces the peering mesh and centralizes VPN/Direct Connect. Still needs **non-overlapping CIDRs**.
- Use **TGW route tables** to **segment** which attachments can talk (e.g., isolate prod from dev).
- **VPC Flow Logs** capture traffic **metadata** (incl. ACCEPT/REJECT), not payloads → to **CloudWatch Logs / S3 / Firehose**. The go-to tool for "was traffic allowed or denied?"
- A VPC is **always IPv4**; IPv6 is **opt-in dual-stack**, has **no NAT**, and needs an **Egress-Only IGW** for private outbound.
- IPv4 and IPv6 firewall rules are **separate** (`0.0.0.0/0` ≠ `::/0`).
- **Default VPC** = `172.31.0.0/16`, public subnets + IGW, auto public IPs — convenient, not production-grade.
- **BYOIP** brings your own public IP ranges into AWS (preserve allow-lists/reputation).
- Routing uses **longest-prefix match**; the **`local` route always wins** and is immutable.

---

## Common Real-World Misconfigurations

- ❌ Building a peering mesh that becomes unmanageable instead of adopting Transit Gateway.
- ❌ Attaching VPCs with overlapping CIDRs to a TGW and expecting clean routing.
- ❌ Forgetting TGW route table associations/propagations, so attachments can't reach each other.
- ❌ Adding IPv6 to subnets but only writing IPv4 SG/NACL rules — IPv6 traffic is unfiltered or blocked.
- ❌ Running production in the default VPC with auto-assigned public IPs and a wide-open layout.
- ❌ Expecting to override the immutable `local` route to redirect intra-VPC traffic — impossible.

---

**Next**: [../04_compute/01_ec2_fundamentals.md — EC2 Fundamentals](../04_compute/01_ec2_fundamentals.md)
