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
| [03_gateways_igw_nat.md](03_gateways_igw_nat.md) | Internet access | Internet Gateway, what makes a subnet public, Elastic IPs, NAT Gateway vs NAT Instance, egress-only IGW for IPv6, outbound-only private subnet access, bastion hosts & SSM Session Manager. |
| [04_security_groups_vs_nacls.md](04_security_groups_vs_nacls.md) | Firewalls | Security Groups (stateful, instance-level) vs Network ACLs (stateless, subnet-level), ephemeral ports, return traffic, exam traps. |
| [05_vpc_peering_and_endpoints.md](05_vpc_peering_and_endpoints.md) | Private connectivity | VPC Peering (non-transitive, no overlapping CIDR), Gateway vs Interface endpoints, PrivateLink. |
| [06_transit_gateway_and_advanced.md](06_transit_gateway_and_advanced.md) | Scale & advanced | Transit Gateway hub-and-spoke, VPC Flow Logs (+ Athena), Traffic Mirroring, AWS Network Firewall, IPv6, default VPC, BYOIP, longest-prefix routing. |
| [07_enis_security_groups_and_service_networking.md](07_enis_security_groups_and_service_networking.md) | ENIs & service networking | Which resources create ENIs, which need security groups, and how endpoint types differ. |
| [08_dhcp_prefix_lists_sharing_analyzer.md](08_dhcp_prefix_lists_sharing_analyzer.md) | The remaining blind spots | VPC DNS attributes, DHCP option sets, managed prefix lists, VPC sharing (AWS RAM), Reachability Analyzer & Network Access Analyzer. |

---

## SAP-C02 Network Connectivity Objective Map

SAP-C02 Domain 1, Task 1.1 asks you to **architect network connectivity
strategies**. It tests design choices across accounts, VPCs, Regions, and
on-premises networks—not only whether you remember a VPC feature.

| Exam knowledge or skill | Primary notes | What you should be able to decide |
|-------------------------|---------------|-----------------------------------|
| Network segmentation and IP addressing | [VPCs, subnets & IPAM](02_vpc_subnets_route_tables.md), [SGs/NACLs](04_security_groups_vs_nacls.md), [VPC sharing](08_dhcp_prefix_lists_sharing_analyzer.md) | Allocate non-overlapping multi-account CIDRs; choose account/VPC/subnet/SG boundaries; enforce policy centrally. |
| Connectivity among multiple VPCs | [Peering, endpoints & PrivateLink](05_vpc_peering_and_endpoints.md), [Transit Gateway](06_transit_gateway_and_advanced.md), [VPC sharing](08_dhcp_prefix_lists_sharing_analyzer.md) | Choose peering, TGW, PrivateLink, or shared VPC from reachability, transitivity, trust, CIDR, scale, and cost requirements. |
| On-premises, colocation, and cloud integration | [Transit Gateway](06_transit_gateway_and_advanced.md), [Hybrid Networking](../14_hybrid_migration_dr/01_hybrid_networking.md) | Combine DX/VPN with TGW, design route propagation and backup, and expose only the required private services. |
| Hybrid DNS | [ENIs & Resolver endpoints](07_enis_security_groups_and_service_networking.md), [DHCP and VPC DNS](08_dhcp_prefix_lists_sharing_analyzer.md), [Route 53 Resolver](../08_dns_edge/05_route53_resolver.md) | Choose inbound/outbound Resolver endpoints, forwarding rules, and private hosted-zone ownership without breaking AWS private DNS. |
| Region/AZ selection and latency | [Subnets](02_vpc_subnets_route_tables.md), [zonal/regional egress](03_gateways_igw_nat.md), [cross-Region PrivateLink](05_vpc_peering_and_endpoints.md), [TGW peering](06_transit_gateway_and_advanced.md) | Keep normal traffic zonal where practical, remove single-AZ dependencies, and distinguish AZ resilience from Region failover. |
| Service endpoints and integrations | [VPC endpoints](05_vpc_peering_and_endpoints.md), [endpoint decision table](07_enis_security_groups_and_service_networking.md) | Select gateway vs interface vs GWLB endpoint; apply DNS, SG, endpoint-policy, hybrid, and cost controls. |
| Network traffic monitoring and troubleshooting | [Flow Logs, Traffic Mirroring & inspection](06_transit_gateway_and_advanced.md), [Reachability and Network Access Analyzer](08_dhcp_prefix_lists_sharing_analyzer.md) | Choose metadata vs packet capture vs static path analysis, then locate a missing route, propagation, SG/NACL, or asymmetric inspection path. |

