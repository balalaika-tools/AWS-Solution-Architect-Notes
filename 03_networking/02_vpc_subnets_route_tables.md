# VPC, Subnets & Route Tables — Building the Virtual Network

> **Who this is for**: Engineers who have read the [Networking Primer](01_networking_primer.md)
> and understand IP, CIDR, subnets, and route tables conceptually. Now we map those exact
> concepts onto AWS. If "longest-prefix match" and "`0.0.0.0/0` is the default route" don't
> ring a bell, go back to file 01 first.

---

## 1. What a VPC Is

A **VPC (Virtual Private Cloud)** is your own logically isolated, software-defined network inside an AWS Region. It's the AWS equivalent of the private network you'd build in your own data center — except it's defined entirely in configuration.

Everything you put inside a VPC (EC2 instances, RDS databases, load balancers, Lambda ENIs) gets a private IP from the VPC's address range and talks over this private network by default.

- A VPC is **scoped to one Region** but **spans all Availability Zones** in that Region.
- Your account gets a **default VPC** in every Region (convenient but not exam-best-practice; see section 6).
- You can have multiple VPCs per Region (default soft limit: 5, adjustable).

> **Key insight**: A VPC is just a private IP range plus the routing and security rules around it. Nothing more magical than the LAN concepts from file 01 — implemented in software at AWS scale.

---

## 2. Choosing the VPC CIDR Block

When you create a VPC you assign it a CIDR block — its entire private address range. This is the most important up-front decision because **you can't easily shrink it later**.

AWS constrains the VPC CIDR to between **`/16` and `/28`**:

| Prefix | Addresses | Notes |
|--------|-----------|-------|
| `/16`  | 65,536 | **Largest** allowed VPC. The common default. |
| `/28`  | 16     | **Smallest** allowed VPC. |

✅ Best practice: use a private RFC1918 range and pick `/16` for room to grow, e.g. `10.0.0.0/16`.

⚠️ **Do not let the VPC CIDR overlap** with other VPCs you may later peer with, or with your on-premises network if you'll connect via VPN/Direct Connect. Overlap makes peering and hybrid connectivity impossible (from file 01: you can't route to an ambiguous address).

💡 You can add **secondary CIDR blocks** to a VPC later if you run out of space, but planning a big enough range up front is cleaner.

---

## 3. Subnets Are AZ-Scoped

You then carve the VPC's range into **subnets**. The single most important AWS-specific fact:

> **Rule**: A subnet lives in exactly **one Availability Zone**. A subnet **cannot span AZs**. A VPC spans all AZs, but each subnet is pinned to one.

This is *the* mechanism for high availability: to survive an AZ failure, you create the *same kind* of subnet in *multiple AZs* and spread your resources across them.

```
VPC 10.0.0.0/16  (Region: spans all AZs)
│
├── Subnet  10.0.1.0/24  → AZ a
├── Subnet  10.0.2.0/24  → AZ b
└── Subnet  10.0.3.0/24  → AZ c
```

Each subnet's CIDR must fit inside the VPC's CIDR and not overlap other subnets (file 01, section 4).

---

## 4. Public vs Private Subnet — It's the Route Table, Not a Checkbox

This is the concept most beginners get wrong, so read carefully.

> **Rule**: There is **no "public" toggle** on a subnet that makes it public. A subnet is **"public" only because its route table sends `0.0.0.0/0` to an Internet Gateway**. Remove that route and the exact same subnet becomes private.

- **Public subnet** = its associated route table has a route `0.0.0.0/0 → igw-xxxx` (Internet Gateway). Resources here *can* be reached from / reach the internet (if they also have a public IP).
- **Private subnet** = no `0.0.0.0/0 → IGW` route. Resources here have no direct internet path. They reach the internet outbound only via a NAT Gateway (section 3 / file 03).

```
Same subnet, different route table = different behavior

PUBLIC subnet route table          PRIVATE subnet route table
┌──────────────┬───────────┐       ┌──────────────┬─────────────┐
│ 10.0.0.0/16  │ local     │       │ 10.0.0.0/16  │ local       │
│ 0.0.0.0/0    │ igw-abc   │ ◄──── │ 0.0.0.0/0    │ nat-xyz     │
└──────────────┴───────────┘  this └──────────────┴─────────────┘
                              route   (or no 0.0.0.0/0 at all)
                              is the
                              difference
```

⚠️ There *is* a setting called **"auto-assign public IP"** on a subnet. That only decides whether instances get a public IP automatically — it does **not** make the subnet public. Without the IGW route, a public IP is useless.

---

## 5. Route Tables and the Main Route Table

A **route table** in AWS is exactly the concept from file 01: destination CIDR → target. In a VPC:

- Every VPC has one **main route table** created automatically. Any subnet **not explicitly associated** with a custom route table uses the main one.
- You create **custom route tables** and associate subnets with them to control behavior per subnet.
- A subnet is associated with **exactly one** route table at a time; a route table can be associated with **many** subnets.

✅ Best practice: leave the main route table private (no internet route) so that any *new* subnet you forget to configure defaults to private — fail safe, not fail open. Create an explicit "public" route table for public subnets.

### The implicit `local` route

Every VPC route table contains an automatic, **unmodifiable, undeletable** route:

```
Destination: <the entire VPC CIDR, e.g. 10.0.0.0/16>
Target:      local
```

This `local` route is why **every subnet in a VPC can talk to every other subnet by default**, with no extra configuration. Routing inside the VPC "just works." (Whether the *firewall* allows that traffic is a separate question — security groups and NACLs, file 04.)

> **Key insight**: Intra-VPC connectivity is free and automatic thanks to the `local` route. You only add routes for traffic that needs to *leave* the VPC (internet, NAT, peering, gateways).

---

## 6. ENIs Consume Subnet IPs

An **Elastic Network Interface (ENI)** is the virtual network card that places a
resource into a subnet. EC2 instances are the obvious example, but many managed
services also create ENIs or endpoint ENIs: load balancers, NAT gateways, VPC
Lambda, ECS/Fargate tasks, EKS nodes/pods, RDS/Aurora, ElastiCache, EFS mount
targets, Route 53 Resolver endpoints, and interface VPC endpoints.

That matters for subnet design:

- An ENI lives in exactly **one subnet/AZ**.
- ENIs take private IPs from the subnet CIDR.
- Security groups attach to ENIs, not to subnets.
- Tiny subnets can run out of IPs even when you have "only a few servers,"
  because endpoints, tasks, functions, databases, and mount targets also need IPs.

For the full cross-service map, see
[ENIs, Security Groups & Service Networking](07_enis_security_groups_and_service_networking.md).

---

## 7. AWS Reserves 5 IPs in Every Subnet

When you create a subnet, AWS reserves the **first four addresses and the last address** — **5 IPs total** — that you cannot assign to instances. For a `/24` (256 addresses) you get only **251** usable.

For subnet `10.0.1.0/24`:

| Address | Reserved for |
|---------|--------------|
| `10.0.1.0`   | Network address |
| `10.0.1.1`   | VPC router |
| `10.0.1.2`   | AWS DNS (the `.2` resolver — important for DNS) |
| `10.0.1.3`   | Reserved for future AWS use |
| `10.0.1.255` | Network broadcast (broadcast unsupported, still reserved) |

> **Exam math**: usable hosts = (2^host_bits) − 5. A `/28` (16 addresses) gives only **11** usable — which is why `/28` is the practical floor.

⚠️ This is a favorite exam trick: "How many usable IPs in a `/27`?" → 32 − 5 = **27**.

---

## 8. Putting It Together — Two-AZ Reference VPC

The canonical exam topology: one VPC, two AZs, each with a public and a private subnet.

```
                          VPC  10.0.0.0/16
   ┌───────────────────────────────────────────────────────────────┐
   │                                                               │
   │   ── Availability Zone A ──        ── Availability Zone B ──  │
   │  ┌──────────────────────────┐    ┌──────────────────────────┐ │
   │  │ PUBLIC subnet            │    │ PUBLIC subnet            │ │
   │  │ 10.0.1.0/24              │    │ 10.0.2.0/24              │ │
   │  │   (ALB, NAT GW, bastion) │    │   (ALB, NAT GW)          │ │
   │  └──────────────────────────┘    └──────────────────────────┘ │
   │                                                               │
   │  ┌──────────────────────────┐    ┌──────────────────────────┐ │
   │  │ PRIVATE subnet           │    │ PRIVATE subnet           │ │
   │  │ 10.0.11.0/24             │    │ 10.0.12.0/24             │ │
   │  │   route: 0.0.0.0/0 → NAT │    │   route: 0.0.0.0/0 → NAT │ │
   │  │   (app servers, RDS)     │    │   (app servers, RDS)     │ │
   │  └──────────────────────────┘    └──────────────────────────┘ │
   │                                                               │
   │   local route 10.0.0.0/16 → local  (all four subnets reach    │
   │                                     each other automatically) │
   └───────────────────────────────────────────────────────────────┘
                                │
                          Internet Gateway  (covered in file 03)
```

- Public subnets in **two AZs** → a load balancer can be highly available.
- Private subnets in **two AZs** → app and database tiers survive an AZ outage.
- All four share the `local` route, so app servers reach the database with no extra routing.
- The Internet Gateway and NAT Gateway that make this work are file 03.

---

## 9. Organization-Scale IP Address Management

