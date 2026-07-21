# Transit Gateway & Advanced Networking

> **Who this is for**: Engineers who understand [VPC peering and endpoints](05_vpc_peering_and_endpoints.md)
> and hit the wall of peering's non-transitivity and mesh explosion. This file covers scaling
> connectivity (Transit Gateway), segmenting and inspecting a routed multi-account network,
> observing traffic (Flow Logs, Traffic Mirroring), inspecting it
> (Network Firewall), and the remaining advanced topics the exam touches: IPv6, the default VPC,
> BYOIP, and longest-prefix routing.

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

## 3. Professional Transit Gateway Design

A production TGW is not one giant route table. It has two routing layers, and
both directions must work:

```
source subnet route table -> TGW attachment
TGW route table associated with that source attachment -> destination attachment
destination subnet route table -> destination

return path repeats the same decisions in reverse
```

The **association** answers, "Which TGW route table evaluates traffic arriving
from this attachment?" An attachment associates with exactly one TGW route
table. **Propagation** answers, "Which route tables learn this attachment's
routes?" One attachment can propagate to several route tables. Association and
propagation are independent; attaching a VPC does not finish the VPC subnet
routes or guarantee a return path.

### Multiple route tables create segmentation domains

Disable automatic default association/propagation when you need deliberate
segmentation, then build route tables around trust domains:

| TGW route table | Associated attachments | Routes it should learn or contain |
|-----------------|------------------------|-----------------------------------|
| `prod` | Production VPCs | Other approved prod networks, shared services, inspection, and on-prem; no dev routes |
| `nonprod` | Development/test VPCs | Approved nonprod networks, shared services, and inspection; no prod routes |
| `core` | Inspection, shared-services, VPN/DX | Return routes to every authorized spoke domain |

This is VRF-like segmentation. A missing route isolates domains even if their
security groups are permissive. Add **blackhole routes** where a broader summary
could otherwise re-enable a forbidden path. Because propagated routes cannot be
filtered individually, use separate route tables, controlled propagation, and
static summaries when the policy needs filtering.

> **Rule**: A spoke needs a route *to* the TGW, the ingress attachment needs the
> correct TGW route-table association, that table needs a route *to* the
> destination attachment, and the entire sequence needs a reverse path.

### Centralized inspection and symmetric routing

For east-west or egress inspection, place stateful firewalls in a dedicated
inspection VPC rather than in every spoke:

```
spoke subnet -> TGW
spoke TGW table: 0.0.0.0/0 or approved prefixes -> inspection attachment
inspection TGW subnet -> firewall / GWLB endpoint -> back to TGW
inspection TGW table: spoke and on-prem prefixes -> destination attachments
```

The inspection VPC needs separate route tables on the TGW attachment subnets and
firewall/appliance subnets so traffic cannot bypass the appliance. Deploy the
inspection path in every used AZ. On the inspection VPC attachment, enable
**appliance mode** when a stateful appliance must see both directions: TGW then
keeps a flow on the same attachment AZ for its lifetime. Without it, a response
can reach a different appliance instance that has no session state and be
dropped.

Appliance mode does not repair an asymmetric topology. Verify that ingress from
the internet, VPN, Direct Connect, and every spoke all enter and leave through
the intended TGW/inspection path. A packet that enters the inspection VPC
through an IGW but returns through TGW can still bypass session symmetry.

### Inter-Region TGW peering

Transit Gateways are Regional. Connect Regions with a **TGW peering attachment**
and add static routes on both TGWs. Routes do **not** propagate dynamically over
the peering attachment, so advertise regional summary CIDRs where the address
plan permits and update both sides when the plan changes.

Inter-Region peering is for network connectivity, not service failover. Each
Region still needs its own attachments, inspection/egress decisions, DNS, and
resilient workloads. Monitor the static route inventory so a newly allocated
regional CIDR does not become a one-way black hole.

### Direct Connect gateway integration

The scalable hybrid path is:

```
on-prem router -> Direct Connect transit VIF -> Direct Connect gateway
               -> Transit Gateway -> VPC attachments
```

BGP-learned on-prem routes can propagate from the Direct Connect gateway
attachment into selected TGW route tables. The Direct Connect gateway's
**allowed prefixes** control which AWS prefixes it advertises toward on-premises;
for a TGW association, those configured prefixes are originated as written, not
automatically expanded into every attached VPC CIDR. Coordinate allowed prefixes,
TGW propagation, and regional summaries or the AWS and on-prem route views will
disagree. Build redundant Direct Connect locations and use VPN as backup when
the availability requirement calls for it.