---

## Reading Order

1. **Networking Primer** — IP, CIDR, subnets, routing, ports, NAT, firewalls. Do not skip this if networking is new to you.
2. **VPC, Subnets & Route Tables** — how AWS turns those concepts into a virtual network.
3. **Gateways: IGW & NAT** — how traffic enters and leaves the VPC.
4. **Security Groups vs NACLs** — the two firewall layers and why they behave differently.
5. **VPC Peering & Endpoints** — connecting VPCs and reaching AWS services privately.
6. **Transit Gateway & Advanced** — scaling connectivity beyond a handful of VPCs.
7. **ENIs, Security Groups & Service Networking** — the cross-service map of what actually gets an ENI/SG.
8. **DHCP, Prefix Lists, VPC Sharing & Reachability Analyzer** — the smaller VPC features the exam still touches, plus the modern connectivity-troubleshooting tools.

---

## Advanced Reading Path

If the fundamentals are already comfortable, follow this path for SAP-C02-style
architecture scenarios:

1. **Start with the address plan** — read the organization-scale IPAM section in
   [file 02](02_vpc_subnets_route_tables.md). Decide regional pools and growth
   reservations before choosing connectivity.
2. **Choose the multi-VPC boundary** — compare peering and PrivateLink in
   [file 05](05_vpc_peering_and_endpoints.md), professional TGW routing and
   segmentation in [file 06](06_transit_gateway_and_advanced.md), and shared-VPC
   ownership in [file 08](08_dhcp_prefix_lists_sharing_analyzer.md).
3. **Add hybrid connectivity** — continue into
   [VPN and Direct Connect](../14_hybrid_migration_dr/01_hybrid_networking.md),
   then return to the TGW route-table and Direct Connect gateway troubleshooting
   model in file 06.
4. **Design centralized egress and inspection** — use the distributed-versus-
   centralized NAT trade-off in [file 03](03_gateways_igw_nat.md), the organization
   policy controls in [file 04](04_security_groups_vs_nacls.md), and appliance-mode
   routing in file 06.
5. **Make DNS hybrid too** — combine Resolver endpoint ENIs in
   [file 07](07_enis_security_groups_and_service_networking.md), VPC DNS/DHCP in
   file 08, and the full
   [Route 53 Resolver note](../08_dns_edge/05_route53_resolver.md).
6. **Prove and observe the result** — use Flow Logs for real traffic history,
   Traffic Mirroring for packets, Reachability Analyzer for one intended path,
   and Network Access Analyzer for unintended paths at scale.

For each scenario, draw the forward **and return** route, name the owning account
for every shared component, mark its AZ/Region failure domain, and identify where
processing and data-transfer charges occur. That turns a feature list into the
trade-off reasoning the professional exam expects.

---

## Prerequisites

- [Foundations](../01_foundations/README.md) — Regions and Availability Zones (subnets live inside AZs)
- No prior networking knowledge required — file `01` builds it from scratch.

---

## Related sections

- [DNS & Edge](../08_dns_edge/README.md) — DNS resolution, Route 53, CloudFront sit on top of this network.
- [Hybrid, Migration & DR](../14_hybrid_migration_dr/README.md) — VPN and Direct Connect extend the VPC to on-premises.
- [Practical Examples](../18_practical_examples/README.md) — hands-on builds of the topologies described here.
