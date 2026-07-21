# Connecting Two VPCs with VPC Peering

> **Who this is for**: Engineers who need two VPCs to talk over private IPs — e.g. a shared
> services VPC and an app VPC. Builds on the [reference VPC](01_vpc_public_private_subnets.md)
> and the firewall rules from [file 04](04_sg_vs_nacl_examples.md). Concept reference:
> [VPC Peering & Endpoints](../03_networking/05_vpc_peering_and_endpoints.md). DNS resolution
> across the peering link is covered next, in [file 06](06_vpc_peering_dns_resolution.md).

---

## 1. The Goal and the Hard Constraint

We have two VPCs and want resources in one to reach resources in the other privately, without going over the internet:

- **VPC-A**: `10.0.0.0/16`
- **VPC-B**: `10.1.0.0/16`

A **VPC peering connection** is a direct, private networking link between two VPCs. Traffic stays on the AWS backbone.

> **Rule**: The two VPC CIDRs **must not overlap**. Peering `10.0.0.0/16` with another `10.0.0.0/16` is impossible — the router couldn't tell which side an address belongs to. Plan non-overlapping ranges up front; you cannot fix this after the fact without re-IPing a VPC.

Four things must all be done — miss any one and connectivity silently fails:
1. **Create** the peering connection (requester side).
2. **Accept** it (accepter side).
3. **Update the route table on BOTH sides** — peering is not automatic routing.
4. **Update security groups / NACLs** to allow the peer's traffic.

---

## 2. ASCII Topology

```
┌──────────── VPC-A 10.0.0.0/16 ────────────┐      ┌──────────── VPC-B 10.1.0.0/16 ────────────┐
│                                            │      │                                            │
│  EC2-A  10.0.10.25                         │      │  EC2-B  10.1.10.40                         │
│     │                                      │      │     │                                      │
│  RT-A:                                     │      │  RT-B:                                     │
│   10.0.0.0/16 → local                      │      │   10.1.0.0/16 → local                      │
│   10.1.0.0/16 → pcx-abc123  ───────┐       │      │   10.0.0.0/16 → pcx-abc123  ───────┐       │
│                                    │       │      │                                    │       │
└────────────────────────────────────│───────┘      └────────────────────────────────────│───────┘
                                     │                                                    │
                                     └──────────────► pcx-abc123 ◄──────────────────────┘
                                            VPC Peering Connection
                                       (private, on the AWS backbone)
```

Both route tables get a route to the *other* VPC's CIDR, with the **peering connection (`pcx-...`)** as the target. The traffic never touches an IGW or NAT.

---

## 3. Non-Transitive Limitation

> **Rule**: VPC peering is **non-transitive**. If A peers with B, and B peers with C, then **A cannot reach C** through B. You'd need a separate A↔C peering.

```
   A ◄──peer──► B ◄──peer──► C

   A → B   ✅ (direct peering)
   B → C   ✅ (direct peering)
   A → C   ❌ NOT routed through B — peering does not chain
```

This is why a full mesh of N VPCs needs N(N-1)/2 peering connections, and why **Transit Gateway** exists for hub-and-spoke at scale (see [Transit Gateway](../03_networking/06_transit_gateway_and_advanced.md)). For two or three VPCs, peering is simplest and cheapest.

---

## 4. Route Tables (both sides)

**VPC-A route table** (`RT-A`):

| Destination | Target | Meaning |
|-------------|--------|---------|
| `10.0.0.0/16` | `local` | Within VPC-A (implicit) |
| `10.1.0.0/16` | `pcx-abc123` | Reach VPC-B via the peering connection |

**VPC-B route table** (`RT-B`):

| Destination | Target | Meaning |
|-------------|--------|---------|
| `10.1.0.0/16` | `local` | Within VPC-B (implicit) |
| `10.0.0.0/16` | `pcx-abc123` | Reach VPC-A via the peering connection |

⚠️ Both directions are needed. A request from A to B might route fine, but if `RT-B` lacks the return route to `10.0.0.0/16`, B's reply has nowhere to go and the connection hangs. **Routing is per-direction; you configure both.**

---

## 5. Security Groups Across the Peering

Within the **same region**, an SG in one VPC can reference an SG in the **peered** VPC by ID — just like the ALB→instance chaining in [file 03](03_alb_to_private_ec2.md), but across the peering connection.

**EC2-B's security group** — allow MySQL from EC2-A's SG:

| Direction | Protocol | Port | Source | Notes |
|-----------|----------|------|--------|-------|
| Inbound | TCP | 3306 | `sg-ec2a` (peer SG) | Only EC2-A may connect to the DB |

If SG referencing isn't available (cross-region peering does **not** support referencing peer SGs), fall back to referencing the **peer VPC's CIDR**:

| Direction | Protocol | Port | Source | Notes |
|-----------|----------|------|--------|-------|
| Inbound | TCP | 3306 | `10.0.0.0/16` | All of VPC-A (coarser, but works cross-region) |

💡 Same-region: reference the peer **SG ID** (precise, follows instances). Cross-region: reference the peer **CIDR**.