A `/16` chosen in isolation is not a strategy. In a multi-account environment,
every VPC that might connect through Transit Gateway, peering, VPN, or Direct
Connect needs a **unique, non-overlapping** CIDR. The network team should assign
address space centrally rather than letting each account pick from RFC1918.

### Build a hierarchy, not a spreadsheet

**Amazon VPC IP Address Manager (IPAM)** discovers and monitors address use and
allocates CIDRs from centrally managed pools. Integrate IPAM with AWS
Organizations, delegate administration to a network account, and share pools to
member accounts with AWS RAM. A useful hierarchy is:

```
Private IPv4 scope: 10.0.0.0/8
├── eu-pool:   10.0.0.0/10
│   ├── prod:  10.0.0.0/12
│   └── nonprod: reserved regional space
└── us-pool:   10.64.0.0/10
    ├── prod
    └── nonprod
```

An account requests a VPC CIDR from its assigned pool; IPAM selects a compliant
block and records the allocation. Pool rules can enforce minimum and maximum
prefix lengths, required tags, and permitted Regions. IPAM also reports
overlapping or out-of-policy resources that were created outside the allocation
workflow.

The exact hierarchy depends on the organization, but keep these invariants:

- Allocate **disjoint blocks by Region and environment** so routing and incident
  boundaries are obvious.
- Reserve unused blocks between current allocations for new accounts, more AZs,
  larger EKS clusters, acquisitions, and future Regions. Do not pack every VPC
  tightly just because the first deployment is small.
- Size subnets from expected **ENI and pod consumption**, not only today's EC2
  count. See [ENI capacity planning](07_enis_security_groups_and_service_networking.md#5-capacity-planning--ips-and-enis-are-hard-limits).
- Track utilization and alert before a pool or subnet is nearly full. A secondary
  VPC CIDR can add space, but it cannot repair an overlapping routed design.

### Plan IPv4 and IPv6 together

For dual-stack designs, manage separate IPv4 and IPv6 pools and assign both
families consistently. IPv6 removes private-address scarcity, but it does not
remove routing or security work: route tables, security groups, NACLs, DNS, and
egress controls need explicit IPv6 entries. Keep IPv4 while dependencies still
require it; treat dual-stack as a migration with two paths to test, not as a
checkbox that instantly removes IPv4.

### Why overlap is expensive to fix

Overlapping VPCs can remain isolated, and PrivateLink can expose a specific
service across overlap. They cannot participate cleanly in ordinary peering,
Transit Gateway, or hybrid routing. Renumbering later usually means allocating a
new CIDR, creating replacement subnets, redeploying ENI-backed resources,
updating DNS and allow-lists, and migrating traffic while both ranges exist.
Databases and managed services may require replacement or a service-specific
migration. That operational cost is why centrally allocated, growth-aware CIDRs
are an architecture decision, not cleanup for later.

---

## 10. Key Exam Points

- A **VPC is Region-scoped**; a **subnet is AZ-scoped** (one AZ only, never spans AZs).
- VPC CIDR must be **`/16` to `/28`**. Use private RFC1918 ranges; avoid overlap with peers/on-prem.
- "Public subnet" = route table has **`0.0.0.0/0 → Internet Gateway`**. Nothing else makes it public.
- The **`local` route** is automatic and immutable — all subnets in a VPC can reach each other.
- The **main route table** is the default for unassociated subnets; keep it private to fail safe.
- ENIs place resources into subnets and consume subnet IPs; plan subnet size for managed services too, not only EC2.
- AWS reserves **5 IPs per subnet**: usable = 2^host_bits − 5.
- Spread subnets across multiple AZs for high availability.
- At organization scale, use **VPC IPAM** and AWS Organizations to allocate
  non-overlapping CIDRs from shared pools, reserve growth space, and monitor
  compliance and utilization.
- Plan dual-stack addressing deliberately; IPv4 and IPv6 have separate routes
  and security rules.

---

## Common Real-World Misconfigurations

- ❌ Picking a `/24` VPC then running out of subnet space immediately. Use `/16`.
- ❌ Choosing `10.0.0.0/16` for two VPCs you later want to peer — overlapping CIDRs block peering forever.
- ❌ Letting every account choose its own RFC1918 range instead of allocating
  from IPAM pools — the collision usually appears only when hybrid or multi-VPC
  routing is added.
- ❌ Assuming "auto-assign public IP" makes a subnet public. Without the IGW route, the public IP does nothing.
- ❌ Forgetting the 5 reserved IPs and undersizing a subnet (`/28` gives only 11 usable hosts).
- ❌ Placing all subnets in one AZ — a single AZ failure takes the whole app down.

---

**Next**: [03_gateways_igw_nat.md — Internet Gateway, NAT, and Internet Access](03_gateways_igw_nat.md)
