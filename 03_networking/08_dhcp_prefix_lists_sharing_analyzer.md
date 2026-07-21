# DHCP Option Sets, Prefix Lists, VPC Sharing & Reachability Analyzer

> **Who this is for**: Engineers who have worked through [VPCs and subnets](02_vpc_subnets_route_tables.md),
> [route tables and gateways](03_gateways_igw_nat.md), [security groups & NACLs](04_security_groups_vs_nacls.md),
> and [Transit Gateway](06_transit_gateway_and_advanced.md). This file fills the four remaining
> VPC blind spots the exam touches: how a VPC hands out DNS settings (**DHCP option sets**),
> how to manage many CIDRs as one object (**managed prefix lists**), how multiple accounts
> share one network (**VPC sharing**), and how to *prove* whether traffic can flow
> (**Reachability Analyzer**). None of these are large, but each is a clean exam answer.

---

## 1. VPC DNS Attributes — The Prerequisite You've Been Using

Before DHCP option sets, two VPC-level switches decide whether DNS works at all. You've relied on them implicitly (the `.2` resolver in [file 02](02_vpc_subnets_route_tables.md#7-aws-reserves-5-ips-in-every-subnet), private DNS on [interface endpoints](05_vpc_peering_and_endpoints.md#interface-endpoint-aws-privatelink)) without naming them:

| Attribute | What it controls | Default (default VPC) | Default (custom VPC) |
|-----------|------------------|------------------------|----------------------|
| `enableDnsSupport` | Whether the **`.2` Amazon DNS resolver** answers queries in the VPC | On | **On** |
| `enableDnsHostnames` | Whether instances with a public IP get a **public DNS hostname** | On | **Off** |

> **Key insight**: Interface endpoint **private DNS** and Route 53 **private hosted zones** require **both** attributes **on**. This is the silent cause of "private DNS won't resolve to the endpoint" — the network is fine; the VPC just isn't allowed to hand out hostnames.

⚠️ A brand-new **custom** VPC has `enableDnsHostnames` **off**. Turn it on before expecting private hosted zones or interface-endpoint private DNS to work.

---

## 2. DHCP Option Sets — How a VPC Configures Its Clients

When an instance boots, it asks for network settings over **DHCP** (Dynamic Host Configuration Protocol) — the same mechanism your laptop uses on Wi-Fi. In a VPC, the answers come from a **DHCP option set** attached to the VPC.

A DHCP option set can specify:

| Option | What it sets | Common use |
|--------|--------------|------------|
| `domain-name-servers` | Which DNS servers instances use | Default is `AmazonProvidedDNS` (the `.2` resolver). Override to point at **your own / on-prem DNS**. |
| `domain-name` | The default domain suffix appended to hostnames | e.g. `corp.example.com` |
| `ntp-servers` | Time servers | Point at a specific NTP source |
| `netbios-name-servers` / `netbios-node-type` | Legacy Windows name resolution | Rarely needed |

Key facts:

- Every VPC has a **default option set** using `AmazonProvidedDNS`. That's why DNS "just works" out of the box.
- You **cannot edit** an option set after creation — you **create a new one and associate it** with the VPC. (Immutable, like an AMI.)
- A VPC has **one** option set at a time; changing it affects instances on their **next DHCP lease renewal** (or reboot), not instantly.

```
Default behavior                    Custom DHCP option set
────────────────                    ──────────────────────
domain-name-servers: AmazonDNS      domain-name-servers: 10.0.0.10, 10.0.0.11  (your AD/DNS)
domain-name:         <region>...    domain-name:         corp.example.com
   → uses the .2 resolver              → instances resolve internal corp names
```

