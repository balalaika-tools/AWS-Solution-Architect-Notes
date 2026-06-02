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

There are exactly two ways AWS gives you that private path, and the entire exam topic is
choosing between them: **Site-to-Site VPN** and **Direct Connect**.

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

⚠️ A single VPN tunnel is not highly available. For real HA, terminate two tunnels, or use two
customer gateway devices, or use **Accelerated VPN** (over the AWS Global Accelerator network)
to reduce jitter.

---

## 4. AWS Direct Connect (DX)

**Direct Connect** is a **dedicated physical connection** between your network and AWS, set up
at a **DX location** (a co-location facility where AWS has a presence). Your traffic **bypasses
the public internet entirely**.

```
 On-prem ──cross-connect──▶ DX Location ──AWS private backbone──▶ VPC
```

Key facts:

- **Consistent, low latency** and **predictable bandwidth** (1, 10, or 100 Gbps dedicated;
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
| Bandwidth | Up to ~1.25 Gbps per tunnel | 1 / 10 / 100 Gbps dedicated |
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

## Key Exam Points

- **VPN = internet + encrypted + fast to deploy; DX = private + unencrypted + slow to deploy.**
- DX is **not encrypted by default** — add a VPN over it when encryption is required.
- DX takes **weeks** to provision; if a question stresses *urgency*, the answer is VPN.
- **Customer Gateway** = AWS object for your router; **VGW** = single-VPC concentrator;
  **Transit Gateway** = multi-VPC hub.
- **Direct Connect Gateway** lets one DX reach VPCs across **multiple Regions**.
- A Site-to-Site VPN connection provides **two tunnels** for redundancy.
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

---

**Next**: [02_migration_services.md — Migration Services: Moving Workloads into AWS](02_migration_services.md)
