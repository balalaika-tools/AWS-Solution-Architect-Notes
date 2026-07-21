# Hybrid Networking: Connecting On-Prem to AWS

> **Who this is for**: An engineer who understands a VPC, route tables, and the idea of a
> Transit Gateway, but has never wired a corporate data center into AWS. No prior WAN,
> BGP, or IPsec experience assumed. Builds on
> [Transit Gateway](../03_networking/06_transit_gateway_and_advanced.md).

---

## 1. Why Hybrid at All?

Most companies don't start in the cloud — they start with a **data center** full of servers,
databases, file shares, and Active Directory. Moving everything to AWS overnight is risky and
slow, so for months or years they run **hybrid**: some workloads on-prem, some in AWS, and the
two need to talk over a **private, reliable** path.

Common reasons you need a hybrid link:

- **Migration in progress** — an app in AWS still calls a database that's still on-prem.
- **Data gravity / compliance** — a system of record can't move yet, but new analytics run in AWS.
- **Bursting** — extra compute in AWS during peak, anchored to on-prem data.
- **DR target** — AWS is the recovery site for an on-prem workload (covered in
  [04_disaster_recovery.md](04_disaster_recovery.md)).

> **Key insight**: Hybrid is not "internet access to AWS." Anyone can reach AWS public
> endpoints over the internet. Hybrid networking is about reaching **private** resources in
> your VPC (RDS in a private subnet, internal EC2) as if they were on the same LAN — with
> private IPs, no public exposure.

The two foundational paths are **Site-to-Site VPN** and **Direct Connect**. Production designs
then combine them with Transit Gateway, redundant customer routers and sometimes SD-WAN or a
managed network provider. The architectural question is therefore not only *VPN or DX?*, but
also where routes terminate, how paths fail over, and who operates each failure domain.

---

## 2. The On-Prem-Side Building Blocks

Before comparing the two services, learn the three components that show up in every diagram.

| Component | Lives where | What it is |
|-----------|-------------|------------|
| **Customer Gateway (CGW)** | AWS-side *config object* representing your on-prem router | A resource in AWS that records your on-prem device's public IP and routing info (BGP ASN). It is metadata; the physical device is yours. |
| **Virtual Private Gateway (VGW)** | Attached to **one** VPC | The AWS-managed VPN concentrator on the VPC side. Terminates the VPN/DX into a single VPC. |
| **Transit Gateway (TGW)** | Regional hub | A router that connects **many** VPCs *and* on-prem links. Use instead of a VGW when you have more than one VPC or want hub-and-spoke. |

```
  ON-PREM                          AWS
 ┌──────────┐                  ┌──────────────┐
 │ Customer │   private link   │ VGW  (1 VPC) │
 │  Router  │═════════════════>│   OR         │
 │  (CGW)   │                  │ TGW (many VPC)│
 └──────────┘                  └──────────────┘
```

> **Rule**: VGW terminates into a single VPC. The moment the answer involves *multiple VPCs*,
> *multiple accounts*, or a *hub*, the exam wants **Transit Gateway**.

---

## 3. Site-to-Site VPN

A **Site-to-Site VPN** builds an **IPsec** tunnel between your on-prem Customer Gateway device
and AWS (a VGW or TGW). The traffic still travels over the **public internet**, but it is
**encrypted** end to end.

```
 On-prem router ──IPsec tunnel (encrypted)──▶  Internet  ──▶  VGW / TGW ──▶ VPC
```

Key facts:

- **Fast to set up** — minutes to hours. No physical work, no telco.
- **Encrypted by default** — that's the whole point of IPsec.
- Each AWS VPN connection gives you **two tunnels** to two different AWS endpoints for
  redundancy (use both for HA).
- **Throughput** is capped (~1.25 Gbps per tunnel) and **latency is variable** because it
  rides the public internet.
- Supports **static routing** or **dynamic (BGP)** routing.

✅ Choose VPN when you need connectivity *now*, traffic volume is modest, and some latency
jitter is acceptable.

⚠️ A single VPN tunnel is not highly available. Use both AWS tunnels and, for stronger failure
isolation, separate customer gateway devices and provider paths. **Accelerated VPN** uses the AWS
global network to improve path performance; it does not replace router/tunnel redundancy.

---

## 4. AWS Direct Connect (DX)

