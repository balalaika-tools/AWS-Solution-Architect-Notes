# Networking

> The foundation for every other section. Almost every AWS service runs inside or talks through a network, so if subnets, route tables, and firewalls don't yet feel obvious, this section is where the exam is won or lost.

[![AWS](https://img.shields.io/badge/AWS-VPC-FF9900.svg?logo=amazonaws&logoColor=white)](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)

---

This section is written for an engineer with a **weak networking background**. It starts from pure first principles — IP addresses, CIDR, subnets, routing, ports, NAT, firewalls — *before* introducing any AWS service. Networking concepts reappear in compute (security groups on EC2), databases (private subnets for RDS), load balancing, DNS, and hybrid connectivity, so the time spent here pays off everywhere.

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_networking_primer.md](01_networking_primer.md) | Pure prerequisites | IPv4, CIDR notation, RFC1918 private ranges, subnets, routing & route tables, the default route, ports & protocols, NAT, stateful vs stateless firewalls. No AWS yet. |
| [02_vpc_subnets_route_tables.md](02_vpc_subnets_route_tables.md) | The virtual network | VPC CIDR sizing, AZ-scoped subnets, public vs private (defined by the route table), the implicit local route, the main route table, AWS-reserved IPs. |
| [03_gateways_igw_nat.md](03_gateways_igw_nat.md) | Internet access | Internet Gateway, what makes a subnet public, Elastic IPs, NAT Gateway vs NAT Instance, egress-only IGW for IPv6, outbound-only private subnet access. |
| [04_security_groups_vs_nacls.md](04_security_groups_vs_nacls.md) | Firewalls | Security Groups (stateful, instance-level) vs Network ACLs (stateless, subnet-level), ephemeral ports, return traffic, exam traps. |
| [05_vpc_peering_and_endpoints.md](05_vpc_peering_and_endpoints.md) | Private connectivity | VPC Peering (non-transitive, no overlapping CIDR), Gateway vs Interface endpoints, PrivateLink. |
| [06_transit_gateway_and_advanced.md](06_transit_gateway_and_advanced.md) | Scale & advanced | Transit Gateway hub-and-spoke, VPC Flow Logs, IPv6, default VPC, BYOIP, longest-prefix routing. |
| [07_enis_security_groups_and_service_networking.md](07_enis_security_groups_and_service_networking.md) | ENIs & service networking | Which resources create ENIs, which need security groups, and how endpoint types differ. |

---

## Reading Order

1. **Networking Primer** — IP, CIDR, subnets, routing, ports, NAT, firewalls. Do not skip this if networking is new to you.
2. **VPC, Subnets & Route Tables** — how AWS turns those concepts into a virtual network.
3. **Gateways: IGW & NAT** — how traffic enters and leaves the VPC.
4. **Security Groups vs NACLs** — the two firewall layers and why they behave differently.
5. **VPC Peering & Endpoints** — connecting VPCs and reaching AWS services privately.
6. **Transit Gateway & Advanced** — scaling connectivity beyond a handful of VPCs.
7. **ENIs, Security Groups & Service Networking** — the cross-service map of what actually gets an ENI/SG.

---

## Prerequisites

- [Foundations](../01_foundations/README.md) — Regions and Availability Zones (subnets live inside AZs)
- No prior networking knowledge required — file `01` builds it from scratch.

---

## Related sections

- [DNS & Edge](../08_dns_edge/README.md) — DNS resolution, Route 53, CloudFront sit on top of this network.
- [Hybrid, Migration & DR](../14_hybrid_migration_dr/README.md) — VPN and Direct Connect extend the VPC to on-premises.
- [Practical Examples](../18_practical_examples/README.md) — hands-on builds of the topologies described here.
