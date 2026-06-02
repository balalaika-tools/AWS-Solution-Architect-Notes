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
- A NAT Gateway **requires** an Elastic IP (it needs a stable public face for the internet).

💡 Modern best practice: don't put public IPs on application servers at all. Front them with a load balancer (public) and keep the instances private — fewer public IPs, smaller attack surface.

---

## 3. How a Private Subnet Reaches the Internet (Outbound Only)

Private-subnet instances often *must* reach the internet outbound — to download OS patches, pull container images, or call public AWS/third-party APIs — while remaining **unreachable from the internet inbound**. This is precisely the NAT pattern from the primer.

A **NAT Gateway** lives in a **public** subnet (it needs the IGW for its own internet access) and an Elastic IP. Private subnets route `0.0.0.0/0` to the NAT Gateway instead of the IGW.

### Packet flow: private EC2 → internet and back

```
 PRIVATE subnet                PUBLIC subnet            VPC edge       Internet
 ┌────────────┐               ┌──────────────┐         ┌────────┐
 │ EC2        │               │ NAT Gateway  │         │  IGW   │
 │ 10.0.11.50 │               │ + Elastic IP │         │        │
 └─────┬──────┘               └──────┬───────┘         └───┬────┘
       │                             │                     │         ┌──────────┐
  (1)  │ src 10.0.11.50 ───────────► │                     │         │  yum /   │
       │ route 0.0.0.0/0 → NAT       │                     │         │  apt /   │
       │                        (2)  │ rewrite src → EIP ─► │         │  api.com │
       │                             │              route 0.0.0.0/0 ─►│          │
       │                             │              → IGW   │ (3) ───►│          │
       │                             │                      │         │          │
       │                        (5)  │ ◄── rewrite dst ───── │ ◄──(4)──│  reply   │
  (6)  │ ◄── dst 10.0.11.50 ──────── │   back to private IP  │         └──────────┘
       │                             │
```

1. Private EC2 sends an outbound packet; its route table directs `0.0.0.0/0` to the NAT Gateway.
2. NAT Gateway rewrites the **source** to its Elastic IP and remembers the mapping.
3. Traffic exits via the IGW to the internet.
4. The remote server replies to the Elastic IP.
5. NAT Gateway reverses the mapping back to `10.0.11.50`.
6. The reply reaches the private instance.

> **Rule**: The internet can never *initiate* a connection to the private instance through a NAT Gateway — only responses to connections the instance started come back. That's the security property: outbound yes, inbound no.

⚠️ A common mistake: putting the NAT Gateway in a *private* subnet. It must live in a **public** subnet, because the NAT Gateway itself needs the IGW route to reach the internet.

---

## 4. NAT Gateway vs NAT Instance

AWS offers two ways to do NAT. **NAT Gateway** is the managed service and the right answer in almost every exam scenario. **NAT Instance** is a self-managed EC2 instance running NAT software — legacy, but still tested as a comparison.

| Factor | NAT Gateway | NAT Instance |
|--------|-------------|--------------|
| Management | Fully managed by AWS | You manage the EC2 instance (patching, AMI) |
| Availability | Highly available **within one AZ** automatically | Single EC2 instance — you build HA yourself |
| Bandwidth | Up to 100 Gbps, scales automatically | Limited by the instance type you chose |
| Maintenance | None | OS patches, monitoring, scaling are on you |
| Performance | Consistent | Depends on instance size |
| Public IP | Requires an Elastic IP | Requires a public IP / EIP |
| Security Groups | **Cannot** attach an SG to a NAT GW | Can attach an SG (it's an EC2 instance) |
| Port forwarding / bastion | Not supported | Can act as a bastion / do port forwarding |
| Cost | Hourly + per-GB data processing | EC2 instance cost (can be cheaper at tiny scale) |
| **Exam default** | ✅ Choose this | Only if you need SG control, port forwarding, or cost at trivial scale |

A NAT Gateway still has a private IP and a requester-managed ENI in its subnet,
but you cannot edit that ENI like a normal EC2 ENI and you cannot attach a
security group. Control the NAT path with route tables, NACLs, and the security
groups on the workloads that send traffic through it.

### NAT Gateway and high availability

A NAT Gateway is HA *within its own AZ*, but it lives in **one** AZ. If that AZ fails, private subnets routing through it lose internet access.

> **Rule**: For AZ-resilient outbound, deploy **one NAT Gateway per AZ**, and point each AZ's private route table at the NAT Gateway in its *own* AZ.

```
 AZ-a private subnet  ── 0.0.0.0/0 ──►  NAT GW in AZ-a
 AZ-b private subnet  ── 0.0.0.0/0 ──►  NAT GW in AZ-b
```

✅ This also avoids **cross-AZ data charges** that you'd pay if AZ-b's traffic crossed to a NAT GW in AZ-a.

Most SAA-style questions assume this zonal NAT Gateway pattern: one NAT Gateway
per AZ, with each private subnet routing to the NAT Gateway in its own AZ.

---

## 5. IPv6: Egress-Only Internet Gateway

NAT exists largely because IPv4 addresses are scarce — you hide many private hosts behind one public IP. **IPv6 has no shortage**, so AWS gives every IPv6 instance a globally routable address; there is **no NAT for IPv6** in AWS.

But you still often want IPv6 instances to reach out *without* being reachable inbound. That's the **Egress-Only Internet Gateway (EIGW)**:

| | Internet Gateway | Egress-Only Internet Gateway | NAT Gateway |
|---|---|---|---|
| IP version | IPv4 + IPv6 | **IPv6 only** | IPv4 only |
| Direction | Inbound + outbound | **Outbound only** (stateful) | Outbound only (stateful) |
| Address translation | 1:1 NAT (IPv4) | None — IPv6 is globally routable | Many-to-one NAT |
| Purpose | Public internet access | Private IPv6 outbound | Private IPv4 outbound |

> **Rule**: For private *IPv6* outbound use an **Egress-Only IGW**. For private *IPv4* outbound use a **NAT Gateway**. Don't mix them up — a NAT Gateway does nothing for IPv6.

---

## 6. Key Exam Points

- **One IGW per VPC**; it must be **attached** and have a `0.0.0.0/0` route to make a subnet public.
- Public reachability needs **all three**: IGW route + public IP/EIP + firewall allow.
- **Elastic IP** = static public IPv4. Now billed even when idle. NAT GW requires one.
- **NAT Gateway** = managed, HA within an AZ, scales to 100 Gbps, requester-managed ENI, **no security group**.
- **NAT Instance** = self-managed EC2; only pick it for SG control, port forwarding, or trivial cost.
- NAT GW must sit in a **public** subnet; deploy **one per AZ** for resilience and to avoid cross-AZ charges.
- NAT is **outbound-initiated only** — the internet can't reach a private instance through it.
- **Egress-Only IGW** = IPv6 outbound-only (IPv6 has no NAT in AWS).

---

## Common Real-World Misconfigurations

- ❌ Placing the NAT Gateway in a private subnet — it then has no internet access itself.
- ❌ A single NAT Gateway for all AZs — an AZ outage kills outbound for other AZs, plus cross-AZ data fees.
- ❌ Expecting inbound connections to a private instance through NAT. NAT is outbound only.
- ❌ Using a NAT Gateway for IPv6 traffic — it's IPv4 only; you need an Egress-Only IGW.
- ❌ Leaving Elastic IPs allocated but unattached and getting billed for them.
- ❌ Public IP assigned but the subnet's route table lacks the IGW route — instance still unreachable.

---

**Next**: [04_security_groups_vs_nacls.md — The Two Firewall Layers](04_security_groups_vs_nacls.md)