> **Rule**: If a question wants instances to use a **custom or on-premises DNS server** (e.g., resolve internal Active Directory names directly), the answer is a **custom DHCP option set** — *or* Route 53 Resolver outbound endpoints ([file 07](07_enis_security_groups_and_service_networking.md#hybrid-dns)) for conditional forwarding. DHCP option set = "replace the resolver entirely"; Resolver endpoints = "forward only certain domains, keep AWS DNS for the rest."

💡 Replacing the resolver wholesale with a DHCP option set means you **lose** automatic resolution of AWS private DNS (endpoints, private hosted zones) unless your custom DNS forwards back to the `.2` resolver. That's why Route 53 Resolver is usually the better hybrid-DNS answer.

---

## 3. Managed Prefix Lists — One Name for Many CIDRs

You've already seen the **AWS-managed** S3 prefix list (`pl-xxxx`) used as a [gateway-endpoint route target](05_vpc_peering_and_endpoints.md#gateway-endpoint). A **managed prefix list** generalizes that: it's a **named, reusable set of CIDR blocks** you reference in route tables and security group rules instead of pasting the same CIDRs everywhere.

There are two kinds:

| Type | Who maintains it | Example |
|------|------------------|---------|
| **AWS-managed** | AWS keeps the IP ranges current | The S3 / DynamoDB prefix lists used by gateway endpoints |
| **Customer-managed** | You create and maintain | "Corporate office ranges", "all on-prem subnets", "partner IPs" |

Why it matters:

- Reference **one prefix list** in a security group rule or route table. When the underlying CIDRs change, you **edit the list once** and every rule/route that references it updates automatically — no hunting through dozens of SGs.
- Works as a **route target** *and* as a **source/destination in SG and NACL rules**.
- Shareable **across accounts** via AWS RAM (ties into §4).

```
Without a prefix list                With a customer-managed prefix list
─────────────────────                ───────────────────────────────────
SG rule 1: allow 443 from 203.0.1.0/24   pl-corp = { 203.0.1.0/24,
SG rule 2: allow 443 from 198.51.7.0/24             198.51.7.0/24,
SG rule 3: allow 443 from 192.0.2.0/24              192.0.2.0/24 }
   ...repeated in every SG...         SG rule: allow 443 from pl-corp   ← one rule, everywhere
```

> **Rule**: "Manage the **same set of CIDRs** across many security groups, route tables, or accounts without editing each one" → **customer-managed prefix list**. Edit the list once; every reference updates.

⚠️ A prefix list has a **max-entries** size you set at creation. Every rule that references it consumes that many entries against SG/route-table limits — size it deliberately, because **you can't shrink** max-entries below current usage.

---

## 4. VPC Sharing — Many Accounts, One Network

In a multi-account setup (AWS Organizations), you often want teams in **separate accounts** to run workloads on **one shared, centrally-managed network** — instead of each account building its own VPC, NAT, and endpoints. **VPC sharing** does this via **AWS Resource Access Manager (RAM)**.

The owner account creates the VPC and **shares specific subnets**; **participant** accounts launch resources **into those shared subnets**.

```
        ┌─────────────────── Networking (owner) account ───────────────────┐
        │  VPC 10.0.0.0/16  +  IGW  +  NAT GW  +  interface endpoints       │
        │  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐              │
        │  │ subnet AZ-a │   │ subnet AZ-b │   │ subnet AZ-c │  shared via  │
        │  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘  AWS RAM     │
        └─────────┼─────────────────┼─────────────────┼───────────────────┘
                  │                 │                 │
           launch │          launch │          launch │
        ┌─────────▼──────┐  ┌────────▼───────┐  ┌──────▼─────────┐
        │ App-team acct  │  │ Data-team acct │  │ Web-team acct  │   (participants)
        │ runs EC2/RDS   │  │ runs Fargate   │  │ runs ALB       │
        └────────────────┘  └────────────────┘  └────────────────┘
```

Who owns what:

- **Owner** account: owns the VPC, subnets, route tables, IGW, NAT, gateways, and **manages all networking**. Participants **cannot** modify the VPC, subnets, or route tables.
- **Participant** accounts: own the **resources they launch** (EC2, RDS, ENIs) and their **security groups**, and pay for those resources. They **see** the shared subnets but can't change the network.

> **Rule**: "Multiple accounts must share **one** VPC / centralize NAT and endpoints / let a network team own routing while app teams own only their workloads" → **VPC sharing via AWS RAM**. Contrast with **Transit Gateway** (separate VPCs that need to *route between* each other) — sharing means **one** VPC used by many; TGW means **many** VPCs connected.

✅ Benefits: fewer VPCs to manage, **shared NAT Gateways and interface endpoints** (big cost saver — no duplicate `$/hr` per account), centralized network governance, and no CIDR-overlap headaches between teams (they're in the *same* VPC).

⚠️ Sharing requires **RAM sharing enabled within the Organization**, and you share **subnets**, not the whole VPC. Security groups **can be referenced across the shared VPC** by participants, but a participant can't delete a subnet or touch the route tables.

### VPC sharing vs Transit Gateway vs Peering — the one table

| Need | Use |
|------|-----|
| Many accounts run workloads on **one** network, central NAT/endpoints | **VPC sharing (RAM)** |
| **Separate** VPCs must route to each other, transitively, at scale | **Transit Gateway** |
| Exactly **two** VPCs need private connectivity, simple | **VPC Peering** |
| Expose **one service** privately (incl. overlapping CIDRs) | **PrivateLink** |

---

## 5. Reachability Analyzer & Network Access Analyzer — Proving the Path

[Flow Logs](06_transit_gateway_and_advanced.md#3-vpc-flow-logs--seeing-the-traffic) tell you what traffic *did* happen (after the fact). Sometimes you need to know whether traffic *can* happen — **before** deploying, or to debug "why can't A reach B" **without sending a single packet**. Two analysis tools do this.

### VPC Reachability Analyzer

A **configuration analysis** tool: pick a **source** and **destination** (instance, ENI, IGW, TGW, endpoint, etc.) and it traces the **virtual network path**, reporting **Reachable** or **Not reachable** — and if blocked, **exactly which component** stopped it (a missing route, an SG, a NACL).

```
Source: app EC2 (eni-aaa)   →   Destination: DB EC2 (eni-bbb)
Result: NOT REACHABLE
Blocking component: NACL on the DB subnet — no inbound rule for 3306
                    └── tells you the exact hop that fails, no packets sent
```

- Analyzes **configuration**, not live traffic — works even before resources serve traffic.
- Pinpoints the **specific blocking hop** (route table, SG, NACL, peering, etc.).
- Works **within an account** and **across accounts** (and across TGW/peering).

> **Rule**: "Why can't these two resources communicate?" / "Will this path work?" answered from **configuration, without generating traffic** → **Reachability Analyzer**. (Flow Logs = *was* a real packet allowed/denied; Reachability Analyzer = *can* a path work at all.)

### Network Access Analyzer

Goes the other direction: instead of "can A reach B," it audits **"what unintended network access exists?"** You define **Network Access Scopes** (e.g., "nothing in a private subnet should be reachable from the internet") and it finds **every path that violates** the scope across your VPCs.

> **Rule**: **Reachability Analyzer** = debug/verify **one specific path**. **Network Access Analyzer** = **audit at scale** for unintended/overly-broad access (compliance, "prove nothing public can reach the database tier").

💡 Both are far faster than the old approach of reading every route table, SG, and NACL by hand — and they're the modern exam answer for connectivity troubleshooting that doesn't involve packet capture.

---

## 6. Key Exam Points

- **`enableDnsSupport` + `enableDnsHostnames`** must **both** be on for **interface-endpoint private DNS** and **Route 53 private hosted zones**. Custom VPCs have **hostnames off** by default.
- **DHCP option set** = how a VPC hands instances their **DNS servers, domain name, NTP**. Default uses `AmazonProvidedDNS` (the `.2` resolver). Option sets are **immutable** — replace, don't edit.
- Use a **custom DHCP option set** to point instances at **on-prem/custom DNS**; use **Route 53 Resolver endpoints** for conditional forwarding that keeps AWS DNS working.
- **Managed prefix list** = a named, reusable set of CIDRs for **route tables and SG/NACL rules**. Edit once, every reference updates. **Customer-managed** ones are shareable via RAM. Watch the immutable **max-entries** size.
- **VPC sharing (AWS RAM)** = many **accounts** run workloads in **one** owner-managed VPC's **shared subnets**; owner controls networking, participants own only their resources. Saves duplicate **NAT/endpoint** cost.
- **VPC sharing vs TGW**: sharing = one VPC, many accounts; TGW = many VPCs, routed together.
- **Reachability Analyzer** = trace **one path** from **configuration** (no packets) and get the **exact blocking component**. "Why can't A reach B?"
- **Network Access Analyzer** = audit **at scale** for **unintended access** against defined scopes. "Prove nothing public reaches the DB tier."

---

## Common Real-World Misconfigurations

- ❌ Expecting interface-endpoint private DNS or a private hosted zone to resolve while `enableDnsHostnames` is **off** (the custom-VPC default).
- ❌ Trying to **edit** a DHCP option set in place — you must create a new one and re-associate, then wait for the DHCP lease to renew.
- ❌ Pointing a DHCP option set at custom DNS that **doesn't forward to the `.2` resolver**, breaking resolution of AWS endpoints and private hosted zones.
- ❌ Pasting the same corporate CIDRs into dozens of security groups instead of one **customer-managed prefix list** — then missing some when the ranges change.
- ❌ Sizing a prefix list's **max-entries** too small (can't grow rule references) or too large (wastes SG/route-table capacity).
- ❌ Building a separate VPC + NAT + endpoints in **every** account when **VPC sharing** would centralize and cut cost.
- ❌ Confusing **VPC sharing** (one VPC, many accounts) with **Transit Gateway** (many VPCs connected).
- ❌ Hand-reading every SG/NACL/route table to debug connectivity when **Reachability Analyzer** would name the blocking hop instantly.

---

**Next**: [../04_compute/01_ec2_fundamentals.md — EC2 Fundamentals](../04_compute/01_ec2_fundamentals.md)
