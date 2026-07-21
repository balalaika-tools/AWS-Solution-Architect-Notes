# Reference Build: A VPC with Public & Private Subnets Across 2 AZs

> **Who this is for**: Engineers who have read [VPC, Subnets & Route Tables](../03_networking/02_vpc_subnets_route_tables.md)
> and want to see the canonical multi-AZ VPC actually assembled — CIDR plan, the two route
> tables, the Internet Gateway, and how "public" and "private" come down to a single route.
> This topology is the base for files 02–05, so build a clear mental model of it here.

---

## 1. What We're Building

The single most common VPC layout, and the one the exam quietly assumes in almost every networking question:

- One VPC: `10.0.0.0/16`.
- **Two Availability Zones** (for high availability — a subnet lives in exactly one AZ).
- In each AZ: one **public** subnet and one **private** subnet.
- One **Internet Gateway (IGW)** attached to the VPC.
- Two route tables: a **public** one (default route to the IGW) and a **private** one (no internet route — yet; NAT comes in [file 02](02_ec2_private_subnet_nat.md)).

> **Key insight**: A subnet is not "public" because of a checkbox. It is public **only because its route table sends `0.0.0.0/0` to an Internet Gateway**. Same subnet, different route table → private. Internalize this; it is the most tested idea in the section.

---

## 2. CIDR Plan

Carve the `/16` into `/24`s. A `/24` gives 256 addresses (251 usable — AWS reserves 5 per subnet). Leaving gaps between ranges keeps room to grow.

| Subnet | CIDR | AZ | Type | Usable IPs | Purpose |
|--------|------|-----|------|-----------|---------|
| `public-az-a`  | `10.0.0.0/24`  | `us-east-1a` | Public  | 251 | ALB nodes, NAT GW, bastion |
| `public-az-b`  | `10.0.1.0/24`  | `us-east-1b` | Public  | 251 | ALB nodes, NAT GW (2nd AZ) |
| `private-az-a` | `10.0.10.0/24` | `us-east-1a` | Private | 251 | App / EC2 / RDS |
| `private-az-b` | `10.0.11.0/24` | `us-east-1b` | Private | 251 | App / EC2 / RDS |

⚠️ **AWS reserves 5 IPs in every subnet**: network address (`.0`), VPC router (`.1`), DNS (`.2`), future use (`.3`), and broadcast (`.255`). So `10.0.0.0/24` gives you `.4`–`.254` to use, not `.1`–`.254`.

💡 The `.10`/`.11` numbering for private subnets (vs `.0`/`.1` for public) is just a readable convention — public subnets low, private subnets higher. It has no technical meaning.

---

## 3. ASCII Topology

```
                          Internet
                              │
                        ┌─────┴─────┐
                        │    IGW    │   (attached to VPC)
                        └─────┬─────┘
                              │
┌─────────────────────────── VPC 10.0.0.0/16 ───────────────────────────┐
│                             │                                          │
│   ── AZ us-east-1a ──       │        ── AZ us-east-1b ──               │
│  ┌──────────────────┐       │       ┌──────────────────┐              │
│  │ public-az-a       │      │       │ public-az-b       │              │
│  │ 10.0.0.0/24       │◄─────┴──────►│ 10.0.1.0/24       │              │
│  │ RT: PUBLIC        │              │ RT: PUBLIC        │              │
│  └──────────────────┘              └──────────────────┘              │
│                                                                        │
│  ┌──────────────────┐              ┌──────────────────┐              │
│  │ private-az-a      │              │ private-az-b      │              │
│  │ 10.0.10.0/24      │              │ 10.0.11.0/24      │              │
│  │ RT: PRIVATE       │              │ RT: PRIVATE       │              │
│  └──────────────────┘              └──────────────────┘              │
│                                                                        │
│   local route 10.0.0.0/16 → local  (implicit, every RT, can't delete) │
└────────────────────────────────────────────────────────────────────────┘
```

Both public subnets share **one** public route table; both private subnets share **one** private route table. You don't need a route table per subnet — you need one per *routing behavior*.

---

## 4. Route Tables

Every route table starts with the **implicit `local` route** for the VPC CIDR — this is what lets any resource talk to any other resource inside the VPC. You cannot remove it.

**Public route table** (associated with `public-az-a`, `public-az-b`):

| Destination | Target | Meaning |
|-------------|--------|---------|
| `10.0.0.0/16` | `local` | Intra-VPC traffic (implicit) |
| `0.0.0.0/0`   | `igw-0abc...` | Everything else → Internet Gateway |

**Private route table** (associated with `private-az-a`, `private-az-b`):

| Destination | Target | Meaning |
|-------------|--------|---------|
| `10.0.0.0/16` | `local` | Intra-VPC traffic (implicit) |
| *(no `0.0.0.0/0` route)* | — | No internet path → subnet is private |