---

## 6. Build It

**CLI:**

```bash
# 1. REQUESTER (VPC-A side): create the peering request
PCX=$(aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-0aaa \
  --peer-vpc-id vpc-0bbb \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

# 2. ACCEPTER (VPC-B side): accept it — without this it stays pending
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id "$PCX"

# 3. Add the route on BOTH route tables
aws ec2 create-route --route-table-id rtb-A \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id "$PCX"

aws ec2 create-route --route-table-id rtb-B \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id "$PCX"

# 4. Allow the traffic on VPC-B's SG, referencing VPC-A's SG (same region)
aws ec2 authorize-security-group-ingress \
  --group-id sg-ec2b --protocol tcp --port 3306 \
  --source-group sg-ec2a
```

**Terraform-style HCL:**

```hcl
resource "aws_vpc_peering_connection" "a_to_b" {
  vpc_id      = aws_vpc.a.id   # requester
  peer_vpc_id = aws_vpc.b.id   # accepter
  auto_accept = true           # same-account, same-region can auto-accept
  tags        = { Name = "a-to-b" }
}

resource "aws_route" "a_to_b" {
  route_table_id            = aws_route_table.a.id
  destination_cidr_block    = "10.1.0.0/16"
  vpc_peering_connection_id = aws_vpc_peering_connection.a_to_b.id
}

resource "aws_route" "b_to_a" {
  route_table_id            = aws_route_table.b.id
  destination_cidr_block    = "10.0.0.0/16"
  vpc_peering_connection_id = aws_vpc_peering_connection.a_to_b.id
}
```

⚠️ `auto_accept` only works for same-account, same-region peering. **Cross-account or cross-region** peering must be accepted explicitly by the other side (the request sits in `pending-acceptance`).

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Peering stuck `pending-acceptance` | Accepter never accepted | Accept on the peer side (or `auto_accept` if same-account/region) |
| Can't even create the peering | **Overlapping CIDRs** (`10.0.0.0/16` both sides) | Re-IP one VPC to a non-overlapping range; you cannot peer overlaps |
| A reaches B but B can't reach A | Route added on only **one** route table | Add the return route on the **other** VPC's route table too |
| Routes on both sides, still no connectivity | SG/NACL blocking the peer's traffic | Allow the peer SG (same-region) or peer CIDR; check NACL ephemeral return ([file 04](04_sg_vs_nacl_examples.md)) |
| A can't reach C through B | Expecting transitive routing | Peering is **non-transitive** — create a direct A↔C peering, or use Transit Gateway |
| Only some subnets in a VPC can reach the peer | Those subnets use a **different route table** without the peer route | Add the peer route to every route table whose subnets need access |
| Cross-region: SG reference rejected | Peer-SG referencing unsupported cross-region | Reference the peer **CIDR** instead |

💡 Debug order for "no connectivity": (1) peering **Active**? (2) route on **both** route tables, on the **right** route tables for those subnets? (3) SG allows the peer? (4) NACL allows it both directions?

---

## 8. Production Peering — Accounts, Scale & Exit Decisions

Peering is intentionally decentralized: each VPC owner controls its half of acceptance, routes,
security, DNS options, tags, and eventual deletion. That is simple for two teams and increasingly
expensive to coordinate as the graph grows.

### 8.1 Cross-account acceptance and ownership

For account A (`111111111111`) requesting access to account B (`222222222222`), use separate
credentials and make the peer owner explicit:

```bash
# Account A / requester
PCX=$(aws --profile account-a ec2 create-vpc-peering-connection \
  --vpc-id vpc-0aaa \
  --peer-owner-id 222222222222 \
  --peer-vpc-id vpc-0bbb \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

# Account B / accepter. Confirm requester account, VPC IDs, CIDRs, and tags before acceptance.
aws --profile account-b ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id "$PCX"
```

For inter-Region peering, the requester also supplies `--peer-region`, and each owner runs
Region-scoped route/DNS operations in the relevant Region. A request expires if it remains pending
for seven days. Either owner can delete an active connection; use change control and alarms because
the status can still appear `active` during a regional data-path impairment.

Each owner then updates only the route tables and SGs it owns. Cross-account tags are not mirrored,
so both accounts should tag the connection with owner, environment, data classification, and
change record. If same-Region account B references account A's SG, include the source account owner:

```bash
aws --profile account-b ec2 authorize-security-group-ingress \
  --group-id sg-ec2b --protocol tcp --port 3306 \
  --source-group sg-ec2a --group-owner 111111111111
```

### 8.2 DNS and security-group Region constraints

| Feature | Same Region | Inter-Region |
|---------|-------------|--------------|
| Peering DNS option resolves peer public EC2 hostname to private IP | Yes, after both owners enable their side | **Yes**, after both owners enable their side |
| Reference peer SG by ID | Yes; cross-account syntax includes account ID/owner | **No**—use precise peer CIDRs or another policy mechanism |
| Route 53 private hosted zone names | Associate the querying VPC with the PHZ; peering toggle is unrelated | Associate the VPC/zone in its supported Region; use Resolver architecture for broader hybrid needs |