**Direct Connect** is a **dedicated physical connection** between your network and AWS, set up
at a **DX location** (a co-location facility where AWS has a presence). Your traffic **bypasses
the public internet entirely**.

```
 On-prem ──cross-connect──▶ DX Location ──AWS private backbone──▶ VPC
```

Key facts:

- **Consistent, low latency** and **predictable bandwidth** (1, 10, 100, or 400 Gbps dedicated;
  smaller hosted connections available via partners).
- **Lower data-transfer cost** at high volume than internet egress.
- **Takes weeks to months** to provision — it's physical cabling and telco coordination.
- ❌ **Not encrypted by default.** A private circuit is not the same as an encrypted one.
- A **Direct Connect Gateway (DXGW)** lets one DX connection reach **multiple VPCs in
  multiple Regions** (globally, except China), and connect to VGWs or Transit Gateways.

Direct Connect carries traffic on **Virtual Interfaces (VIFs)**:

| VIF type | Reaches | Use for |
|----------|---------|---------|
| **Private VIF** | VPC private resources (via VGW or DXGW) | Private subnets, RDS, internal EC2 |
| **Public VIF** | AWS public services (S3, DynamoDB public endpoints) over the DX line | High-volume access to public AWS services without the internet |
| **Transit VIF** | Transit Gateway (via DXGW) | Hub-and-spoke to many VPCs |

💡 A single DX link is itself a single point of failure (one cable, one location). For
resilience, AWS recommends a **second DX connection** (ideally at a second location), or a
**Site-to-Site VPN as backup** (next section).

---

## 5. VPN vs Direct Connect — Comparison

| Dimension | Site-to-Site VPN | Direct Connect (DX) |
|-----------|------------------|---------------------|
| Path | Public internet | Dedicated private link |
| Encryption | ✅ Encrypted (IPsec) by default | ❌ Not encrypted by default |
| Setup time | Minutes–hours | Weeks–months (physical) |
| Latency | Variable (internet) | Consistent / low |
| Bandwidth | Up to ~1.25 Gbps per tunnel | 1 / 10 / 100 / 400 Gbps dedicated; hosted options vary |
| Cost model | Low fixed + internet data | Port-hours + lower per-GB transfer |
| Best for | Quick, modest, encrypted | High volume, steady, low-latency |
| Terminates at | VGW or TGW | VGW / DXGW / TGW (via Transit VIF) |

---

## 6. Combining Them: VPN over Direct Connect, and DX + VPN Backup

These two combinations are favorite exam distractors — know which problem each solves.

**VPN over Direct Connect — for *encryption*.**
DX is private but unencrypted. If a compliance requirement says *"all traffic must be
encrypted in transit,"* a private circuit isn't enough. You run an **IPsec VPN inside the
Direct Connect connection** (over a public VIF). You get DX's consistent performance **and**
encryption.

```
 On-prem ──[ IPsec VPN ]── over ── Direct Connect ──▶ AWS
            (encryption)        (private, fast path)
```

**Site-to-Site VPN as a *backup* for DX — for *resilience*.**
DX is a single physical link. Configure a VPN over the internet as a cheap standby. If the DX
circuit drops, BGP fails traffic over to the VPN automatically (lower bandwidth, but
connectivity stays up).

```
 On-prem ══Direct Connect (primary)══▶ AWS
        └─Site-to-Site VPN (backup)──▶ AWS   ← takes over if DX fails
```

| Goal | Solution |
|------|----------|
| Need **encryption** on a DX link | **VPN over Direct Connect** |
| Need **failover** if DX goes down | **Direct Connect + VPN backup** |
| Need it **cheap and quick**, encryption matters | **Site-to-Site VPN** alone |
| Need **high, steady throughput**, low latency | **Direct Connect** (add VPN for the gaps) |

---

## 7. Choosing for the Exam

> **Mental model**: Read the requirement keywords, not the service names.

- *"Quickest to set up", "encrypted", "temporary", "low cost"* → **Site-to-Site VPN**
- *"Consistent/predictable latency", "high bandwidth", "dedicated", "large data transfer"* → **Direct Connect**
- *"Encrypted **and** private/consistent"* → **VPN over Direct Connect**
- *"Resilient / highly available connection to on-prem"* → **DX + VPN backup** (or dual DX)
- *"Connect many VPCs / accounts to on-prem"* → terminate on a **Transit Gateway** (with a Transit VIF for DX)