### Multicast is a separate feature with hard boundaries

For applications that genuinely require one-to-many IP delivery, create a new
TGW with multicast enabled, create **multicast domains**, and associate VPC
subnets with them. Group members can join through IGMPv2 or be registered through
the API. A domain's `StaticSourcesSupport` and `Igmpv2Support` modes cannot both
be enabled.

Multicast routing works between associated VPC subnets. It is **not supported**
over Direct Connect, Site-to-Site VPN, TGW peering, or Connect attachments, and
fragmented multicast packets are dropped. Throughput and membership quotas make
it unsuitable for some latency- or performance-sensitive systems, including
high-frequency trading. Check the current multicast quotas before treating TGW
as a drop-in replacement for an on-prem multicast fabric.

### Route-propagation troubleshooting checklist

When an attachment is `available` but traffic fails, check in this order:

1. **Source VPC route** — does the source subnet route the destination CIDR to
   the TGW, and was a TGW attachment subnet enabled in that AZ?
2. **Association** — is the source attachment associated with the TGW route
   table you are inspecting?
3. **Propagation/static route** — does that table contain an active route to the
   destination attachment? Look for disabled propagation, a more-specific route,
   or a blackhole route.
4. **Destination and return routes** — does the destination subnet route the
   source CIDR back to TGW, and does its attachment's associated TGW table route
   back to the source?
5. **Hybrid controls** — for VPN/DX, are BGP sessions up, prefixes advertised,
   and Direct Connect gateway allowed prefixes correct? For TGW peering, remember
   that only static routes cross the peer.
6. **Policy and inspection** — do SGs, NACLs, Network Firewall rules, appliance
   health, and appliance-mode symmetry allow both directions?

Use TGW route-table route search for the control plane, VPC Flow Logs for actual
ACCEPT/REJECT records, and Reachability Analyzer where the path type is
supported. Do not start by changing security groups if the TGW never learned a
route.

---

## 4. VPC Flow Logs — Seeing the Traffic

You can't see packets inside a VPC by default. **VPC Flow Logs** capture **metadata** about IP traffic — source/destination IP and port, protocol, bytes, and **ACCEPT or REJECT** — and publish it to **CloudWatch Logs**, **S3**, or **Amazon Data Firehose**.

