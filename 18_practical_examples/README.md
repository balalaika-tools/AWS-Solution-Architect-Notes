# Practical Examples

> Hands-on, end-to-end builds of the architectures the exam asks about. Each file is a concrete walkthrough — CIDR plans, route tables, security-group rules, ASCII topology diagrams, CLI / Terraform-style config, and a Troubleshooting section. This is where the concepts from the earlier sections turn into something you could actually deploy.

[![AWS](https://img.shields.io/badge/AWS-Solutions%20Architect-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)
[![Exam](https://img.shields.io/badge/Exam-SAA--C03-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)

---

These notes assume you have already read the conceptual sections — especially [Networking](../03_networking/README.md). The earlier sections explain *what* a NAT Gateway or an ALB *is*; this section shows you *how the pieces wire together* in a real topology, and where engineers get it wrong. The configuration snippets use real values (CIDRs, ports, SG references) so the relationships are unambiguous, not placeholder `foo`/`bar`.

If a build references a service you haven't met yet, follow the inline link back to the concept file, then return.

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_vpc_public_private_subnets.md](01_vpc_public_private_subnets.md) | Reference VPC | A `10.0.0.0/16` VPC with public + private subnets across 2 AZs, an Internet Gateway, and the route tables that define "public" vs "private". The base topology everything else builds on. |
| [02_ec2_private_subnet_nat.md](02_ec2_private_subnet_nat.md) | Outbound via NAT | An EC2 instance in a private subnet reaching the internet outbound through a NAT Gateway, while inbound stays blocked. Packet-flow diagram. |
| [03_alb_to_private_ec2.md](03_alb_to_private_ec2.md) | Public ALB → private app | An internet-facing ALB in public subnets forwarding to EC2 in private subnets, with security-group chaining (instance SG trusts the ALB SG, not the internet). |
| [04_sg_vs_nacl_examples.md](04_sg_vs_nacl_examples.md) | Firewalls, worked | Side-by-side rule tables: a stateful SG (no return rule needed) vs a stateless NACL (must allow ephemeral return traffic), plus the classic "NACL denies but SG allows" interaction. |
| [05_vpc_peering.md](05_vpc_peering.md) | Connect two VPCs | Peering VPC-A (`10.0.0.0/16`) and VPC-B (`10.1.0.0/16`): request/accept, updating **both** route tables, cross-referencing peer SGs, and the non-transitive limitation. |
| [06_vpc_peering_dns_resolution.md](06_vpc_peering_dns_resolution.md) | Peering + private DNS | Enabling DNS resolution across a peering connection so private hostnames resolve to private IPs. |
| [07_route53_public_hosted_zone.md](07_route53_public_hosted_zone.md) | Public DNS | A public hosted zone serving records for a registered domain on the internet. |
| [08_route53_private_hosted_zone.md](08_route53_private_hosted_zone.md) | Private DNS | A private hosted zone resolving names only inside associated VPCs. |
| [09_route53_resolver_hybrid_dns.md](09_route53_resolver_hybrid_dns.md) | Hybrid DNS | Route 53 Resolver inbound/outbound endpoints bridging on-prem DNS and VPC DNS. |
| [10_s3_static_site_vs_cloudfront.md](10_s3_static_site_vs_cloudfront.md) | Static hosting | S3 static website hosting vs S3 + CloudFront (OAC), HTTPS, and caching. |
| [11_rds_private_access.md](11_rds_private_access.md) | Private database | RDS in private subnets, DB subnet group, SG chaining from the app tier. |
| [12_rds_multiaz_vs_read_replica.md](12_rds_multiaz_vs_read_replica.md) | HA vs read scaling | Multi-AZ (failover) vs read replicas (read scaling) — when to use which. |
| [13_ecs_fargate_behind_alb.md](13_ecs_fargate_behind_alb.md) | Containers | An ECS Fargate service in private subnets fronted by an ALB. |
| [14_lambda_in_vpc.md](14_lambda_in_vpc.md) | Lambda networking | Putting a Lambda function in a VPC to reach private resources, and the NAT requirement for internet egress. |
| [15_sqs_with_asg.md](15_sqs_with_asg.md) | Queue-driven scaling | An SQS queue driving an Auto Scaling Group of workers. |
| [16_cloudwatch_alarm_scaling.md](16_cloudwatch_alarm_scaling.md) | Metric scaling | CloudWatch alarms triggering Auto Scaling policies. |
| [17_cloudtrail_vs_cloudwatch_vs_config.md](17_cloudtrail_vs_cloudwatch_vs_config.md) | Observability split | Which service answers "who did it", "is it healthy", and "is it compliant". |
| [18_kms_encryption_examples.md](18_kms_encryption_examples.md) | Encryption | KMS keys, envelope encryption, and encrypting S3 / EBS / RDS. |
| [19_disaster_recovery_strategies.md](19_disaster_recovery_strategies.md) | DR patterns | Backup & restore, pilot light, warm standby, multi-site active/active, with RTO/RPO. |
| [20_vpc_endpoints_private_aws_services.md](20_vpc_endpoints_private_aws_services.md) | VPC endpoints | Private access to S3, ECR, Logs, Secrets, SQS and other AWS services without public IPs/NAT. |

---

## Reading Order

The first five files build on each other and should be read in sequence — each adds one layer to the same VPC:

1. **VPC with public & private subnets** — the foundation topology.
2. **EC2 in a private subnet via NAT** — outbound internet without exposure.
3. **ALB to private EC2** — accepting inbound traffic safely.
4. **SG vs NACL examples** — the firewall rules that make the above work (and fail).
5. **VPC Peering** — connecting two VPCs.
6. **Peering + DNS** onward — DNS, edge, data, compute, and operational builds. These are more independent; pick by topic once you have the networking base.

---

## Prerequisites

- [Networking](../03_networking/README.md) — VPC, subnets, route tables, IGW/NAT, SG/NACL, peering. **Essential** for files 01–06.
- [Compute](../04_compute/README.md) — EC2 basics, for files 02–03.
- [HA & Scaling](../07_ha_scaling/README.md) — load balancers and Auto Scaling, for files 03, 15, 16.
- [DNS & Edge](../08_dns_edge/README.md) — Route 53 and CloudFront, for files 06–10.

---

**Next**: [01_vpc_public_private_subnets.md — Reference VPC Build](01_vpc_public_private_subnets.md)