---

## 8. Direct Connect Beyond a Single Circuit

A production DX design separates the **physical connection**, the **virtual interface**, and
the **AWS gateway**. These are independent choices:

| Design choice | Use it when | Important constraint |
|---------------|-------------|----------------------|
| **Dedicated connection** | You want an AWS port allocated to one customer and direct control of the BGP sessions | You arrange the cross-connect at the DX location; available port speeds and MACsec support depend on the location and connection type. |
| **Hosted connection** | An AWS Direct Connect Partner supplies the last mile or a sub-rate connection | The partner owns part of the provisioning and operational boundary; confirm bandwidth, redundancy, monitoring and encryption with it. |
| **Private VIF → VGW** | One connection reaches private addresses in one VPC | Simple, but a VGW is not a multi-VPC hub. |
| **Private VIF → DX gateway → VGWs** | One hybrid connection reaches VPCs in multiple Regions | A DX gateway is a global routing resource, not a transit router between attached VPCs. |
| **Transit VIF → DX gateway → TGW** | A multi-account network uses Transit Gateway as its Regional hub | The TGW owns VPC segmentation and routing; use more than one TGW/Region when the recovery design requires it. |
| **Public VIF** | On-premises needs AWS public service prefixes without using the public internet path | Endpoints still use public IP addresses; control and filter the large set of public routes deliberately. |

**Direct Connect SiteLink** can carry traffic between supported DX points of presence over the
AWS backbone instead of hairpinning through a VPC or the public WAN. It is useful for connecting
branches or data centers, but it does not replace the need to design VPC/TGW reachability,
inspection, and a backup path. Account for SiteLink data-processing charges before choosing it
over an existing WAN.

### Encryption choices

DX supplies a private path, not automatic encryption. Choose one of these controls when policy
requires encryption in transit:

- **MAC Security (MACsec)** protects frames on selected **10, 100, and 400 Gbps dedicated**
  connections and supported locations. It is not available on hosted connections; verify the port,
  router and location before selecting it.
- **IPsec VPN over a public VIF** gives network-layer encryption and can terminate on a VGW or
  TGW. It adds tunnel throughput, MTU and operational constraints.
- **Application TLS** is still valuable end to end, even when the circuit or tunnel is encrypted.

### Redundancy and BGP policy

Start from the business failure requirement, then select a topology:

1. **Development** may accept one DX with two VPN tunnels as backup.
2. **High resilience** normally uses connections in separate DX locations, separate on-premises
   routers and diverse provider paths. Do not treat two virtual interfaces on one physical port
   as two failure domains.
3. Run both paths **active-active** only when the application and on-premises network tolerate
   multipath and asymmetric flows. Use **active-passive** when deterministic troubleshooting is
   more important; influence outbound and inbound selection with BGP attributes and advertised
   prefixes rather than static-route surprises.
4. Test failure of a router, circuit, location and BGP session independently. A green interface
   does not prove that an application route, DNS answer or return path works.

For multi-Region recovery, associate the DX gateway with the required Regional gateways and
advertise only the intended prefixes. Keep a Region-independent path—often VPN—if one Regional
TGW is part of the failure scenario. Check that the standby Region has quotas, capacity, DNS,
security controls and return routes before calling the network resilient.

---

## 9. Hybrid Connectivity Troubleshooting Runbook

Troubleshoot from routing evidence, not from the symptom `timeout` alone:

1. **Bound the failure.** Record source/destination IP and port, DNS answer, timestamp, Region,
   account, VPC, subnet and expected path. Test both directions where the protocol allows it.
2. **Check the physical and control planes.** Inspect DX connection/VIF state or both VPN tunnels,
   BGP session state, recently changed prefixes and AWS/on-premises router logs. A VIF can be up
   while the needed route is absent.
3. **Compare advertisements.** Confirm the exact prefixes accepted on each side, longest-prefix
   matches, summarization and any AS-path or local-preference policy. Reject overlapping CIDRs
   before attachment; NAT or renumbering is usually required when overlap cannot be removed.
4. **Walk AWS routing.** Inspect VGW/TGW attachment state, TGW route-table association and
   propagation, VPC subnet routes, security groups, NACLs and inspection-appliance routes.
   Asymmetric inspection is a common cause of one-way sessions.
