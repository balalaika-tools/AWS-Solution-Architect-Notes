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

## Key Exam Points

- VPC peering requires **non-overlapping CIDRs** — this is a hard, unfixable-after-the-fact constraint.
- You must do all four: **create, accept, route on BOTH sides, allow in SG/NACL.** Routing is **not** automatic.
- Peering is **non-transitive** — A↔B and B↔C does **not** give A↔C. Use **Transit Gateway** for hub-and-spoke at scale.
- Same-region peering can **reference the peer's SG by ID**; cross-region cannot — use the peer **CIDR** there.
- Peering works **cross-account and cross-region**; non-same-account/region needs **explicit acceptance**.
- Traffic stays **private on the AWS backbone** — no IGW, NAT, or public IPs involved.
- "A reaches B but not the reverse" → a **missing return route** on the other VPC's route table.

---

**Next**: [06_vpc_peering_dns_resolution.md — DNS Resolution Across a Peering Connection](06_vpc_peering_dns_resolution.md)