- Captured at three levels: **VPC**, **subnet**, or **ENI** (instance interface).
- Flow logs record **metadata only — not packet contents/payloads**. (For payloads you'd use **Traffic Mirroring**, §5.)
- The **ACCEPT/REJECT** field tells you whether the SG/NACL allowed the flow — invaluable for debugging "why can't these two talk."

```
2 123456789 eni-abc 10.0.1.5 10.0.2.9 51000 3306 6 12 1500 ... ACCEPT OK
2 123456789 eni-abc 203.0.113.7 10.0.1.5 40500 22  6  4  240 ... REJECT OK
                                                              └────┴── blocked by SG/NACL
```

💡 Exam use cases for Flow Logs: **troubleshooting** connectivity (which rule dropped traffic), **security forensics**, and **compliance auditing**. If a question asks how to *see whether traffic was accepted or rejected*, the answer is Flow Logs.

### Analyzing flow logs with Athena

Sending flow logs to **S3** lets you query them with **Amazon Athena** (serverless SQL over S3 — no servers, pay per scan). This is the standard "analyze historical traffic" answer:

```sql
-- Top source IPs that were REJECTED (e.g. spotting a port scan / misconfig)
SELECT sourceaddress, destinationport, COUNT(*) AS rejects
FROM   vpc_flow_logs
WHERE  action = 'REJECT'
GROUP  BY sourceaddress, destinationport
ORDER  BY rejects DESC
LIMIT  20;
```

> **Rule**: Flow Logs → **S3** → query with **Athena** (optionally visualize in Amazon Quick Sight) is the go-to pattern for *ad-hoc / historical* traffic analysis. Flow Logs → **CloudWatch Logs** is the pattern for *real-time* alarms and metric filters.

---

## 5. VPC Traffic Mirroring — Capturing Actual Packets

Flow Logs give you **metadata** (the headers: who talked to whom, allowed or denied). When you need the **actual packet payloads** — for deep packet inspection, an IDS/IPS, or network forensics — you use **VPC Traffic Mirroring**.

Traffic Mirroring copies inbound/outbound traffic from an **ENI** and sends it to a **target** (an ENI, or an NLB fronting a fleet of security/monitoring appliances) for out-of-band analysis.

| | **VPC Flow Logs** | **VPC Traffic Mirroring** |
|---|-------------------|----------------------------|
| Captures | **Metadata** (5-tuple, bytes, ACCEPT/REJECT) | **Full packets** (headers **+ payload**) |
| Use case | "Was it allowed? Who talked to whom?" | Deep packet inspection, IDS/IPS, content forensics |
| Destination | CloudWatch Logs / S3 / Data Firehose | An ENI or NLB → monitoring/security appliances |
| Overhead | Negligible | Higher (copies real traffic); can filter by rules |

> **Rule**: Need headers only / "accepted or rejected" → **Flow Logs**. Need the **packet contents** for inspection (IDS/IPS, forensics) → **Traffic Mirroring**.

⚠️ Traffic Mirroring is supported on Nitro-based instances and lets you **filter** which traffic to mirror, so you don't have to copy everything.

---

## 6. AWS Network Firewall — Managed VPC-Wide Inspection

Security Groups and NACLs are stateful/stateless **L3/L4** filters tied to instances and subnets. **AWS Network Firewall** is a **managed, stateful** firewall that inspects **all traffic at the VPC perimeter** — including deep, L7-aware features that SG/NACLs can't do.

- Deployed into dedicated **firewall subnets**; you route VPC traffic *through* it (it integrates with route tables, and with **Transit Gateway** for centralized inspection across many VPCs).
- Capabilities beyond SG/NACL: **stateful rule groups**, **domain-name (FQDN) allow/deny lists**, **Suricata-compatible IPS signatures**, protocol/L7 filtering, and traffic logging.
- Scales and is highly available automatically (managed by AWS).

| Layer | Scope | Stateful? | Best for |
|-------|-------|-----------|----------|
| **Security Group** | Instance/ENI | Stateful | Allow rules per workload (L3/L4) |
| **NACL** | Subnet | Stateless | Coarse subnet allow/deny (L3/L4) |
| **AWS Network Firewall** | **VPC / TGW perimeter** | Stateful | **Centralized** egress filtering, **FQDN** rules, **IPS/IDS** (L3–L7) |
| **AWS WAF** | CloudFront / ALB / API GW | — | **HTTP(S) L7** app attacks (SQLi, XSS, rate limit) |

> **Rule**: "Centralized, managed **stateful inspection / IPS / domain-name filtering** inside the VPC" → **Network Firewall**. "Filter **HTTP/L7** web attacks at the app edge" → **WAF**. "Deploy **third-party** security appliances inline" → **Gateway Load Balancer** (see [security services](../13_security_services/03_threat_detection_services.md)).

---

## 7. IPv6 in a VPC (Brief)

- A VPC is **always** IPv4 (you can't disable it), but you can **add** an IPv6 CIDR block (a `/56` from Amazon's pool) and dual-stack your subnets.
- Amazon-provided global IPv6 addresses are globally unique, but route tables and
  firewalls still determine reachability. IPAM can also manage private IPv6 space.
- Native IPv6-to-IPv6 traffic does not use NAT. NAT Gateway supports **NAT64**
  only for an IPv6 client reaching an IPv4 destination, normally with DNS64.
- To allow IPv6 **outbound-only** for private instances, use an **Egress-Only Internet Gateway** (file 03), the IPv6 analog of a NAT Gateway.
- NACLs and security groups need **separate IPv6 rules** — an IPv4 `0.0.0.0/0` rule does **not** cover IPv6 `::/0`. This is a common oversight.

⚠️ `0.0.0.0/0` (IPv4) and `::/0` (IPv6) are different default routes/rules. Dual-stack subnets need both covered.

---

## 8. The Default VPC

Every account gets a **default VPC** per Region so you can launch instances immediately:

- CIDR `172.31.0.0/16`, with a **default public subnet in each AZ**, an **IGW pre-attached**, and a route to it. Instances launched here get a **public IP automatically**.
- ✅ Convenient for quick tests; ❌ **not** following least-exposure best practice for production.
- If deleted, you can recreate it, but production designs should use a **purpose-built custom VPC** (file 02).

---

## 9. Bring Your Own IP (BYOIP) — Teaser

**BYOIP** lets you bring your **own public IPv4/IPv6 address ranges** into AWS and use them for resources (e.g., Elastic IPs). Useful when customers have **allow-listed your existing IPs**, you have IP **reputation** to preserve, or for licensing tied to IP. You import the range and prove ownership via a signed authorization (ROA). Just know it exists and *why* (keep existing IPs) for the exam.

---

## 10. Route Priority — Longest-Prefix Match (Recap + AWS Specifics)

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

## 11. Hybrid Networking — Where This Goes Next

Transit Gateway is also the centralized attachment point for connecting AWS to **on-premises** networks. The two mechanisms — **Site-to-Site VPN** (encrypted tunnel over the internet) and **Direct Connect** (dedicated private fiber) — are covered in depth in **[Hybrid Networking](../14_hybrid_migration_dr/01_hybrid_networking.md)**. Everything in this section (CIDR planning, route tables, TGW, non-overlapping ranges) is the prerequisite for that.

---

## 12. Key Exam Points

- **Transit Gateway** = regional **hub-and-spoke** with **transitive routing**; replaces the peering mesh and centralizes VPN/Direct Connect. Still needs **non-overlapping CIDRs**.
- Use multiple **TGW route tables** to create segmentation domains. Association
  selects the table for traffic arriving from an attachment; propagation installs
  attachment routes into selected tables.
- **Appliance mode** on the inspection VPC attachment preserves AZ symmetry for
  stateful inspection, but the VPC and TGW routes must still force both directions
  through the appliance.
- TGW peering is inter-Region and uses **static routes only**. Direct Connect
  integration uses a transit VIF, Direct Connect gateway, BGP propagation, and
  allowed prefixes.
- TGW multicast requires multicast domains and VPC subnet associations; it does
  not traverse DX, VPN, peering, or Connect attachments.
- **VPC Flow Logs** capture traffic **metadata** (incl. ACCEPT/REJECT), not payloads → to **CloudWatch Logs / S3 / Data Firehose**. The go-to tool for "was traffic allowed or denied?" Query historical logs in **S3 with Athena**.
- **Traffic Mirroring** captures **full packets (payload included)** for IDS/IPS/forensics — use it when metadata isn't enough; Flow Logs when it is.
- **AWS Network Firewall** = managed, **stateful**, VPC/TGW-perimeter inspection with **FQDN filtering** and **IPS** (L3–L7). Distinct from **WAF** (HTTP L7 at the app edge) and **GWLB** (insert third-party appliances).
- A VPC is **always IPv4**; IPv6 is opt-in. Use an **Egress-Only IGW** for
  native IPv6 private outbound, or DNS64/NAT64 for IPv6-to-IPv4 access.
- IPv4 and IPv6 firewall rules are **separate** (`0.0.0.0/0` ≠ `::/0`).
- **Default VPC** = `172.31.0.0/16`, public subnets + IGW, auto public IPs — convenient, not production-grade.
- **BYOIP** brings your own public IP ranges into AWS (preserve allow-lists/reputation).
- Routing uses **longest-prefix match**; the **`local` route always wins** and is immutable.

---

## Common Real-World Misconfigurations

- ❌ Building a peering mesh that becomes unmanageable instead of adopting Transit Gateway.
- ❌ Attaching VPCs with overlapping CIDRs to a TGW and expecting clean routing.
- ❌ Forgetting TGW route table associations/propagations, so attachments can't reach each other.
- ❌ Putting every attachment in the default TGW route table and accidentally
  creating full prod/dev/shared-services connectivity.
- ❌ Sending stateful inspection traffic through different appliance AZs on the
  forward and return paths because appliance mode or symmetric routes are missing.
- ❌ Expecting propagated routes to cross an inter-Region TGW peering attachment;
  add static routes on both TGWs.
- ❌ Updating Direct Connect allowed prefixes without matching TGW and on-prem
  route policy, producing one-way hybrid connectivity.
- ❌ Adding IPv6 to subnets but only writing IPv4 SG/NACL rules — IPv6 traffic is unfiltered or blocked.
- ❌ Running production in the default VPC with auto-assigned public IPs and a wide-open layout.
- ❌ Expecting to override the immutable `local` route to redirect intra-VPC traffic — impossible.
- ❌ Reaching for Flow Logs when you need packet **payloads** (use Traffic Mirroring), or Traffic Mirroring when headers/metadata would do (cheaper Flow Logs).
- ❌ Trying to do **FQDN/domain-based egress filtering or IPS** with Security Groups/NACLs — that's **Network Firewall** territory.

---

**Next**: [07_enis_security_groups_and_service_networking.md — ENIs & Service Networking](07_enis_security_groups_and_service_networking.md)
