# Practical Examples

> Hands-on builds for both certification tracks. Examples 01–21 begin with an associate-level
> service foundation and now extend it with production failure, security, observability, cost and
> organizational concerns. Examples 22–27 start with professional, multi-account and lifecycle
> scenarios.

[![AWS](https://img.shields.io/badge/AWS-Solutions%20Architect-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)
[![Exam](https://img.shields.io/badge/Exam-SAA--C03-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)
[![Exam](https://img.shields.io/badge/Exam-SAP--C02-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-professional/)

---

These notes assume you have already read the conceptual sections — especially [Networking](../03_networking/README.md). The earlier sections explain *what* a NAT Gateway or an ALB *is*; this section shows you *how the pieces wire together* in a real topology, and where engineers get it wrong. The configuration snippets use real values (CIDRs, ports, SG references) so the relationships are unambiguous, not placeholder `foo`/`bar`.

If a build references a service you haven't met yet, follow the inline link back to the concept file, then return.

> **Track label:** In files 01–21, treat the original/base walkthrough as an **SAA-C03
> foundation**. Its Production/SAP extension raises the review depth, but a professional exercise
> is complete only after you can defend ownership, failure, quota, observability, security, cost and
> rollback decisions for the stated scenario. Files 22–27 are **SAP-C02 lifecycle scenarios** from
> the outset.

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
| [21_api_gateway_lambda_dynamodb.md](21_api_gateway_lambda_dynamodb.md) | Serverless API | API Gateway HTTP API fronting Lambda and DynamoDB, with IAM, CORS, throttling, and idempotent writes. |
| [22_control_tower_landing_zone.md](22_control_tower_landing_zone.md) | Multi-account landing zone | Control Tower/Organizations OUs, federated access, delegated administration, account vending, controls, exceptions and measurable governance. |
| [23_centralized_inspection_and_logging.md](23_centralized_inspection_and_logging.md) | Central inspection and evidence | Segmented Transit Gateway routing, stateful inspection symmetry, hybrid/egress paths, organization logs and tested degraded modes. |
| [24_iac_cicd_rollback.md](24_iac_cicd_rollback.md) | IaC + CI/CD rollback | Immutable artifacts, cross-account deployment roles, change/policy gates, canary waves, drift and data-aware rollback. |
| [25_migration_wave_cutover.md](25_migration_wave_cutover.md) | Migration wave | Portfolio/7-R assessment, MGN/DMS/DataSync rehearsal, a timestamped cutover, data-divergence rollback and stabilization. |
| [26_config_automated_remediation.md](26_config_automated_remediation.md) | Safe Config remediation | Organization aggregation, conformance packs and a bounded Systems Manager remediation with canary rollout and evidence. |
| [27_tested_multi_region_application.md](27_tested_multi_region_application.md) | Tested multi-Region application | Regional API/Lambda, DynamoDB/S3 state, single-writer control, executable failover/failback and measured RTO/RPO. |

---

## SAP-C02 Scenario Index

The professional exam asks for the best **multi-step** design under competing requirements. Do not
stop after reproducing a base build: state the business SLO/RTO/RPO, account owners, quotas,
failure behavior, evidence, cost and rollback. The production extensions in examples 01–21 are the
bridge from their associate foundations to that review depth.

| SAP-C02 domain | Weight | Primary builds | What to defend |
|----------------|--------|----------------|----------------|
| **1. Design Solutions for Organizational Complexity** | **26%** | [Landing zone](22_control_tower_landing_zone.md), [central inspection/logging](23_centralized_inspection_and_logging.md), [organization observability](17_cloudtrail_vs_cloudwatch_vs_config.md), [cross-account KMS](18_kms_encryption_examples.md), [central endpoints](20_vpc_endpoints_private_aws_services.md), [hybrid DNS](09_route53_resolver_hybrid_dns.md) | Account/OU boundaries, delegated administrators, least-privilege cross-account paths, central-versus-distributed blast radius and evidence ownership |
| **2. Design for New Solutions** | **29%** | [Production ALB](03_alb_to_private_ec2.md), [RDS access/HA](11_rds_private_access.md), [ECS service](13_ecs_fargate_behind_alb.md), [SQS workers](15_sqs_with_asg.md), [serverless API](21_api_gateway_lambda_dynamodb.md), [multi-Region application](27_tested_multi_region_application.md) | Requirements to services, failure isolation, data consistency, scaling/back-pressure, security, observability, quotas and cost |
| **3. Continuous Improvement for Existing Solutions** | **25%** | [IaC delivery](24_iac_cicd_rollback.md), [Config remediation](26_config_automated_remediation.md), [CloudWatch scaling](16_cloudwatch_alarm_scaling.md), [CloudFront delivery](10_s3_static_site_vs_cloudfront.md), [NAT/egress optimization](02_ec2_private_subnet_nat.md) | Measured SLO/KPI evidence, safe deployment, automated bounded repair, capacity/cost investigation and improvement validation |
| **4. Accelerate Workload Migration and Modernization** | **20%** | [Migration-wave cutover](25_migration_wave_cutover.md), [landing-zone prerequisites](22_control_tower_landing_zone.md), [hybrid DNS](09_route53_resolver_hybrid_dns.md), [RDS migration/failover choices](12_rds_multiaz_vs_read_replica.md), [DR runbooks](19_disaster_recovery_strategies.md) | Discovery, seven-R decision, dependency waves, target platform, cutover/data validation, rollback, recovery and post-migration optimization |

### Professional review drill

For any row, introduce one conflicting constraint—such as a 15-minute RTO with strict cost limits,
overlapping CIDRs during an acquisition, a Region-deny policy, or a forward-only schema migration.
Write the selected architecture and reject two plausible alternatives on concrete reliability,
security, performance, cost, operations or migration-risk grounds.

---

## Reading Order

The first five files build on each other and should be read in sequence — each adds one layer to the same VPC:

1. **VPC with public & private subnets** — the foundation topology.
2. **EC2 in a private subnet via NAT** — outbound internet without exposure.
3. **ALB to private EC2** — accepting inbound traffic safely.
4. **SG vs NACL examples** — the firewall rules that make the above work (and fail).
5. **VPC Peering** — connecting two VPCs.
6. **Peering + DNS** onward — DNS, edge, data, compute, and operational builds. These are more independent; pick by topic once you have the networking base.

For SAP-C02, complete one scenario from each domain in the index, then combine
[landing zone](22_control_tower_landing_zone.md) → [central connectivity/evidence](23_centralized_inspection_and_logging.md)
→ [safe delivery](24_iac_cicd_rollback.md) → [migration cutover](25_migration_wave_cutover.md)
→ [automated improvement](26_config_automated_remediation.md) → [tested recovery](27_tested_multi_region_application.md)
as one enterprise lifecycle.

---

## Prerequisites

- [Networking](../03_networking/README.md) — VPC, subnets, route tables, IGW/NAT, SG/NACL, peering. **Essential** for files 01–06.
- [Compute](../04_compute/README.md) — EC2 basics, for files 02–03.
- [HA & Scaling](../07_ha_scaling/README.md) — load balancers and Auto Scaling, for files 03, 15, 16.
- [DNS & Edge](../08_dns_edge/README.md) — Route 53 and CloudFront, for files 06–10.
- [Serverless](../10_serverless/README.md) — Lambda and API Gateway, for files 14 and 21.
- [Databases](../06_databases/README.md) — RDS and DynamoDB, for files 11–12 and 21.
- [IAM & Organizations](../02_iam/README.md) — account boundaries, federation and policy layers for files 22–24 and 26.
- [Monitoring & Config](../12_monitoring/README.md) and [Security Services](../13_security_services/README.md) — central evidence, findings and remediation for files 22–24 and 26.
- [Hybrid, Migration & DR](../14_hybrid_migration_dr/README.md) — assessment, cutover and recovery prerequisites for files 25 and 27.

---

**Next**: [01_vpc_public_private_subnets.md — Reference VPC Build](01_vpc_public_private_subnets.md)