The peering connection must be `active`, and both VPCs need DNS support/hostnames before the DNS
option can be changed. Each owner modifies its requester/accepter option. Deleting a peer or its SG
can leave a **stale SG reference** that is not automatically removed; audit with
`describe-stale-security-groups`. See [file 06](06_vpc_peering_dns_resolution.md) for the complete
DNS flow.

### 8.3 Route and quota growth

Every directly connected peer consumes a peering connection and at least one route in every subnet
route table that must reach it. A full mesh of `N` VPCs needs `N(N-1)/2` connections, and each VPC
needs routes to `N-1` peer CIDRs—multiplied again when a VPC has separate route tables per AZ/tier or
multiple CIDR associations.

Before adding a peer, check current Service Quotas for active/pending peerings, routes per route
table, SG rules/references, and the prefixes advertised by each side. Reserve headroom for a
migration connection and temporary routes. Aggregate routes only when the aggregate cannot attract
traffic for an unconnected network; a short route table is not worth a blackhole.

Use AWS Config/Network Access Analyzer or an IaC pipeline to detect missing reverse routes,
forbidden environment paths, and stale SG references. Peering has no central route-propagation
control plane, so spreadsheet ownership becomes the limiting quota before AWS's numeric limit does.

### 8.4 Migrate away from overlapping CIDRs

If **any** IPv4 or IPv6 CIDR on the two VPCs overlaps, peering creation is rejected. Attaching a
non-overlapping secondary CIDR to one existing VPC does not help while its original overlapping
CIDR remains—and a VPC's primary IPv4 CIDR cannot be removed.

A safe renumbering pattern is:

1. Allocate a non-overlapping CIDR from IPAM and build a replacement VPC with equivalent subnets,
   endpoints, security, logging, quotas, and connectivity.
2. Deploy a second application stack; replicate databases/state with a service-specific migration
   method. EC2 ENIs and subnets cannot be moved between VPCs.
3. Connect the **new** VPC through peering or Transit Gateway, validate all paths, then shift traffic
   through DNS/load balancing in measured batches.
4. Hold the old stack read-only or otherwise reconcile writes during rollback. Remove its routes,
   delegation, and VPC only after the rollback window and flow logs show no consumers.

For a single producer service, PrivateLink can bridge overlapping consumer/provider CIDRs without
renumbering because consumers connect to endpoint ENIs rather than routing the provider network.
Transit Gateway and Cloud WAN simplify topology but do not magically make ambiguous overlapping
addresses routable; use a documented NAT design or renumber.

### 8.5 Choose the connectivity product by the relationship

| Choose | When it is the right boundary | Do not choose it for |
|--------|-------------------------------|----------------------|
| **VPC peering** | A few direct, non-transitive VPC relationships; low-latency private routing with decentralized ownership | Large meshes, transitive routing, or mandatory central inspection |
| **Transit Gateway** | Many VPCs/on-prem networks in a hub, segmented route domains, route propagation, centralized egress/inspection | A single service consumer or unresolved overlapping CIDRs |
| **Cloud WAN** | Policy-governed, multi-account and multi-Region global core with segments and centralized network operations | Two local VPCs where the global control plane adds no value |
| **PrivateLink** | Many consumers need one provider service, least-privilege one-way initiation, or overlapping CIDRs | General VPC-to-VPC connectivity or protocols the endpoint service cannot expose |

The practical threshold is operational, not a magic VPC count. Move from peering when every new VPC
requires many bilateral tickets/routes, when transitive shared services or inspection are required,
or when no team can confidently explain the full mesh and its failure blast radius.

---

## Key Exam Points

- VPC peering requires **non-overlapping CIDRs** — this is a hard, unfixable-after-the-fact constraint.
- You must do all four: **create, accept, route on BOTH sides, allow in SG/NACL.** Routing is **not** automatic.
- Peering is **non-transitive** — A↔B and B↔C does **not** give A↔C. Use **Transit Gateway** for hub-and-spoke at scale.
- Same-region peering can **reference the peer's SG by ID**; cross-region cannot — use the peer **CIDR** there.
- Peering works **cross-account and cross-region**; non-same-account/region needs **explicit acceptance**.
- Traffic stays **private on the AWS backbone** — no IGW, NAT, or public IPs involved.
- "A reaches B but not the reverse" → a **missing return route** on the other VPC's route table.
- Peering DNS options work for same- and inter-Region peering; cross-peer SG references work only
  in the same Region and require the owner ID across accounts.
- A secondary non-overlapping CIDR doesn't cure an overlapping primary CIDR. Rebuild/renumber, or
  use PrivateLink for a bounded service—not TGW/Cloud WAN as an overlap shortcut.
- Route/ownership growth, transitivity, central inspection, and global policy are the signals to
  choose Transit Gateway or Cloud WAN instead of adding another bilateral peer.

---

**Next**: [06_vpc_peering_dns_resolution.md — DNS Resolution Across a Peering Connection](06_vpc_peering_dns_resolution.md)