> **Rule**: The presence of a `0.0.0.0/0 → igw` route in a subnet's route table is the *definition* of a public subnet. Add NAT instead of IGW for that route and the subnet becomes private-with-egress ([file 02](02_ec2_private_subnet_nat.md)).

---

## 5. Build It (Terraform-style HCL)

```hcl
# --- VPC ---
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true   # lets instances resolve via the .2 resolver
  enable_dns_hostnames = true   # gives instances internal DNS names
  tags = { Name = "main-vpc" }
}

# --- Internet Gateway ---
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "main-igw" }
}

# --- Public subnets (map_public_ip_on_launch so instances get a public IP) ---
resource "aws_subnet" "public_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.0.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
  tags                    = { Name = "public-az-a" }
}
resource "aws_subnet" "public_b" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1b"
  map_public_ip_on_launch = true
  tags                    = { Name = "public-az-b" }
}

# --- Private subnets (no public IP) ---
resource "aws_subnet" "private_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.10.0/24"
  availability_zone = "us-east-1a"
  tags              = { Name = "private-az-a" }
}
resource "aws_subnet" "private_b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.11.0/24"
  availability_zone = "us-east-1b"
  tags              = { Name = "private-az-b" }
}

# --- Public route table: default route to the IGW ---
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id   # <-- this line makes it "public"
  }
  tags = { Name = "public-rt" }
}
resource "aws_route_table_association" "public_a" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.public.id
}
resource "aws_route_table_association" "public_b" {
  subnet_id      = aws_subnet.public_b.id
  route_table_id = aws_route_table.public.id
}

# --- Private route table: only the implicit local route (no internet path yet) ---
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "private-rt" }
}
resource "aws_route_table_association" "private_a" {
  subnet_id      = aws_subnet.private_a.id
  route_table_id = aws_route_table.private.id
}
resource "aws_route_table_association" "private_b" {
  subnet_id      = aws_subnet.private_b.id
  route_table_id = aws_route_table.private.id
}
```

Equivalent CLI for the route that defines "public":

```bash
aws ec2 create-route \
  --route-table-id rtb-0public \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0abc123
```

---

## 6. Troubleshooting

**An instance in a "public" subnet can't be reached from the internet.**

Walk the checklist in order — all four must be true:
- ✅ The subnet's route table has `0.0.0.0/0 → igw`. (Most common miss.)
- ✅ The instance has a **public IP** (or Elastic IP). A public-subnet instance with only a private IP is unreachable from outside.
- ✅ The **security group** allows the inbound port (e.g. 22 or 443).
- ✅ The **NACL** allows both the inbound port *and* the ephemeral return ports ([file 04](04_sg_vs_nacl_examples.md)).

**A subnet I meant to be private has internet access.**

⚠️ It's probably associated with the public route table, or with the **main route table** that happens to carry an IGW route. Subnets with no explicit association fall back to the main route table — check what that one contains.

**I forgot to attach the IGW to the VPC.**

The route `0.0.0.0/0 → igw-xxxx` is accepted but traffic blackholes. The IGW must be both *created* and *attached* to the VPC.

**`enable_dns_hostnames` is off and nothing resolves names.**

Internal DNS and many AWS integrations (private endpoints, peering DNS in [file 06](06_vpc_peering_dns_resolution.md)) need both `enableDnsSupport` and `enableDnsHostnames` set to true.

---

## 7. Production Extension — Three Tiers Across Three AZs

The two-AZ build explains the mechanics. A production VPC normally reserves more failure and
growth headroom: three AZs, a separate subnet for each trust tier in each AZ, and routing that
keeps a failed egress zone from becoming every zone's dependency.

```
                         internet-facing ALB
                    edge-a     edge-b     edge-c
                       │          │          │
                    app-a      app-b      app-c       ASG/ECS/EKS compute
                       │          │          │
                   data-a     data-b     data-c       RDS/cache; no default route
```

| Tier | Subnets | Route behavior | Normal occupants |
|------|---------|----------------|------------------|
| **Edge** | One per AZ | `0.0.0.0/0 → IGW`; IPv6 `::/0 → IGW` only where public IPv6 is intended | ALB nodes and zonal public NAT gateways; avoid general-purpose instances |
| **Application** | One per AZ | IPv4 default to the same-AZ NAT; AWS-service prefixes to VPC endpoints | Auto Scaling, ECS/EKS, Lambda ENIs |
| **Data** | One per AZ | Local and explicitly required private routes only; normally no internet default | RDS/Aurora subnet groups, caches, internal data services |

Keep a route table per **AZ and routing behavior**. Three app route tables let `app-a` use
`nat-a`, `app-b` use `nat-b`, and `app-c` use `nat-c`; sharing one private route table would
reintroduce a single zonal egress dependency. Data route tables stay separate so a future app
egress change cannot accidentally give databases an internet path.

### 7.1 Derive CIDRs from IPAM

Use VPC IP Address Manager (IPAM) as the allocation authority rather than copying CIDRs into
spreadsheets. A network account can share environment/Region pools through AWS RAM. The workload
account asks for a prefix length; IPAM selects a non-overlapping allocation and records its owner.