5. **Check packet size and translation.** Compare MTU/MSS across DX, VPN and appliances; an ICMP
   test can succeed while larger TLS packets fail. Verify NAT addresses and stateful firewall
   expectations in both directions.
6. **Check hybrid DNS separately.** Validate Route 53 Resolver inbound/outbound endpoints,
   shared rules, forwarding targets and on-premises conditional forwarders. Network reachability
   does not prove name resolution.
7. **Use telemetry with its limits.** DX/VPN CloudWatch metrics show link or tunnel health; VPC
   Flow Logs show accepted/rejected flows at supported interfaces, not the physical DX segment.
   Reachability Analyzer validates modeled AWS paths but does not send packets or prove the
   external WAN. Network Access Analyzer finds paths that violate a policy; it is not a live
   packet test.
8. **Check service quotas and capacity.** Include VIFs, routes, TGW attachments, VPN connections,
   Resolver endpoints and firewall capacity in pre-cutover reviews.

Capture the known-good BGP table, TGW/VPC routes, DNS path and representative Flow Log records
before migration. That baseline makes rollback decisions faster than trying to reconstruct the
old state during an outage.

### Worked failure: a more-specific route creates an asymmetric path

After a maintenance change, an on-premises application can open small TCP connections to
`10.40.8.25` in AWS, but large TLS requests time out. The intended forward and return path is the
primary DX; VPN should be standby.

Evidence shows:

- DX and both VPN tunnels are `UP` in CloudWatch, so link state alone does not explain the fault.
- AWS advertises the summarized `10.40.0.0/16` over DX. An on-premises router accidentally
  advertises the more-specific application source prefix over VPN, so AWS longest-prefix routing
  returns that traffic through VPN while requests arrive over DX.
- TGW attachment/route tables and VPC routes otherwise select the expected attachments. Flow Logs
  show accepted request/return ENI traffic, while the stateful on-premises firewall records the
  return flow on the unexpected tunnel.
- A small ping succeeds, but TLS payloads fail because the VPN leg adds encapsulation and the path's
  MTU/MSS handling is wrong. Resolver query logs show the correct private address, eliminating DNS
  as the cause. Reachability Analyzer confirms the modeled AWS segment only; it does not claim the
  external BGP/MTU path is healthy.

The team withdraws the accidental more-specific VPN advertisement, restores the intended BGP
policy, and clamps MSS/fixes path-MTU discovery for the backup tunnel before retesting. It then
checks advertised/accepted routes, TGW/VIF/VPN quotas, both flow directions, DNS, large TLS payloads
and forced DX failover. The durable fix adds a route-policy test comparing approved prefixes on
both customer routers; merely restarting the DX VIF would have hidden none of the causes.

---

## Key Exam Points

- **VPN = internet + encrypted + fast to deploy; DX = private + unencrypted + slow to deploy.**
- DX is **not encrypted by default** — add a VPN over it when encryption is required.
- DX takes **weeks** to provision; if a question stresses *urgency*, the answer is VPN.
- **Customer Gateway** = AWS object for your router; **VGW** = single-VPC concentrator;
  **Transit Gateway** = multi-VPC hub.
- **Direct Connect Gateway** lets one DX reach VPCs across **multiple Regions**.
- A Site-to-Site VPN connection provides **two tunnels** for redundancy.
- A resilient DX design removes shared routers, facilities and provider paths; two logical VIFs
  on one port are not physical redundancy.
- VPN-as-backup-for-DX gives **resilience**; VPN-over-DX gives **encryption** — don't confuse them.

---

## Common Mistakes

- ❌ Assuming Direct Connect is encrypted because it's "private." It is not — that's a
  classic trap.
- ❌ Picking Direct Connect when the requirement is *speed of deployment*. DX can't be stood
  up in a hurry.
- ❌ Using a single VGW when the scenario clearly has multiple VPCs — that's a Transit Gateway.
- ❌ Confusing **VPN over DX** (solves encryption) with **VPN backup for DX** (solves failover).
- ❌ Thinking a single DX line or a single VPN tunnel is "highly available." Real HA needs
  redundancy on both.
- ❌ Using Flow Logs or Reachability Analyzer as proof that the on-premises WAN is healthy. Each
  observes only part of the end-to-end path.

---

**Next**: [02_migration_services.md — Migration Services: Moving Workloads into AWS](02_migration_services.md)
