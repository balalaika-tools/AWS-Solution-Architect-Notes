# AWS Solutions Architect — Study Notes

> A structured, exam-focused path through the AWS Certified Solutions Architect – Associate (SAA-C03) domains. Every topic starts from the prerequisite concept (networking, DNS, identity, messaging theory) before explaining the AWS service, then covers *what it does, why it exists, when to use it, how it connects, and the exam traps*.

[![AWS](https://img.shields.io/badge/AWS-Solutions%20Architect-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)
[![Exam](https://img.shields.io/badge/Exam-SAA--C03-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)
[![Level](https://img.shields.io/badge/Level-Associate-FF9900.svg)](https://aws.amazon.com/certification/)

---

## Structure

```
aws-solutions-architect-notes/
│
│ ── FOUNDATIONS ─────────────────────────────────────────
├── 01_foundations/         Cloud computing model, AWS global infrastructure, shared responsibility
│
│ ── SECURITY & IDENTITY ──────────────────────────────────
├── 02_iam/                 AuthN vs AuthZ, users/groups/roles, policies, Organizations & STS
│
│ ── NETWORKING ───────────────────────────────────────────
├── 03_networking/          IP/CIDR primer, VPC, subnets, gateways, SG/NACL, peering, endpoints, TGW
│
│ ── COMPUTE ──────────────────────────────────────────────
├── 04_compute/             EC2, pricing models, AMIs, user data, metadata, placement groups
│
│ ── STORAGE ──────────────────────────────────────────────
├── 05_storage/             EBS, EFS, FSx, S3 (classes/lifecycle/security), Storage Gateway
│
│ ── DATABASES ────────────────────────────────────────────
├── 06_databases/           SQL/NoSQL theory, RDS, Aurora, DynamoDB, ElastiCache
│
│ ── HIGH AVAILABILITY & SCALING ──────────────────────────
├── 07_ha_scaling/          Load-balancing concepts, ELB/ALB/NLB/GWLB, Auto Scaling Groups
│
│ ── DNS & EDGE ───────────────────────────────────────────
├── 08_dns_edge/            DNS fundamentals, Route 53, CloudFront, Global Accelerator
│
│ ── CONTAINERS ───────────────────────────────────────────
├── 09_containers/          Container concepts, ECS, ECR, Fargate, EKS
│
│ ── SERVERLESS ───────────────────────────────────────────
├── 10_serverless/          Lambda, API Gateway, Step Functions, Cognito
│
│ ── MESSAGING & EVENTS ───────────────────────────────────
├── 11_messaging/           Decoupling theory, SQS, SNS, EventBridge, Kinesis
│
│ ── OBSERVABILITY ────────────────────────────────────────
├── 12_monitoring/          CloudWatch, CloudTrail, AWS Config
│
│ ── ENCRYPTION & SECURITY SERVICES ───────────────────────
├── 13_security_services/   KMS, Secrets Manager, ACM, WAF, Shield, GuardDuty, Inspector, Macie
│
│ ── HYBRID, MIGRATION & DR ───────────────────────────────
├── 14_hybrid_migration_dr/ Direct Connect, VPN, migration services, AWS Backup, DR strategies
│
│ ── COST & WELL-ARCHITECTED ──────────────────────────────
├── 15_cost_well_architected/  Cost optimization, the Well-Architected Framework
│
│ ── ANALYTICS & ML SURVEY ────────────────────────────────
├── 16_analytics_ml_survey/ Concise exam-clue summaries for analytics + AI/ML services
│
│ ── EXAM PATTERNS ────────────────────────────────────────
├── 17_exam_patterns/       Architecture decision guides, service comparison cheat sheets
│
│ ── PRACTICAL EXAMPLES ───────────────────────────────────
└── 18_practical_examples/  21 hands-on scenario walkthroughs
```

---

## SAA-C03 Domain Map

AWS weights the current SAA-C03 exam around four design domains. Use this table
as a final-review map after reading the service chapters.

| Exam domain | Weight | Strongest notes here |
|-------------|--------|----------------------|
| **Design secure architectures** | 30% | [IAM](02_iam/README.md), [Networking](03_networking/README.md), [Security Groups vs NACLs](03_networking/04_security_groups_vs_nacls.md), [ENIs & service networking](03_networking/07_enis_security_groups_and_service_networking.md), [Security services](13_security_services/README.md), [KMS examples](18_practical_examples/18_kms_encryption_examples.md) |
| **Design resilient architectures** | 26% | [VPC topology](03_networking/02_vpc_subnets_route_tables.md), [ELB & ASG](07_ha_scaling/README.md), [Route 53](08_dns_edge/02_route_53.md), [RDS/Aurora](06_databases/README.md), [DR strategies](14_hybrid_migration_dr/04_disaster_recovery.md), [practical HA scenarios](18_practical_examples/README.md) |
| **Design high-performing architectures** | 24% | [EC2 families](04_compute/01_ec2_fundamentals.md), [EBS/EFS/S3](05_storage/README.md), [DynamoDB](06_databases/04_dynamodb.md), [CloudFront & Global Accelerator](08_dns_edge/README.md), [containers/serverless](09_containers/README.md), [analytics survey](16_analytics_ml_survey/README.md) |
| **Design cost-optimized architectures** | 20% | [EC2 pricing](04_compute/02_ec2_pricing_models.md), [S3 classes](05_storage/04_s3_storage_classes_and_management.md), [Cost optimization](15_cost_well_architected/01_cost_optimization.md), [VPC endpoints vs NAT](03_networking/05_vpc_peering_and_endpoints.md), [decision guides](17_exam_patterns/01_architecture_decision_guides.md) |

---

## Contents

### 1. Foundations — [index](01_foundations/README.md)

| Guide | Description |
|-------|-------------|
| [Cloud Computing Fundamentals](01_foundations/01_cloud_computing_fundamentals.md) | IaaS/PaaS/SaaS, CapEx vs OpEx, the six advantages of cloud |
| [AWS Global Infrastructure](01_foundations/02_aws_global_infrastructure.md) | Regions, AZs, edge locations, Local/Wavelength zones |
| [Shared Responsibility & Account Basics](01_foundations/03_shared_responsibility_and_account_basics.md) | Security model, account structure, support plans |

### 2. IAM & Identity — [index](02_iam/README.md)

| Guide | Description |
|-------|-------------|
| [Authentication vs Authorization](02_iam/01_authn_vs_authz.md) | Identity concepts before the service |
| [Users, Groups & Roles](02_iam/02_users_groups_roles.md) | Principals, roles, instance profiles |
| [IAM Policies](02_iam/03_iam_policies.md) | Policy JSON, evaluation logic, identity vs resource policies |
| [Organizations, STS & Federation](02_iam/04_organizations_sts_federation.md) | SCPs, AssumeRole, identity federation, IAM Identity Center |

### 3. Networking — [index](03_networking/README.md)

| Guide | Description |
|-------|-------------|
| [Networking Primer](03_networking/01_networking_primer.md) | IP, CIDR, subnets, routing, ports, NAT, firewalls |
| [VPC, Subnets & Route Tables](03_networking/02_vpc_subnets_route_tables.md) | Building the virtual network |
| [Internet & NAT Gateways](03_networking/03_gateways_igw_nat.md) | Public vs private internet access |
| [Security Groups vs NACLs](03_networking/04_security_groups_vs_nacls.md) | Stateful vs stateless firewalls |
| [VPC Peering & Endpoints](03_networking/05_vpc_peering_and_endpoints.md) | Private connectivity, PrivateLink |
| [Transit Gateway & Advanced](03_networking/06_transit_gateway_and_advanced.md) | Hub-and-spoke, flow logs, IPv6 |
| [ENIs, Security Groups & Service Networking](03_networking/07_enis_security_groups_and_service_networking.md) | Which resources create ENIs, require SGs, and use endpoint types |

### 4. Compute — [index](04_compute/README.md)

| Guide | Description |
|-------|-------------|
| [EC2 Fundamentals](04_compute/01_ec2_fundamentals.md) | Instance families, lifecycle, networking |
| [EC2 Pricing Models](04_compute/02_ec2_pricing_models.md) | On-Demand, Reserved, Spot, Savings Plans |
| [AMIs, User Data & Metadata](04_compute/03_amis_userdata_metadata.md) | Golden images, bootstrapping, IMDS |
| [Placement Groups](04_compute/04_placement_groups.md) | Cluster, spread, partition |

### 5. Storage — [index](05_storage/README.md)

| Guide | Description |
|-------|-------------|
| [EBS](05_storage/01_ebs.md) | Block storage, volume types, snapshots |
| [EFS & FSx](05_storage/02_efs_and_fsx.md) | Shared file systems |
| [S3 Fundamentals](05_storage/03_s3_fundamentals.md) | Buckets, objects, durability, consistency |
| [S3 Storage Classes & Data Management](05_storage/04_s3_storage_classes_and_management.md) | Classes, lifecycle, versioning, replication, security |
| [S3 Advanced Features](05_storage/05_s3_advanced_features.md) | CORS, access logs, Access Points, Storage Lens, Batch Operations, performance, legacy S3 Select/Object Lambda notes |
| [Storage Gateway & Data Transfer](05_storage/06_storage_gateway_and_transfer.md) | Hybrid storage, DataSync, Transfer Family, Data Transfer Terminal, legacy Snow notes |

### 6. Databases — [index](06_databases/README.md)

| Guide | Description |
|-------|-------------|
| [Database Fundamentals](06_databases/01_database_fundamentals.md) | SQL vs NoSQL, OLTP vs OLAP, normalization |
| [RDS](06_databases/02_rds.md) | Managed relational, Multi-AZ, read replicas |
| [Aurora](06_databases/03_aurora.md) | Cloud-native relational, Serverless, Global DB |
| [DynamoDB](06_databases/04_dynamodb.md) | Managed NoSQL, capacity, GSIs, streams |
| [ElastiCache & Other Databases](06_databases/05_elasticache_and_others.md) | Valkey/Redis OSS/Memcached, Neptune, DocumentDB, Keyspaces, Timestream |

### 7. High Availability & Scaling — [index](07_ha_scaling/README.md)

| Guide | Description |
|-------|-------------|
| [Load Balancing Concepts](07_ha_scaling/01_load_balancing_concepts.md) | Why distribute traffic, listeners, target groups |
| [ELB: ALB, NLB & GWLB](07_ha_scaling/02_elb_alb_nlb_gwlb.md) | Choosing the right load balancer |
| [Auto Scaling Groups](07_ha_scaling/03_auto_scaling_groups.md) | Dynamic, scheduled, predictive scaling |

### 8. DNS & Edge — [index](08_dns_edge/README.md)

| Guide | Description |
|-------|-------------|
| [DNS Fundamentals](08_dns_edge/01_dns_fundamentals.md) | Resolution flow, record types, TTL |
| [Route 53](08_dns_edge/02_route_53.md) | Hosted zones, routing policies, health checks |
| [CloudFront](08_dns_edge/03_cloudfront.md) | CDN, origins, caching, OAC, security |
| [Global Accelerator](08_dns_edge/04_global_accelerator.md) | Anycast networking vs CloudFront |
| [Route 53 Resolver](08_dns_edge/05_route53_resolver.md) | `.2` VPC resolver, endpoints, DNS Firewall, query logging |

### 9. Containers — [index](09_containers/README.md)

| Guide | Description |
|-------|-------------|
| [Container Fundamentals](09_containers/01_container_fundamentals.md) | Containers vs VMs, images, registries |
| [ECS, ECR, Fargate & EKS](09_containers/02_ecs_ecr_fargate_eks.md) | Orchestration choices on AWS |

### 10. Serverless — [index](10_serverless/README.md)

| Guide | Description |
|-------|-------------|
| [Lambda](10_serverless/01_lambda.md) | Functions, triggers, concurrency, VPC |
| [API Gateway](10_serverless/02_api_gateway.md) | REST/HTTP/WebSocket APIs, auth, throttling |
| [Step Functions](10_serverless/03_step_functions.md) | State machines, orchestration patterns |
| [Cognito](10_serverless/04_cognito.md) | User pools vs identity pools |

### 11. Messaging & Events — [index](11_messaging/README.md)

| Guide | Description |
|-------|-------------|
| [Messaging & Decoupling Concepts](11_messaging/01_messaging_concepts.md) | Sync vs async, queues vs pub/sub vs streams |
| [SQS](11_messaging/02_sqs.md) | Standard vs FIFO, visibility timeout, DLQ |
| [SNS](11_messaging/03_sns.md) | Pub/sub, fan-out, filtering |
| [EventBridge](11_messaging/04_eventbridge.md) | Event bus, rules, schema registry |
| [Kinesis & Data Firehose](11_messaging/05_kinesis.md) | Data Streams, Amazon Data Firehose, real-time analytics |

### 12. Monitoring & Auditing — [index](12_monitoring/README.md)

| Guide | Description |
|-------|-------------|
| [CloudWatch](12_monitoring/01_cloudwatch.md) | Metrics, alarms, logs, dashboards |
| [CloudTrail](12_monitoring/02_cloudtrail.md) | API audit logging |
| [AWS Config](12_monitoring/03_config.md) | Resource configuration compliance |

### 13. Encryption & Security Services — [index](13_security_services/README.md)

| Guide | Description |
|-------|-------------|
| [Encryption & KMS](13_security_services/01_encryption_and_kms.md) | Symmetric/asymmetric, envelope encryption, KMS keys |
| [Secrets Manager & ACM](13_security_services/02_secrets_manager_and_acm.md) | Secret rotation, TLS certificates |
| [Threat Detection & Protection](13_security_services/03_threat_detection_services.md) | WAF, Shield, GuardDuty, Inspector, Macie |

### 14. Hybrid, Migration & DR — [index](14_hybrid_migration_dr/README.md)

| Guide | Description |
|-------|-------------|
| [Hybrid Networking](14_hybrid_migration_dr/01_hybrid_networking.md) | Site-to-Site VPN, Direct Connect |
| [Migration Services](14_hybrid_migration_dr/02_migration_services.md) | DMS, MGN, DataSync, Transfer Family, current physical-transfer options |
| [AWS Backup](14_hybrid_migration_dr/03_backup.md) | Centralized backup & retention |
| [Disaster Recovery Strategies](14_hybrid_migration_dr/04_disaster_recovery.md) | Backup/restore, pilot light, warm standby, active-active |

### 15. Cost & Well-Architected — [index](15_cost_well_architected/README.md)

| Guide | Description |
|-------|-------------|
| [Cost Optimization](15_cost_well_architected/01_cost_optimization.md) | Billing tools, pricing levers, right-sizing |
| [Well-Architected Framework](15_cost_well_architected/02_well_architected_framework.md) | The six pillars and design principles |

### 16. Analytics & ML Survey — [index](16_analytics_ml_survey/README.md)

| Guide | Description |
|-------|-------------|
| [Analytics Services](16_analytics_ml_survey/01_analytics_services.md) | Athena, Glue, Redshift, EMR, OpenSearch, Amazon Quick Sight, MSK |
| [AI/ML Services](16_analytics_ml_survey/02_ml_ai_services.md) | Rekognition, Textract, Comprehend, SageMaker, etc. |

### 17. Exam Patterns — [index](17_exam_patterns/README.md)

| Guide | Description |
|-------|-------------|
| [Architecture Decision Guides](17_exam_patterns/01_architecture_decision_guides.md) | "Choose X vs Y" decision trees by scenario |
| [Service Comparison Cheat Sheets](17_exam_patterns/02_service_comparison_cheatsheets.md) | Side-by-side tables and keyword triggers |

### 18. Practical Examples — [index](18_practical_examples/README.md)

| Guide | Description |
|-------|-------------|
| [VPC with Public & Private Subnets](18_practical_examples/01_vpc_public_private_subnets.md) | Reference network build |
| [EC2 in Private Subnet via NAT](18_practical_examples/02_ec2_private_subnet_nat.md) | Outbound internet without exposure |
| [ALB → Private EC2](18_practical_examples/03_alb_to_private_ec2.md) | Public ingress to private compute |
| [Security Group vs NACL Examples](18_practical_examples/04_sg_vs_nacl_examples.md) | Concrete rule walkthroughs |
| [VPC Peering Implementation](18_practical_examples/05_vpc_peering.md) | Connecting two VPCs |
| [VPC Peering DNS Resolution](18_practical_examples/06_vpc_peering_dns_resolution.md) | When private DNS resolves (and when it doesn't) |
| [Route 53 Public Hosted Zone](18_practical_examples/07_route53_public_hosted_zone.md) | Public domain setup |
| [Route 53 Private Hosted Zone](18_practical_examples/08_route53_private_hosted_zone.md) | Internal DNS |
| [Route 53 Resolver / Hybrid DNS](18_practical_examples/09_route53_resolver_hybrid_dns.md) | Inbound/outbound endpoints |
| [S3 Static Site vs CloudFront + S3](18_practical_examples/10_s3_static_site_vs_cloudfront.md) | Hosting trade-offs |
| [RDS Private Access from EC2](18_practical_examples/11_rds_private_access.md) | Secure DB connectivity |
| [RDS Multi-AZ vs Read Replica](18_practical_examples/12_rds_multiaz_vs_read_replica.md) | HA vs scaling reads |
| [ECS/Fargate Service Behind ALB](18_practical_examples/13_ecs_fargate_behind_alb.md) | Container service exposure |
| [Lambda Inside vs Outside a VPC](18_practical_examples/14_lambda_in_vpc.md) | Networking trade-offs |
| [SQS with Auto Scaling Group](18_practical_examples/15_sqs_with_asg.md) | Queue-depth scaling |
| [CloudWatch Alarm Triggering Scaling](18_practical_examples/16_cloudwatch_alarm_scaling.md) | Metric-driven automation |
| [CloudTrail vs CloudWatch vs Config](18_practical_examples/17_cloudtrail_vs_cloudwatch_vs_config.md) | Which tool answers which question |
| [KMS Encryption Examples](18_practical_examples/18_kms_encryption_examples.md) | Encrypting EBS, S3, RDS, secrets |
| [Disaster Recovery Strategies](18_practical_examples/19_disaster_recovery_strategies.md) | The four DR patterns compared |
| [VPC Endpoints for Private AWS Services](18_practical_examples/20_vpc_endpoints_private_aws_services.md) | Private S3/ECR/Logs/Secrets/SQS access without public IPs/NAT |
| [Serverless API with API Gateway, Lambda, DynamoDB](18_practical_examples/21_api_gateway_lambda_dynamodb.md) | HTTP API, Lambda proxy integration, DynamoDB access patterns |

---

## Reading Order

> [!TIP]
> Pick the path that matches your goal. If you're new to networking, do not skip Section 3's primer.

### Full Certification Path (recommended)

Work the directories in numeric order, `01` → `18`. Each builds on the prior: foundations and IAM frame everything, networking underpins every other service, and the practical examples in `18` tie multiple services together.

### Networking-First Path (for the networking-light)

1. [Networking Primer](03_networking/01_networking_primer.md) — IP, CIDR, subnets, ports before anything AWS
2. [VPC, Subnets & Route Tables](03_networking/02_vpc_subnets_route_tables.md) — the virtual network
3. [Internet & NAT Gateways](03_networking/03_gateways_igw_nat.md) — public vs private access
4. [Security Groups vs NACLs](03_networking/04_security_groups_vs_nacls.md) — firewalls
5. [DNS Fundamentals](08_dns_edge/01_dns_fundamentals.md) → [Route 53](08_dns_edge/02_route_53.md)
6. [Practical: VPC Public/Private Subnets](18_practical_examples/01_vpc_public_private_subnets.md) and the networking scenarios that follow

### Exam-Crunch Path (review before test day)

1. [Service Comparison Cheat Sheets](17_exam_patterns/02_service_comparison_cheatsheets.md)
2. [Architecture Decision Guides](17_exam_patterns/01_architecture_decision_guides.md)
3. [Well-Architected Framework](15_cost_well_architected/02_well_architected_framework.md)
4. The "Key Exam Points" and "Common Mistakes" sections in each service file

### Practical / Hands-On Path

Skim the conceptual file for a service, then jump straight to the matching scenario in [18_practical_examples](18_practical_examples/README.md).
```
# AWS-Solution-Architect-Notes
