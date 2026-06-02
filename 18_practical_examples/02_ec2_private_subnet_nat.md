# EC2 in a Private Subnet Reaching the Internet via NAT Gateway

> **Who this is for**: Engineers who built the [reference VPC](01_vpc_public_private_subnets.md)
> and now need a private instance to download packages, call an external API, or fetch OS
> updates — **outbound only**, with no way in from the internet. Builds directly on file 01;
> reuses the same VPC, subnets, and route tables. Concept reference:
> [Gateways: IGW & NAT](../03_networking/03_gateways_igw_nat.md).

---

## 1. The Problem

A private subnet (from file 01) has no `0.0.0.0/0` route, so its instances can't reach the internet at all. But real workloads need *outbound* access — `yum update`, `pip install`, calling a payment API — while still refusing all *inbound* connections from the internet.

The answer is a **NAT Gateway**: a managed device that performs source NAT for outbound traffic. The private instance's request goes out through the NAT GW's public IP; replies come back to the NAT GW and are forwarded to the instance. Because the connection is *initiated from inside*, the NAT GW only ever forwards return traffic for connections the instance started — nothing on the internet can initiate a connection inward.

> **Key insight**: NAT gives **outbound-only** internet access. It is one-directional by design. If you need inbound (someone on the internet connecting *to* your private instance), that's a load balancer or a public IP — not NAT. This asymmetry is the whole point and a constant exam theme.

---

## 2. NAT Gateway Placement (the part everyone gets wrong)

> **Rule**: The NAT Gateway lives in a **public** subnet, not the private one. It needs the IGW route to reach the internet. The private subnet's route table then points `0.0.0.0/0` at the **NAT Gateway**.

A NAT Gateway requires:
- A **public subnet** to live in (so its own traffic exits via the IGW).
- An **Elastic IP** attached (its stable public-facing address).

❌ Putting the NAT Gateway in the private subnet is the classic mistake. It then has no path to the internet itself, and everything blackholes.

For true HA you deploy **one NAT GW per AZ** and point each AZ's private route table at the NAT GW in the *same* AZ — otherwise an AZ outage that takes down the single NAT GW cuts egress for every AZ, and cross-AZ NAT traffic incurs data-transfer charges.

---

## 3. ASCII Topology + Packet Flow

```
                                  Internet
                                     │
                               ┌─────┴─────┐
                               │    IGW    │
                               └─────┬─────┘
                                     │  (4) NAT's EIP is the source the internet sees
┌──────────────────────── VPC 10.0.0.0/16 ─────────────────────────┐
│   ── AZ us-east-1a ──                                             │
│  ┌────────────────────────────────┐                              │
│  │ public-az-a 10.0.0.0/24         │                              │
│  │   ┌──────────────────────┐      │  RT PUBLIC: 0.0.0.0/0 → igw  │
│  │   │  NAT Gateway          │◄─────┼──(3)──                       │
│  │   │  + Elastic IP         │      │                              │
│  │   └──────────┬───────────┘      │                              │
│  └──────────────│──────────────────┘                              │
│                 │ (2) RT PRIVATE: 0.0.0.0/0 → nat-gw              │
│  ┌──────────────│──────────────────┐                              │
│  │ private-az-a │ 10.0.10.0/24      │                              │
│  │   ┌──────────┴───────────┐      │                              │
│  │   │  EC2  10.0.10.25      │      │                              │
│  │   │  (no public IP)       │──(1)─┘  outbound request           │
│  │   └──────────────────────┘      │                              │
│  └────────────────────────────────┘                              │
└────────────────────────────────────────────────────────────────────┘

  (1) EC2 sends to e.g. 52.x.x.x:443
  (2) Private RT: 0.0.0.0/0 → NAT GW, so packet goes to NAT GW
  (3) NAT rewrites source to its EIP, public RT sends it to IGW → internet
  (4) Reply returns to the EIP → NAT GW → back to 10.0.10.25
      Nothing on the internet can START a connection to 10.0.10.25.
```

---

## 4. Route Tables After Adding NAT

The **public** route table is unchanged from file 01. Only the **private** route table gains a default route — pointed at the NAT Gateway:

**Private route table** (now with egress):

| Destination | Target | Meaning |
|-------------|--------|---------|
| `10.0.0.0/16` | `local` | Intra-VPC (implicit) |
| `0.0.0.0/0`   | `nat-0a1b2c...` | Outbound internet → NAT Gateway |

