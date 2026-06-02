# VPC Peering & Endpoints — Private Connectivity Without the Internet

> **Who this is for**: Engineers comfortable with [route tables](02_vpc_subnets_route_tables.md),
> [gateways](03_gateways_igw_nat.md), and [security groups](04_security_groups_vs_nacls.md).
> This file covers connecting two VPCs together (peering) and reaching AWS services privately
> without traversing the internet (endpoints). The [overlapping-CIDR rule](01_networking_primer.md#3️⃣-private-vs-public-ip-ranges-rfc1918)
> from the primer is central here.

---

## 1. VPC Peering — Privately Connecting Two VPCs

By default, two VPCs are completely isolated — even in the same account and Region. **VPC Peering** creates a direct, private network connection between two VPCs so their resources can talk using **private IPs**, as if on one network. Traffic stays on the **AWS backbone** — it never touches the public internet.

```
   VPC A  10.0.0.0/16                 VPC B  172.16.0.0/16
   ┌──────────────────┐               ┌──────────────────┐
   │  EC2 10.0.1.5    │◄═════════════►│  EC2 172.16.2.9  │
   │                  │  peering conn │                  │
   └──────────────────┘  (pcx-xxxx)   └──────────────────┘
        private IPs, AWS backbone, no IGW / no internet
```

Peering works **across accounts** and **across Regions** (inter-region peering), not just within one account/Region.

### Three things you MUST do for peering to work

1. **Establish the peering connection** (one side requests, the other accepts).
2. **Update route tables on BOTH VPCs** — add a route in each VPC pointing the *other* VPC's CIDR at the peering connection (`pcx-xxxx`). Peering does *not* auto-create routes.
3. **Update security groups / NACLs** to allow the traffic. You *can* reference an SG in a *peered* VPC, but only within the **same Region**.

⚠️ The most common "peering doesn't work" cause is forgetting the route table entries on one or both sides.

### The two iron rules of peering

> **Rule 1 — Non-transitive.** If A peers with B, and B peers with C, then **A cannot reach C** through B. Peering is strictly point-to-point. To connect three VPCs fully you need three peering connections (a full mesh), and N VPCs need N(N−1)/2 connections — which is why Transit Gateway exists (file 06).

```
   A ──peer── B ──peer── C
   A  ──✗──  C    (no transitive routing through B)
```

> **Rule 2 — No overlapping CIDRs.** The two VPCs' CIDR blocks **must not overlap**. From the primer: routing to an ambiguous address is impossible — if both VPCs used `10.0.0.0/16`, AWS couldn't tell which `10.0.1.5` you meant. AWS rejects the peering request outright.

### DNS resolution across a peering connection

By default, an instance in VPC A querying a private DNS name in VPC B resolves it to the *public* IP, not the private one. To get **private IP resolution** across the peer, you must **enable DNS resolution on the peering connection** (both directions, as needed).

For the detailed worked example of when private DNS resolves and when it doesn't, see **[Practical: VPC Peering DNS Resolution](../18_practical_examples/06_vpc_peering_dns_resolution.md)**.

---

## 2. VPC Endpoints — Reaching AWS Services Privately

AWS services like **S3** and **DynamoDB** have *public* endpoints. By default, an instance in a private subnet reaches them by going **out through the NAT Gateway and IGW to the public internet** — even though both ends are inside AWS. That works, but it:

- routes private data over the public internet,
- depends on (and pays for) the NAT Gateway,
- can't be locked down by network path.

A **VPC Endpoint** lets resources in your VPC connect to supported AWS services **privately**, keeping all traffic **inside the AWS network** — no IGW, no NAT, no public internet.

```
WITHOUT endpoint                     WITH endpoint
─────────────────                    ──────────────
private EC2 → NAT GW → IGW           private EC2 ──► VPC Endpoint ──► S3
   → internet → S3                       (stays on AWS private network)
```

There are **three endpoint types** to know for the exam. Gateway and interface
endpoints are the everyday choices; Gateway Load Balancer endpoints show up in
inspection-appliance designs.

---

## 3. Gateway vs Interface vs Gateway Load Balancer Endpoints

### Gateway Endpoint

- A target you add to a **route table** (just like an IGW or NAT — it's a routing construct).
- Supports **only two services: S3 and DynamoDB.**
- **Free.**
- Regional; access controlled via route tables, IAM/resource policies, and endpoint policies.
- Does **not** create an ENI and does **not** have a security group.
- Works from the VPC route tables associated with the endpoint. It is not directly reachable from
  on-premises, a peered VPC, or through Transit Gateway.

```
Route table (private subnet)
┌──────────────────────────┬──────────────────┐
│ 10.0.0.0/16              │ local            │
│ pl-xxxx (S3 prefix list) │ vpce-gateway-S3  │ ← gateway endpoint route
└──────────────────────────┴──────────────────┘
```

### Interface Endpoint (AWS PrivateLink)

- An **Elastic Network Interface (ENI)** with a **private IP** placed *inside your subnet*. Traffic to the service goes to that ENI.
- Powered by **AWS PrivateLink**.
- Supports **most AWS services** (and partner/your-own services) — *not* just S3/DynamoDB.
- **Charged** per hour and per GB of data processed.
- Secured with a **security group** on the ENI (it's an ENI, so SGs apply).
- Uses **private DNS** so the service's normal hostname resolves to the endpoint's private IP
  transparently, when private DNS is enabled and the VPC has DNS support/hostnames enabled.

If private DNS is disabled, clients must use the endpoint-specific DNS name
(`vpce-...`) or an alias you create. If DNS resolves to the private IP but the
endpoint security group blocks inbound 443 from the workload, the symptom looks
like a normal connection timeout.

### Gateway Load Balancer Endpoint

- A VPC endpoint used to route traffic through a **Gateway Load Balancer** and a fleet of virtual
  appliances such as firewalls or IDS/IPS.
- Creates an endpoint ENI in a selected subnet/AZ and is used as a **route table target**.
- Powered by PrivateLink, billed hourly and per GB.
- Not a general AWS-service endpoint and not the right answer for S3, DynamoDB, or API access.

| Feature | Gateway Endpoint | Interface Endpoint (PrivateLink) | Gateway Load Balancer Endpoint |
|---------|------------------|----------------------------------|--------------------------------|
| Implemented as | Route table target | **ENI with a private IP** in your subnet | Endpoint ENI + route table target |
| Supported services | **S3 and DynamoDB only** | Most AWS services + partner/custom services | GWLB endpoint services for appliances |
| Cost | **Free** | Hourly + per-GB data processing | Hourly + per-GB data processing |
| Access control | Endpoint policy + route table | **Security group** + endpoint policy | Route tables, NACLs, appliance controls |
| DNS | Uses the service's public DNS (via prefix list route) | **Private DNS** -> resolves to the ENI's private IP | Routing construct, not service DNS |
| Cross-Region / on-prem reach | No | Yes (reachable over VPN/Direct Connect/peering, depending on DNS/routing) | Used for inspection paths |
| Scope | The VPC route tables associated with it | The subnets where you place endpoint ENIs | One endpoint subnet/AZ per endpoint |

> **Rule**: **S3 or DynamoDB → use a Gateway Endpoint** (free, route-table based). **Any other service, or you need access from on-prem/peered VPCs → use an Interface Endpoint (PrivateLink).** Memorize the "only S3 and DynamoDB are gateway" fact — it's tested constantly.

💡 S3 also supports an Interface Endpoint, used when you need to reach S3 privately from **on-premises** (over Direct Connect/VPN) — something a Gateway Endpoint can't do because gateway endpoints are reachable only from within the VPC's route tables.

### Endpoint policies

Endpoint policies are an extra filter, not a replacement for IAM or resource
policies. The default policy is usually full access unless you restrict it.
Common exam pattern:

- IAM allows the principal to call S3.
- A gateway endpoint policy limits which buckets can be reached through the endpoint.
- The S3 bucket policy requires `aws:sourceVpce` or `aws:sourceVpc`.

All three layers must allow the request. If any layer denies it, the request
fails even though the network path exists.

---

## 4. PrivateLink — Exposing Your Own Service Privately

PrivateLink isn't only for consuming AWS services. It lets one party **publish a service** (behind a Network Load Balancer) and other VPCs **consume it privately** through an Interface Endpoint — without VPC peering, without exposing anything to the internet, and **even if both sides use overlapping CIDRs** (because traffic terminates at the ENI, not via routed IPs).

```
 Provider VPC                          Consumer VPC
 ┌─────────────────────┐               ┌──────────────────────┐
 │  Service behind NLB │               │  App → Interface     │
 │   (Endpoint Service)│◄══PrivateLink═│  Endpoint ENI (10.x) │
 └─────────────────────┘               └──────────────────────┘
   only the NLB is exposed; consumer never sees provider's network
```

✅ PrivateLink is the answer when: a SaaS vendor exposes a service to many customer VPCs, or you must connect VPCs with **overlapping CIDRs** (which peering forbids), or you want to share *one specific service* without granting full network reachability.

---

## 5. Why Endpoints Matter (Security + Cost)

- ✅ **Security**: sensitive data to S3/DynamoDB/etc. never traverses the public internet; you can write endpoint policies and remove the NAT/IGW path entirely for service traffic.
- ✅ **Cost**: a Gateway Endpoint is free and removes NAT Gateway data-processing charges for S3/DynamoDB traffic — often a large bill reduction.
- ✅ **Compliance**: lets you run truly private subnets with **no internet path at all** while still using AWS services.

---

## 6. Key Exam Points

- **VPC Peering** = private, backbone connection between two VPCs (same/cross account, same/cross Region).
- Peering is **non-transitive** and **forbids overlapping CIDRs**; you must **add routes on both sides** and adjust SGs/NACLs.
- Enabling **DNS resolution on the peering connection** is required for private-IP DNS across the peer.
- **Gateway Endpoint** = route-table target, **S3 & DynamoDB only**, **free**.
- **Interface Endpoint (PrivateLink)** = ENI with a private IP, **most services**, **paid**, secured by a **security group**, uses **private DNS**.
- **Gateway Load Balancer Endpoint** = route target to appliance inspection through GWLB; not for normal AWS API access.
- Need S3/DynamoDB privately → Gateway. Need anything else, or on-prem/peer access → Interface.
- **PrivateLink** exposes a single service privately and works even across **overlapping CIDRs**.
- Endpoints keep traffic off the public internet → better security and lower NAT cost.

---

## Common Real-World Misconfigurations

- ❌ Creating a peering connection but forgetting route table entries on one or both VPCs.
- ❌ Expecting transitive routing (A→B→C) over peering — it doesn't exist; use Transit Gateway.
- ❌ Designing VPCs with overlapping CIDRs and then needing to peer them.
- ❌ Using a Gateway Endpoint and expecting it to work from on-premises — it won't; use an Interface Endpoint.
- ❌ Forgetting the security group on an Interface Endpoint, silently blocking the service.
- ❌ Thinking a gateway endpoint has an ENI or SG. It is a route-table target.
- ❌ Forgetting endpoint policies and bucket policies can still deny traffic after routing works.
- ❌ Paying for NAT Gateway data processing on heavy S3 traffic instead of adding a free Gateway Endpoint.

---

**Next**: [06_transit_gateway_and_advanced.md — Transit Gateway, Flow Logs, IPv6 & Advanced Routing](06_transit_gateway_and_advanced.md)
