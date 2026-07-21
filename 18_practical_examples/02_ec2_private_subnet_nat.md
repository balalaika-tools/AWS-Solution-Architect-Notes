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

> **Rule for a zonal public NAT**: The NAT Gateway lives in a **public** subnet, not the private
> one. It needs the IGW route to reach the internet. The private subnet's route table then points
> `0.0.0.0/0` at the **NAT Gateway**.

This rule describes the **zonal public NAT Gateway used in this walkthrough**. AWS also offers a
Regional NAT Gateway for public internet egress; it uses one regional NAT ID, doesn't require a
public subnet, and can expand into workload AZs automatically. The production comparison in §8
shows when that newer mode changes the design.

A NAT Gateway requires:
- A **public subnet** to live in (so its own traffic exits via the IGW).
- An **Elastic IP** attached (its stable public-facing address).

❌ Putting the NAT Gateway in the private subnet is the classic mistake. It then has no path to the internet itself, and everything blackholes.

For a resilient **zonal** design you deploy one NAT GW per AZ and point each AZ's private route
table at the NAT GW in the *same* AZ—otherwise an AZ outage that takes down the single NAT GW cuts
egress for every AZ, and cross-AZ NAT traffic incurs data-transfer charges.

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

## 8. Production Egress Architecture

The basic route proves NAT works. Production design decides whether egress should be distributed,
regional, centralized, replaced by endpoints, or removed with IPv6-only controls.

### 8.1 Zonal failure and cost model

| Design | Zonal failure behavior | Cost and operations |
|--------|------------------------|---------------------|
| One zonal NAT shared by all AZs | If its AZ is impaired, every subnet routed to that NAT ID loses new IPv4 internet connections | One hourly NAT charge, but cross-AZ data transfer plus NAT processing and a broad failure domain |
| One zonal NAT per active AZ | Only the affected AZ's egress path is lost; surviving app AZs keep their own NAT | One NAT-hour charge per AZ; avoids normal cross-AZ NAT traffic and aligns failure domains |
| Regional NAT, automatic mode | One route target expands/contracts across workload AZs and maintains zonal affinity | Simpler routing and automatic multi-AZ coverage; model current regional-NAT hourly, public IPv4, and processing prices |

A zonal NAT Gateway is redundant **inside its AZ**, not across AZs. VPC route tables don't perform
health checks and do not rewrite `nat-a` to `nat-b` when AZ-a fails. A runbook can replace the
route target with a standby NAT, but that is a control-plane recovery, takes time, and existing
flows are lost. Per-AZ routing avoids making that update part of another zone's recovery.

Regional NAT removes the per-AZ route-ID problem for public egress, but automatic expansion into a
newly used AZ can take up to 60 minutes. Establish and test the intended AZ footprint before a
launch or evacuation. Use zonal NAT for private NAT use cases; Regional NAT currently targets public
connectivity.

Calculate monthly egress as:

```
NAT hours + NAT processed GiB + public IPv4/EIP cost
+ cross-AZ transfer (if workload and zonal NAT differ)
+ Transit Gateway/firewall processing (if centralized)
+ interface-endpoint hours and GiB
```

Use Cost and Usage Reports by NAT gateway/endpoint and CloudWatch NAT metrics to validate the
estimate. A cheaper one-NAT diagram can cost more than per-AZ NAT when data crosses zones, while
many low-volume interface endpoints can cost more than the NAT traffic they replace.

### 8.2 Substitute VPC endpoints before buying NAT capacity

- Add **gateway endpoints** for S3 and DynamoDB. They add service-prefix routes to selected route
  tables, require no NAT/IGW, and have no endpoint hourly or data-processing charge.
- Add **interface endpoints** for high-volume or security-sensitive regional services such as ECR,
  Systems Manager, CloudWatch Logs, KMS, Secrets Manager, and STS. Put endpoint ENIs in the AZs that
  use them, enable private DNS, restrict their SGs, and apply endpoint policies.