That single new row is the entire change. The instance has no public IP and the IGW route never appears in its route table — yet it now has outbound internet.

---

## 5. Build It

**CLI:**

```bash
# 1. Allocate an Elastic IP for the NAT Gateway
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc \
  --query 'AllocationId' --output text)

# 2. Create the NAT Gateway IN THE PUBLIC SUBNET (subnet-public-a)
NAT_ID=$(aws ec2 create-nat-gateway \
  --subnet-id subnet-0public_a \
  --allocation-id "$EIP_ALLOC" \
  --query 'NatGateway.NatGatewayId' --output text)

# Wait until it's available (takes a minute or two)
aws ec2 wait nat-gateway-available --nat-gateway-ids "$NAT_ID"

# 3. Add the default route to the PRIVATE route table pointing at the NAT GW
aws ec2 create-route \
  --route-table-id rtb-0private \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id "$NAT_ID"
```

**Terraform-style HCL** (extends file 01):

```hcl
resource "aws_eip" "nat" {
  domain = "vpc"
  tags   = { Name = "nat-eip-az-a" }
}

resource "aws_nat_gateway" "nat_a" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_a.id   # PUBLIC subnet — required
  tags          = { Name = "nat-az-a" }
  depends_on    = [aws_internet_gateway.igw]
}

# Add the egress route to the existing private route table
resource "aws_route" "private_egress" {
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.nat_a.id
}
```

💡 **NAT Gateway vs NAT Instance**: prefer the managed **NAT Gateway** — it scales automatically, is highly available within its AZ, and needs no patching. A NAT Instance is a self-managed EC2 box you'd only choose to save cost in tiny dev setups; it also requires you to **disable the source/destination check** on the instance.

---

## 6. Why Inbound Is Still Blocked

Even with this route in place, the internet cannot initiate a connection to `10.0.10.25`:

- The instance has **no public IP** — it isn't addressable from the internet at all.
- The NAT Gateway performs **source NAT for outbound flows only**; it has no port-forwarding or inbound listener. It forwards inbound packets *only* if they match the state of a connection the instance already opened.
- There is **no `0.0.0.0/0 → igw`** route in the private route table, so even if a packet somehow arrived, there's no inbound path mapping to the instance.

✅ This is exactly the security property you want for app servers and databases: they can pull updates and call APIs, but are unreachable from the public internet.

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Private EC2 has no internet at all | No `0.0.0.0/0 → nat-gw` route in the **private** route table | Add the route to the route table that subnet is associated with |
| NAT GW created but still no egress | NAT GW placed in the **private** subnet | Recreate it in a **public** subnet (one with the IGW route) |
| Egress works from AZ-a, broken in AZ-b | Single NAT GW; AZ-b's private RT points at AZ-a's NAT, or the AZ is down | Deploy one NAT GW per AZ; point each AZ's private RT at its local NAT GW |
| Route added but traffic blackholes | Public subnet's RT is missing the IGW route, so the NAT GW itself can't egress | Ensure the NAT GW's public subnet has `0.0.0.0/0 → igw` |
| Outbound works, but you expected inbound too | NAT is outbound-only by design | Use an ALB/NLB or a public IP for inbound — NAT cannot do it |
| Unexpected data-transfer charges | Cross-AZ NAT traffic (instance in AZ-b using NAT in AZ-a) | Per-AZ NAT keeps traffic in-AZ |

⚠️ To debug, check the route table associated with the *specific subnet* the instance is in — not "the private route table" you assume it uses. An unassociated subnet silently uses the main route table.

---

## Key Exam Points

- **NAT Gateway = outbound-only internet for private subnets.** It cannot accept inbound-initiated connections.
- The NAT Gateway **lives in a public subnet** and needs an **Elastic IP**. The *private* route table points `0.0.0.0/0` at the NAT GW.
- For HA, deploy **one NAT GW per AZ**; a single NAT GW is a per-AZ single point of failure and adds cross-AZ data charges.
- **NAT Gateway is managed and auto-scaling**; a NAT Instance is a self-managed EC2 needing **source/dest check disabled**. Prefer NAT Gateway.
- The private instance needs **no public IP** — that's what keeps it private. NAT provides egress without exposure.
- For **IPv6** outbound-only, the equivalent is an **egress-only Internet Gateway**, not a NAT GW.

---

**Next**: [03_alb_to_private_ec2.md — Public ALB Routing to Private EC2](03_alb_to_private_ec2.md)
