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

## Key Exam Points

- A subnet is **public iff its route table routes `0.0.0.0/0` to an Internet Gateway**. Nothing else makes it public.
- A subnet lives in **exactly one AZ**; spread subnets across ≥2 AZs for HA. A VPC spans all AZs in the Region.
- **One route table can serve many subnets.** You associate by routing behavior, not one-per-subnet.
- The **local route** (`VPC CIDR → local`) exists in every route table, is implicit, and cannot be deleted or modified.
- AWS **reserves 5 IPs per subnet**; a `/24` yields 251 usable, not 254.
- Subnets with no explicit route-table association use the **main route table** — a frequent source of "why is this public?".
- Public IP + `0.0.0.0/0→igw` + SG allow + NACL allow are **all four** required for inbound internet reachability.

---

**Next**: [02_ec2_private_subnet_nat.md — EC2 in a Private Subnet Reaching the Internet via NAT](02_ec2_private_subnet_nat.md)