```hcl
# The parent pool is governed and RAM-shared by the network account.
resource "aws_vpc" "prod" {
  ipv4_ipam_pool_id    = var.prod_us_east_1_pool_id
  ipv4_netmask_length  = 20
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "orders-prod" }
}

# A resource-planning pool carved from the VPC CIDR can allocate subnet CIDRs.
# Repeat for edge/app/data and for three distinct AZs.
resource "aws_subnet" "app" {
  for_each            = toset(["us-east-1a", "us-east-1b", "us-east-1c"])
  vpc_id              = aws_vpc.prod.id
  availability_zone   = each.value
  ipv4_ipam_pool_id   = var.orders_subnet_pool_id
  ipv4_netmask_length = 24
  tags = { Name = "app-${each.value}"; Tier = "application" }
}
```

Choose the VPC prefix only after estimating subnets, expansion Regions, acquisitions, and hybrid
routes. A VPC with an overlapping primary CIDR cannot later be made peerable merely by attaching a
non-overlapping secondary CIDR. Reserve unused blocks in IPAM, and alarm on IPAM's
`SubnetIPUsage` before an ALB, interface endpoint, Lambda, or container rollout consumes the last
addresses.

### 7.2 Remove NAT traffic where private endpoints fit

Add gateway endpoints for **S3 and DynamoDB** to the app route tables; they need neither NAT nor an
internet gateway and have no hourly endpoint charge. Add interface endpoints in multiple AZs for
services actually used privately—commonly Systems Manager, ECR API/Docker, CloudWatch Logs, KMS,
Secrets Manager, and STS. Enable private DNS deliberately, restrict endpoint security groups to the
workload SGs, and narrow endpoint policies as well as IAM policies.

Endpoints do not replace arbitrary internet egress, and interface endpoints have hourly and
data-processing cost per AZ. Compare that cost with NAT processing volume instead of deploying
every possible endpoint by habit. Data subnets should receive only the endpoints and routes their
database operations genuinely require.

### 7.3 Make the network observable and testable

Enable VPC Flow Logs at VPC scope and deliver them to a protected central S3 bucket, CloudWatch
Logs, or Firehose. Include account, VPC/subnet/ENI, AZ, source/destination, ports, action, flow-log
status, and traffic-path fields. Flow logs show metadata rather than packet contents; `REJECT`
proves a network control rejected traffic but is not, by itself, proof that the SG rather than the
NACL did it.

Treat network IaC like application code:

1. Run formatter, syntax validation, a reviewed plan, and policy checks that reject public IPs in
   app/data subnets, unrestricted administrative ports, or a database default route.
2. Confirm every subnet has an explicit route-table association and that app defaults point to the
   same-AZ egress path.
3. Use Reachability Analyzer for expected and forbidden paths—for example internet → app ENI must
   be unreachable, while ALB → app health port must be reachable.
4. Use Network Access Analyzer to detect broad paths such as IGW → any ENI not tagged `Tier=edge`.
5. Run canaries from each app AZ to required endpoints, DNS, approved internet destinations, and
   the data tier. Then remove one zonal egress path and verify only that AZ's new connections fail
   or use the documented recovery path.

### 7.4 Budget quotas and headroom before launch

Record the current Service Quotas values and peak model for VPCs, subnets, route-table entries,
security-group rules/references, Elastic IPs, NAT gateways, interface/gateway endpoints, ENIs, and
IPAM allocations. Leave space for rolling deployments and failure: an N+1 ASG replacement,
additional ALB nodes, endpoint ENIs, and the surviving AZs absorbing a zonal evacuation. A subnet
that is 80% full during normal load has little production headroom even if its resource quota has
not been reached.

---

## Key Exam Points

- A subnet is **public iff its route table routes `0.0.0.0/0` to an Internet Gateway**. Nothing else makes it public.
- A subnet lives in **exactly one AZ**; spread subnets across ≥2 AZs for HA. A VPC spans all AZs in the Region.
- **One route table can serve many subnets.** You associate by routing behavior, not one-per-subnet.
- The **local route** (`VPC CIDR → local`) exists in every route table, is implicit, and cannot be deleted or modified.
- AWS **reserves 5 IPs per subnet**; a `/24` yields 251 usable, not 254.
- Subnets with no explicit route-table association use the **main route table** — a frequent source of "why is this public?".
- Public IP + `0.0.0.0/0→igw` + SG allow + NACL allow are **all four** required for inbound internet reachability.
- A production three-tier VPC uses separate edge, app, and data subnets in at least three AZs,
  IPAM-governed non-overlapping CIDRs, same-AZ egress, endpoints, flow logs, and tested quota/IP
  headroom.

---

**Next**: [02_ec2_private_subnet_nat.md — EC2 in a Private Subnet Reaching the Internet via NAT](02_ec2_private_subnet_nat.md)