- Keep NAT only for public package repositories, third-party APIs, and AWS services without an
  appropriate private endpoint. Endpoints don't provide arbitrary internet access.

Test DNS after enabling private DNS: the normal regional service hostname should resolve to
endpoint private IPs. Also test endpoint policy and IAM failures; private routing does not grant
service authorization.

### 8.3 Centralized egress with Transit Gateway and inspection

For many VPCs, a network account can own an egress/inspection VPC:

```
spoke private RT 0.0.0.0/0 → TGW
TGW spoke route table       → egress VPC attachment
inspection subnet RT        → Network Firewall/GWLB appliance
egress subnet RT            → zonal NAT → IGW
return route                → TGW → originating spoke
```

Separate TGW route tables prevent production, non-production, and shared services from becoming
implicitly reachable. With stateful virtual appliances or Gateway Load Balancer, enable the
appropriate TGW **appliance mode** and design both directions through the same inspection AZ so the
firewall sees the complete flow. Network Firewall route insertion has its own documented pattern;
don't improvise an asymmetric return route.

Centralization gives one policy/logging point and can reduce duplicated NATs, but adds TGW
attachment/data charges, inspection processing, longer paths, and a larger blast radius. Deploy
inspection and NAT paths across AZs, capacity-test the surviving paths, and remember that VPC
peering cannot be used as a transit path to another VPC's NAT Gateway.

### 8.4 IPv6 outbound-only design

For dual-stack private app subnets, add an **egress-only internet gateway** and route
`::/0 → eigw-...`. It is stateful: instances can initiate IPv6 internet connections, but the
gateway rejects connections initiated from the internet. IPv6 doesn't use IPv4 NAT; security
groups and NACLs must still allow the intended IPv6 prefixes and return traffic.

Keep `0.0.0.0/0 → NAT` only for IPv4 destinations. An IPv6-only workload that must call an
IPv4-only service can use DNS64 with NAT64 on a NAT Gateway. Verify dependencies first—hard-coded
IPv4 literals, private DNS, agents, and package repositories often prevent an immediate IPv6-only
cutover.

### 8.5 Failure test

From a canary in each AZ, record DNS, TCP/TLS, and HTTP success through endpoints, NAT, and the
approved IPv6 path. Then remove or blackhole one zonal NAT route and verify:

- app-a fails its new IPv4 internet flows as designed;
- app-b/app-c continue through their local paths;
- S3/DynamoDB gateway-endpoint traffic remains available;
- alarms fire on NAT errors/connection attempts and canary failure;
- the documented route replacement restores new connections within the measured RTO.

---

## Key Exam Points

- **NAT Gateway = outbound-only internet for private subnets.** It cannot accept inbound-initiated connections.
- The **zonal public NAT** in this build lives in a public subnet and needs an Elastic IP. Its
  private route table points `0.0.0.0/0` at that NAT ID.
- For a resilient **zonal NAT** design, deploy **one NAT GW per AZ**; a single zonal NAT is a per-AZ single point of failure and adds cross-AZ data charges. A Regional NAT Gateway is the current alternative when its expansion behavior and service limits fit the workload.
- **NAT Gateway is managed and auto-scaling**; a NAT Instance is a self-managed EC2 needing **source/dest check disabled**. Prefer NAT Gateway.
- The private instance needs **no public IP** — that's what keeps it private. NAT provides egress without exposure.
- For **IPv6** outbound-only, the equivalent is an **egress-only Internet Gateway**, not a NAT GW.
- Zonal NAT routes don't fail over automatically. Use same-AZ zonal NATs, or evaluate the current
  Regional NAT option for automatically distributed public egress.
- Gateway endpoints can remove S3/DynamoDB traffic from NAT; centralized TGW egress adds inspection
  consistency but also cost, route complexity, and a shared blast radius.

---

**Next**: [03_alb_to_private_ec2.md — Public ALB Routing to Private EC2](03_alb_to_private_ec2.md)
