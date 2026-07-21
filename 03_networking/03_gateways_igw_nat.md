# Internet Gateway, NAT, and Internet Access

> **Who this is for**: Engineers who understand [VPCs, subnets, and route tables](02_vpc_subnets_route_tables.md)
> and know that "public subnet" means a route to an Internet Gateway. This file explains the
> gateways themselves and how a *private* instance reaches the internet without being exposed.
> The NAT concept from the [primer](01_networking_primer.md#7️⃣-nat--network-address-translation)
> is the foundation here.

---

## 1. The Internet Gateway (IGW)

An **Internet Gateway** is the component that connects your VPC to the public internet. It's a horizontally scaled, redundant, highly available AWS-managed object — you don't size it, patch it, or worry about its availability.

- **One IGW per VPC** (and one VPC per IGW). You create it, then **attach** it to the VPC.
- An IGW does two jobs: it provides a target for internet-bound traffic in a route table, and it performs **1:1 NAT** between an instance's private IP and its **public** IP.

### What actually makes an instance reachable from the internet

Three conditions must *all* be true:

```
1. The subnet's route table has   0.0.0.0/0 → igw-xxxx       (public subnet)
2. The instance has a public IPv4 address (or Elastic IP)
3. Security Group + NACL allow the traffic                    (file 04)
```

```
          Internet
             │
             ▼
      ┌─────────────┐
      │ Internet GW │   attached to the VPC
      └──────┬──────┘
             │   route table:  0.0.0.0/0 → igw
   ┌─────────▼──────────┐
   │ PUBLIC subnet      │   EC2 with public IP  ◄──► internet
   │ 10.0.1.0/24        │
   └────────────────────┘
```

⚠️ Miss any one of the three conditions and connectivity silently fails. The IGW route + public IP + firewall rules must align. This is the #1 "why can't I reach my instance" support case.

> **Key insight**: The IGW gives a *public* instance two-way internet access. A *private* instance (no public IP, no IGW route) needs a different mechanism for outbound — NAT.

---

## 2. Elastic IPs

A normal public IPv4 address assigned to an instance is **dynamic** — stop/start the instance and it usually changes. An **Elastic IP (EIP)** is a **static, public IPv4 address you own** in your account and can attach to an instance (or NAT Gateway) and re-map at will.

- Stays the same across stop/start and can be moved between instances (useful for failover).
- ⚠️ AWS now **charges for every public IPv4 address**, including idle EIPs. You're billed for an EIP that isn't attached to a running resource — a classic surprise on the bill.
- A **zonal public NAT Gateway** uses an Elastic IP. A regional public NAT
  Gateway can manage public addresses automatically or use addresses you assign,
  depending on its mode.

💡 Modern best practice: don't put public IPs on application servers at all. Front them with a load balancer (public) and keep the instances private — fewer public IPs, smaller attack surface.

---

## 3. How a Private Subnet Reaches the Internet (Outbound Only)

Private-subnet instances often *must* reach the internet outbound — to download OS patches, pull container images, or call public AWS/third-party APIs — while remaining **unreachable from the internet inbound**. This is precisely the NAT pattern from the primer.

A traditional **zonal public NAT Gateway** lives in a **public** subnet (it needs
the IGW for its own internet access) and has an Elastic IP. Private subnets route
`0.0.0.0/0` to the NAT Gateway instead of the IGW. AWS also offers a **regional
public NAT Gateway**, covered in §4, which is attached to the VPC rather than a
public subnet and expands across AZs automatically.

### Packet flow: private EC2 → internet and back

```
 PRIVATE subnet                PUBLIC subnet             VPC edge       Internet
 ┌────────────┐               ┌──────────────┐          ┌────────┐
 │ EC2        │               │ NAT Gateway  │          │  IGW   │
 │ 10.0.11.50 │               │ + Elastic IP │          │        │
 └─────┬──────┘               └──────┬───────┘          └───┬────┘
       │                             │                      │         ┌──────────┐
  (1)  │ src 10.0.11.50 ───────────► │                      │         │  yum /   │
       │ route 0.0.0.0/0 → NAT       │                      │         │  apt /   │
       │                        (2)  │ rewrite src → EIP ─► │         │  api.com │
       │                             │              route 0.0.0.0/0 ─►│          │
       │                             │              → IGW   │ (3) ───►│          │
       │                             │                      │         │          │
       │                        (5)  │ ◄── rewrite dst ─────│ ◄──(4)──│  reply   │
  (6)  │ ◄── dst 10.0.11.50 ──────── │   back to private IP │         └──────────┘
       │                             │
```

1. Private EC2 sends an outbound packet; its route table directs `0.0.0.0/0` to the NAT Gateway.
2. NAT Gateway rewrites the **source** to its Elastic IP and remembers the mapping.
3. Traffic exits via the IGW to the internet.
4. The remote server replies to the Elastic IP.
5. NAT Gateway reverses the mapping back to `10.0.11.50`.
6. The reply reaches the private instance.

> **Rule**: The internet can never *initiate* a connection to the private instance through a NAT Gateway — only responses to connections the instance started come back. That's the security property: outbound yes, inbound no.

⚠️ For the zonal design shown here, putting the NAT Gateway in a *private*
subnet is a mistake. A zonal public NAT Gateway must live in a **public** subnet.
A regional public NAT Gateway does not use a customer-selected public subnet.

---

## 4. NAT Gateway vs NAT Instance

AWS offers two ways to do NAT. **NAT Gateway** is the managed service and the right answer in almost every exam scenario. **NAT Instance** is a self-managed EC2 instance running NAT software — legacy, but still tested as a comparison.

| Factor | NAT Gateway | NAT Instance |
|--------|-------------|--------------|
| Management | Fully managed by AWS | You manage the EC2 instance (patching, AMI) |
| Availability | Zonal: HA within one AZ; regional: expands across AZs | Single EC2 instance — you build HA yourself |
| Bandwidth | Up to 100 Gbps, scales automatically | Limited by the instance type you chose |
| Maintenance | None | OS patches, monitoring, scaling are on you |
| Performance | Consistent | Depends on instance size |
| Public IP | Public NAT needs public IPv4 addresses; zonal mode uses an EIP | Requires a public IP / EIP |
| Security Groups | **Cannot** attach an SG to a NAT GW | Can attach an SG (it's an EC2 instance) |
| Port forwarding / bastion | Not supported | Can act as a bastion / do port forwarding |
| Cost | Hourly + per-GB data processing | EC2 instance cost (can be cheaper at tiny scale) |
| **Exam default** | ✅ Choose this | Only if you need SG control, port forwarding, or cost at trivial scale |

A zonal NAT Gateway has a private IP and a requester-managed ENI in its subnet;
a regional NAT Gateway manages its networking across AZs at VPC scope. You
cannot attach a security group to either mode. Control the NAT path with route
tables, NACLs, and the security groups on the workloads that send traffic through
it.

### Zonal and regional NAT Gateway availability

A **zonal** NAT Gateway is redundant inside one AZ but is still an AZ-scoped
failure domain. If every private subnet routes through a zonal NAT Gateway in
AZ-a and that AZ fails, workloads in the other AZs lose internet egress too.

> **Rule for zonal NAT**: Deploy **one NAT Gateway per AZ**, and point each AZ's
> private route table at the NAT Gateway in its *own* AZ.

```
 AZ-a private subnet  ── 0.0.0.0/0 ──►  NAT GW in AZ-a
 AZ-b private subnet  ── 0.0.0.0/0 ──►  NAT GW in AZ-b
```

✅ This also avoids **cross-AZ data charges** that you'd pay if AZ-b's traffic crossed to a NAT GW in AZ-a.

The newer **regional public NAT Gateway** uses one NAT Gateway ID and maintains
zonal affinity while automatically expanding and contracting with the workload
footprint. It does not require public subnets and provides multi-AZ internet
egress by default. Automatic expansion into a newly used AZ can take time, so
traffic can temporarily be processed in another AZ. Regional NAT does **not**
support private NAT; use zonal NAT Gateways when the translation path is toward
private VPC or on-premises networks.

Exam questions may describe either design. Look for **"one per AZ"** or a NAT in
a public subnet for zonal mode, and **"single ID," "automatic multi-AZ,"** or
**"no public subnet"** for regional mode.

### Distributed versus centralized egress

At organization scale, decide where NAT and inspection live:

| Design | Packet path | Strengths | Costs and risks |
|--------|-------------|-----------|-----------------|
| **Distributed egress** | Workload VPC → local NAT → IGW | Simple routing; small VPC blast radius; zonal NAT can keep traffic in-AZ | NAT hourly cost repeated in every VPC; policies and public source IPs are harder to govern centrally |
| **Centralized egress VPC** | Spoke VPC → Transit Gateway → egress/inspection VPC → NAT → IGW | One policy and allow-list boundary; fewer NAT fleets; consistent inspection and logging | TGW and NAT processing charges; possible cross-AZ transfer; more routes; the egress VPC is a larger failure domain |

A typical centralized path is:

```
spoke private subnet: 0.0.0.0/0 -> TGW
TGW route table:       0.0.0.0/0 -> inspection/egress VPC attachment
inspection VPC:        TGW -> Network Firewall or GWLB endpoint -> NAT -> IGW
return traffic:        IGW -> NAT -> inspection -> TGW -> originating spoke
```

Stateful firewalls must see both directions of a flow. Keep the forward and
return route tables symmetric, deploy inspection endpoints in every used AZ,
and enable Transit Gateway **appliance mode** on the inspection VPC attachment
when required. See the [professional Transit Gateway design](06_transit_gateway_and_advanced.md#3-professional-transit-gateway-design).

Before centralizing NAT, remove traffic that does not need internet egress:

- Use free **gateway endpoints** for S3 and DynamoDB.
- Use **interface endpoints** for supported AWS APIs when their endpoint cost is
  lower than the NAT/TGW path or policy requires a private service path.
- Use private DNS and endpoint policies so applications use the endpoint rather
  than silently falling back to public service endpoints.

Endpoints reduce NAT data processing, public exposure, and inspection volume.
They do not replace NAT for arbitrary third-party APIs or software repositories.

---

## 5. IPv6: Egress-Only Internet Gateway and NAT64

Native IPv6 does not need address translation just to conserve addresses. To let
an IPv6 workload reach the IPv6 internet without permitting unsolicited inbound
connections, use an **Egress-Only Internet Gateway (EIGW)**:

| | Internet Gateway | Egress-Only Internet Gateway | NAT Gateway |
|---|---|---|---|
| IP version | IPv4 + IPv6 | **IPv6 only** | IPv4; also NAT64 for IPv6 clients reaching IPv4 |
| Direction | Inbound + outbound | **Outbound only** (stateful) | Outbound only (stateful) |
| Address translation | 1:1 NAT (IPv4) | None | IPv4 translation or IPv6-to-IPv4 NAT64 |
| Purpose | Public internet access | Private native-IPv6 outbound | IPv4 egress; IPv6-to-IPv4 translation |

> **Rule**: IPv6 workload → IPv6 internet uses an **Egress-Only IGW**. IPv4
> workload → IPv4 internet uses a **NAT Gateway**. An IPv6-only workload that
> must reach an **IPv4-only** destination uses Route 53 Resolver **DNS64** with a
> NAT Gateway performing **NAT64**.

---

## 6. Bastion Hosts (Jump Boxes) — Inbound Admin Access to Private Instances

NAT solves *outbound* for private instances. But how do you **SSH/RDP into** an instance that has no public IP? You don't expose it — you go *through* a hardened gateway in the public subnet called a **bastion host** (or **jump box**).

```
   Admin (office IP)                  VPC
        │                ┌──────────────────────────────────────┐
        │  SSH :22       │  PUBLIC subnet      PRIVATE subnet    │
        ▼                │  ┌───────────┐      ┌──────────────┐  │
   ────────────► IGW ───►│  │ Bastion   │─SSH─►│ App EC2      │  │
        SG allows :22    │  │ (public IP)│      │ (no public IP)│  │
        from MY IP only  │  └───────────┘      └──────────────┘  │
                         │   SG: only the bastion's SG can SSH ──┘
                         └──────────────────────────────────────┘
```

The bastion sits in a **public subnet** with a public IP; the private instances sit in **private subnets**. You SSH to the bastion, then from the bastion to the private instances. The security wins come from the **security group chaining**:

- **Bastion SG**: allow inbound `:22` (or `:3389`) **only from your corporate/admin IP range** — never `0.0.0.0/0`.
- **Private instance SG**: allow inbound `:22` **only from the bastion's security group** (reference the SG as the source, not an IP). Now nothing but the bastion can even attempt to connect.

> **Key insight**: A bastion shrinks the attack surface to **one** hardened, monitored, patched host. Private instances never expose SSH/RDP to the internet — only the bastion does, and only to known admin IPs.

⚠️ **Modern exam preference — SSM Session Manager.** AWS Systems Manager **Session Manager** lets you open a shell on a **private** instance with **no bastion, no open inbound port, no SSH key, and no public IP at all** — the instance's SSM Agent makes an *outbound* call to SSM (reachable via a NAT Gateway or, better, **interface VPC endpoints**). Access is IAM-controlled and fully logged to CloudTrail/S3.

| | **Bastion host** | **SSM Session Manager** |
|---|------------------|--------------------------|
| Inbound port open | Yes (`:22`/`:3389` from admin IPs) | **None** |
| Public IP / bastion EC2 | Required | **Not needed** |
| Access control | SSH keys + SG | **IAM policies** |
| Auditing | Build it yourself | **Built-in** (CloudTrail, session logs) |
| Exam signal | "jump host", legacy/required SSH | "no bastion / no open ports / no SSH keys" |

> **Rule**: If a question says *"access private instances **without** a bastion / without opening inbound ports / without SSH keys"* → **Session Manager**. If it explicitly wants a **jump host** → **bastion** with SG chaining and admin-IP-restricted inbound.

---

## 7. Key Exam Points

- **One IGW per VPC**; it must be **attached** and have a `0.0.0.0/0` route to make a subnet public.
- Public reachability needs **all three**: IGW route + public IP/EIP + firewall allow.
- **Elastic IP** = static public IPv4. Public IPv4 addresses are billed; zonal public NAT Gateways use EIPs.
- **NAT Gateway** = managed, scales to 100 Gbps, requester-managed networking, **no security group**.
- **NAT Instance** = self-managed EC2; only pick it for SG control, port forwarding, or trivial cost.
- **Zonal public NAT** sits in a public subnet; deploy **one per AZ** for
  resilience and zonal affinity. **Regional public NAT** uses one VPC-level ID,
  no public subnet, and automatic multi-AZ expansion.
- NAT is **outbound-initiated only** — the internet can't reach a private instance through it.
- **Egress-Only IGW** = native IPv6 outbound-only. **DNS64 + NAT64** lets an
  IPv6-only client reach an IPv4-only destination.
- At scale, compare distributed NAT with centralized TGW egress, include both
  processing and cross-AZ charges, and bypass NAT with VPC endpoints where possible.
- **Bastion host** = hardened jump box in a **public** subnet; private instance SG allows SSH **only from the bastion's SG**; bastion SG allows inbound **only from admin IPs**.
- **SSM Session Manager** = access private instances with **no bastion, no inbound port, no SSH key, no public IP** — IAM-controlled and logged. Preferred when the question rules out a bastion.

---

## Common Real-World Misconfigurations

- ❌ Placing a **zonal public** NAT Gateway in a private subnet — it then has no internet access itself.
- ❌ Routing every AZ to one **zonal** NAT Gateway — an AZ outage kills outbound for other AZs and cross-AZ data charges apply.
- ❌ Expecting inbound connections to a private instance through NAT. NAT is outbound only.
- ❌ Sending native IPv6 internet traffic to NAT instead of an Egress-Only IGW,
  or expecting an EIGW to translate IPv6 clients to IPv4 destinations.
- ❌ Centralizing egress without symmetric inspection routes, turning a stateful
  firewall or the egress VPC into a traffic black hole.
- ❌ Paying NAT and TGW processing charges for S3, DynamoDB, or supported AWS
  APIs that could use VPC endpoints.
- ❌ Leaving Elastic IPs allocated but unattached and getting billed for them.
- ❌ Public IP assigned but the subnet's route table lacks the IGW route — instance still unreachable.
- ❌ Opening the bastion's SSH port to `0.0.0.0/0` instead of your admin IP range — it becomes a brute-force target.
- ❌ Putting a bastion in a *private* subnet (it then has no inbound path) or giving private instances public IPs "to SSH in" instead of chaining through the bastion / using Session Manager.

---

**Next**: [04_security_groups_vs_nacls.md — The Two Firewall Layers](04_security_groups_vs_nacls.md)
