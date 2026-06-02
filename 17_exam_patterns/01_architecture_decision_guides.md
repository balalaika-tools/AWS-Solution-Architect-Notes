# Architecture Decision Guides

> **Who this is for**: An engineer in final SAA-C03 review who already knows what each service *does* and now needs to choose between them fast. Every section is a scenario-driven "if you need… → choose…" guide. Exam questions are decisions in disguise — this file trains the reflex.

> **Principle**: The exam almost never asks "what does X do?" It asks "which of these four valid-looking options is *best* for this requirement?" The right answer is the one that nails the discriminator the question is testing: latency, cost, ordering, durability, operational overhead, or blast radius. Read every question for that discriminator first.

---

## 1. Compute: EC2 vs Lambda vs ECS/Fargate vs EKS vs Batch

The first split is **always**: is the work event-driven and short, or long-running/stateful?

```
                        ┌─ Run < 15 min, spiky/event-driven, pay-per-ms?
                        │      └─► Lambda
   Need compute? ───────┤
                        │   Run containers? ──┬─ Need Kubernetes API / multi-cloud portability? ─► EKS
                        │                      └─ Just want to run containers, no servers? ─────► ECS on Fargate
                        │
                        ├─ Large batch / queue of jobs, scientific/HPC, Spot-friendly? ─► AWS Batch
                        │
                        └─ Need OS-level control, licensing, GPUs, long-running, lift-and-shift? ─► EC2
```

| If you need… | → Choose | Why |
|---|---|---|
| Event-driven glue, < 15 min, no idle cost | **Lambda** | Pay per invocation/ms; scales to zero; 15-min hard ceiling |
| Containers without managing servers or the control plane | **ECS on Fargate** | Serverless containers; lowest ops overhead for containers |
| Containers with control of the host (GPUs, daemonsets, custom kernel) | **ECS/EKS on EC2** | You manage the EC2 capacity |
| Kubernetes specifically (existing k8s tooling, portability) | **EKS** | Managed control plane; pay $0.10/hr per cluster |
| Full OS control, specialized instances, commercial licenses, lift-and-shift VMs | **EC2** | Maximum flexibility, maximum ops responsibility |
| Run thousands of queued batch jobs, HPC, array jobs | **AWS Batch** | Manages job queues + Spot provisioning for you |
| Cheapest fault-tolerant compute, interruptible | **EC2 Spot / Fargate Spot** | Up to 90% off; can be reclaimed with 2-min notice |

✅ "No servers to manage" + "containers" → **Fargate**. "No servers to manage" + "function/event" → **Lambda**.
❌ Don't pick EKS just because the workload is containerized — EKS implies the question wants Kubernetes specifically (operators, Helm, existing k8s skills).
⚠️ Lambda's **15-minute** max means any "process for hours/long-running" scenario rules it out → ECS/Fargate, Batch, or EC2.

> See detail: [Compute](../04_compute/README.md) · [Containers](../09_containers/README.md) · [Serverless / Lambda](../10_serverless/01_lambda.md)

---

## 2. Database: RDS vs Aurora vs DynamoDB vs ElastiCache vs Redshift

First split: **relational vs key-value vs analytics vs cache**.

```
   Need a data store? ─┬─ Sub-millisecond key/value, massive scale, serverless? ─► DynamoDB
                       │
                       ├─ Relational (SQL, joins, transactions)? ──┬─ Want max performance/HA, AWS-native? ─► Aurora
                       │                                            └─ Need a specific engine/version, lower cost? ─► RDS
                       │
                       ├─ Analytics / OLAP / complex aggregations over TBs–PBs? ─► Redshift
                       │
                       └─ In-memory cache / leaderboard / session store / sub-ms reads? ─► ElastiCache
```

| If you need… | → Choose | Why |
|---|---|---|
| Managed MySQL/PostgreSQL/MariaDB/Oracle/SQL Server | **RDS** | Pick when a specific engine or lowest cost matters |
| MySQL/PostgreSQL-compatible with 5×/3× throughput, 6 copies across 3 AZs, fast failover | **Aurora** | AWS-native, auto-scaling storage to 128 TiB |
| Variable/unpredictable relational load, scale-to-zero relational | **Aurora Serverless v2** | Auto-scales capacity, pay for what you use |
| Single-digit-ms NoSQL, key-value/document, any scale, serverless | **DynamoDB** | No servers; on-demand or provisioned capacity |
| Microsecond reads in front of DynamoDB | **DynamoDB + DAX** | In-memory cache purpose-built for DynamoDB |
| In-memory cache, session store, leaderboard, pub/sub, geospatial | **ElastiCache (Redis)** | Sub-ms; Redis adds persistence/replication/sorted sets |
| Simplest cache, multi-threaded, no persistence needed | **ElastiCache (Memcached)** | Pure cache; horizontal sharding, no replication |
| Petabyte-scale data warehouse, BI dashboards, complex SQL aggregations | **Redshift** | Columnar OLAP; not for OLTP |
| Graph relationships | **Neptune** · Time-series → **Timestream** · Ledger → **QLDB** | Purpose-built engines |

✅ "Single-digit millisecond" + "NoSQL/scale" → **DynamoDB**. "Microsecond" + DynamoDB → **DAX**.
✅ "Complex queries / data warehouse / OLAP / BI" → **Redshift** (never RDS/DynamoDB).
❌ DynamoDB is wrong when the question needs **joins, complex queries, or strong relational integrity** → RDS/Aurora.
⚠️ ElastiCache and DAX are caches, not systems of record — they sit *in front of* a database.

> See detail: [Databases](../06_databases/README.md) · [ElastiCache & others](../06_databases/05_elasticache_and_others.md) · [Analytics / Redshift](../16_analytics_ml_survey/01_analytics_services.md)

---

## 3. Storage: EBS vs EFS vs FSx vs S3 vs Instance Store

First split: **block vs file vs object**, then **single-attach vs shared**.

| If you need… | → Choose | Access model |
|---|---|---|
| A boot disk or single-instance block volume | **EBS** | One AZ; attach to one EC2 (Multi-Attach only io1/io2) |
| Temporary scratch, highest IOPS, data can vanish on stop | **Instance Store** | Ephemeral, physically attached |
| Shared POSIX file system across many Linux EC2 / containers / Lambda, auto-scaling | **EFS** | NFS, multi-AZ, thousands of clients |
| Windows file shares (SMB), Active Directory integration | **FSx for Windows** | Managed SMB |
| High-performance compute (HPC, ML training) over S3 data | **FSx for Lustre** | Sub-ms, links to S3 |
| NetApp ONTAP features / VMware / multi-protocol | **FSx for NetApp ONTAP** | Snapshots, dedup, NFS+SMB+iSCSI |
| Object storage: web assets, backups, data lake, infinite scale, 11 9s durability | **S3** | HTTP API, not a mounted disk |
| Archive at lowest cost, retrieval in minutes–hours | **S3 Glacier / Deep Archive** | Cold object storage |

```
   Block (a disk)? ──┬─ Ephemeral & fastest, OK to lose? ─► Instance Store
                     └─ Persistent, attach to one instance? ─► EBS

   File (mountable, shared)? ──┬─ Linux/NFS, many clients? ─► EFS
                               ├─ Windows/SMB + AD? ───────► FSx for Windows
                               └─ HPC/ML over S3? ─────────► FSx for Lustre

   Object (HTTP, infinite scale)? ─► S3 (→ Glacier for archive)
```

✅ "Shared across many instances/AZs" + Linux → **EFS**. + Windows → **FSx for Windows**.
✅ "Multiple instances must attach the *same volume*" + you insist on block → **EBS Multi-Attach (io1/io2)**, but the exam usually wants **EFS** for shared file access.
❌ EBS cannot be shared across AZs and (outside Multi-Attach) not across instances.
⚠️ Instance Store data is **lost on stop/terminate/hardware failure** — never for durable data.

> See detail: [Storage](../05_storage/README.md) · [EBS](../05_storage/01_ebs.md) · [EFS & FSx](../05_storage/02_efs_and_fsx.md) · [S3 classes](../05_storage/04_s3_storage_classes_and_management.md)

---

## 4. Load Balancer: ALB vs NLB vs GWLB

Discriminator: **OSI layer + what you're balancing**.

| If you need… | → Choose | Layer |
|---|---|---|
| HTTP/HTTPS routing by path, host, header; web apps, microservices, containers | **ALB** | L7 |
| Extreme performance, millions of req/s, ultra-low latency, **static IP / Elastic IP**, TCP/UDP, PrivateLink target | **NLB** | L4 |
| Deploy/scale 3rd-party virtual appliances (firewalls, IDS/IPS, DPI) transparently | **GWLB** | L3 (GENEVE) |

```
   HTTP-aware routing (paths, hosts, headers, WAF, OIDC auth)? ─► ALB
   Raw TCP/UDP, static IP, millions of req/s, lowest latency?  ─► NLB
   Inserting inline security appliances?                       ─► GWLB
```

✅ "Static IP" or "millions of requests" or "TCP/UDP" or "PrivateLink endpoint service" → **NLB**.
✅ "Route by URL path / hostname", "host multiple apps on one LB", "WAF", "authenticate via Cognito/OIDC" → **ALB**.
✅ "Third-party firewall / inspection appliance fleet" → **GWLB**.
⚠️ Only **NLB** preserves the client source IP by default and supports assigning a fixed Elastic IP.

> See detail: [ELB: ALB, NLB & GWLB](../07_ha_scaling/02_elb_alb_nlb_gwlb.md)

---

## 5. Network Security Controls: SG/NACL vs WAF/Shield vs Network Firewall vs GWLB

Discriminator: **where** in the traffic path you need enforcement.

| If you need… | → Choose | Scope |
|---|---|---|
| Allow-only, stateful firewall on EC2/RDS/Lambda/ECS task/interface endpoint ENIs | **Security Group** | Resource/ENI |
| Stateless subnet guardrail or explicit deny by IP CIDR | **Network ACL** | Subnet |
| Layer-7 HTTP protection: SQLi/XSS, managed rules, rate limits, geo match | **AWS WAF** | CloudFront, ALB, API Gateway, AppSync |
| DDoS protection | **Shield Standard/Advanced** | Edge/L3-L4 plus response features |
| Centralized stateful/stateless inspection across VPC subnets | **AWS Network Firewall** | VPC traffic path |
| Scale third-party firewall/IDS/IPS appliances inline | **Gateway Load Balancer** | Appliance fleet |
| Centrally manage WAF/Shield/SG policies across accounts | **Firewall Manager** | Organizations-wide policy |

```
Resource allow rules?      -> Security Group
Subnet explicit deny?      -> NACL
HTTP attack filtering?     -> WAF
DDoS?                      -> Shield
Managed VPC firewall?      -> Network Firewall
Third-party appliance?     -> GWLB
Multi-account enforcement? -> Firewall Manager
```

> See detail: [Security Groups vs NACLs](../03_networking/04_security_groups_vs_nacls.md) · [Threat Detection & Protection](../13_security_services/03_threat_detection_services.md)

---

## 6. Decoupling: SQS vs SNS vs EventBridge vs Kinesis

Discriminator: **queue (one consumer pulls) vs pub/sub (fan-out) vs event router (rules/SaaS) vs stream (ordered, replayable)**.

| If you need… | → Choose | Model |
|---|---|---|
| One-to-one work queue, buffer/decouple producer & consumer, retries, DLQ | **SQS** | Queue, consumer polls |
| Exactly-once + strict ordering in a queue | **SQS FIFO** | Ordered queue (dedup) |
| Fan-out one message to many subscribers (push) | **SNS** | Pub/sub, push |
| Durable fan-out: every consumer gets every message + replay buffer | **SNS → multiple SQS** (fan-out) | Pub/sub + queues |
| Route events by content to many targets, SaaS integrations, schedule, event bus | **EventBridge** | Event router, rules + filtering |
| Ordered, replayable real-time stream; many consumers read same data; analytics | **Kinesis Data Streams** | Stream, retention 1–365 days |
| Just load streaming data into S3/Redshift/OpenSearch, no code | **Kinesis Data Firehose** | Managed delivery, near-real-time |

```
   Buffer work, one logical consumer pulls?      ─► SQS  (FIFO if order+exactly-once)
   Push one event to many subscribers now?       ─► SNS  (+SQS for durable fan-out)
   Route/filter events to many targets, SaaS,
     schedule, AWS-service events?               ─► EventBridge
   Replayable ordered stream, multiple readers,
     real-time analytics, high throughput?       ─► Kinesis Data Streams
```

✅ "Decouple" / "buffer" / "process when ready" / "smooth spikes" → **SQS**.
✅ "Fan-out" / "notify multiple systems" → **SNS** (durable fan-out = SNS + SQS).
✅ "Event-driven", "rules", "match on content", "third-party SaaS source", "cron/schedule" → **EventBridge**.
✅ "Real-time", "ordered", "replay", "multiple consumers of the same data", "clickstream/IoT/telemetry" → **Kinesis**.
❌ SQS messages are consumed and deleted — no replay; if you need replay or multiple independent readers of the same data, use **Kinesis**.
⚠️ SNS push vs SQS pull: SNS is fire-and-forget to subscribers; SQS holds the message until a consumer pulls it.

> See detail: [Messaging concepts](../11_messaging/01_messaging_concepts.md) · [SQS](../11_messaging/02_sqs.md) · [SNS](../11_messaging/03_sns.md) · [EventBridge](../11_messaging/04_eventbridge.md) · [Kinesis](../11_messaging/05_kinesis.md)

---

## 7. DNS & Edge — Route 53 routing policy + CloudFront vs Global Accelerator

### 6a. Route 53 routing policy

| If you need… | → Choose |
|---|---|
| One record, no logic | **Simple** |
| Failover to a standby when primary is unhealthy (active-passive DR) | **Failover** |
| Lowest-latency region for the user | **Latency** |
| Route by user's geographic location (compliance, localization, geo-blocking) | **Geolocation** |
| Route by geographic *distance* with bias adjustment | **Geoproximity** (Traffic Flow) |
| Split traffic by assigned percentage (A/B, canary) | **Weighted** |
| Return multiple healthy IPs, client picks (poor-man's LB) | **Multivalue answer** |
| Route by user attributes via DNS (e.g., subdivisions of geolocation) | **IP-based** |

✅ "Active-passive DR" → **Failover**. "Closest/lowest latency" → **Latency**. "Comply with data residency / block a country" → **Geolocation**. "Canary / weighted split" → **Weighted**.

### 6b. CloudFront vs Global Accelerator

| If you need… | → Choose |
|---|---|
| Cache static/dynamic **HTTP(S)** content at the edge, lower origin load, WAF at edge | **CloudFront** |
| Improve performance/failover for **TCP/UDP** apps via AWS backbone + 2 static anycast IPs, instant regional failover | **Global Accelerator** |
| Non-HTTP protocols (gaming, IoT, VoIP), fixed entry IPs, fast regional failover | **Global Accelerator** |
| Content delivery, video streaming, static websites | **CloudFront** |

✅ "Cache content" / "CDN" / "static website + HTTPS" → **CloudFront**.
✅ "Static anycast IPs" / "non-HTTP / TCP-UDP" / "fast failover across Regions" / "no caching needed" → **Global Accelerator**.

> See detail: [Route 53](../08_dns_edge/02_route_53.md) · [CloudFront](../08_dns_edge/03_cloudfront.md) · [Global Accelerator](../08_dns_edge/04_global_accelerator.md)

---

## 8. High Availability & DR — Multi-AZ vs Multi-Region & DR strategy choice

### 7a. Scope: Multi-AZ vs Multi-Region

| If you need… | → Choose |
|---|---|
| Survive an **AZ** failure within one Region (typical HA requirement) | **Multi-AZ** |
| Survive a **Region** failure, meet data-residency, or serve global low latency | **Multi-Region** |
| Lowest-RTO read scaling within a Region | Multi-AZ + read replicas |

✅ Default exam answer for "highly available" is **Multi-AZ**. Only go **Multi-Region** when the question says "Region outage", "global users", "data sovereignty", or "RTO/RPO near zero across Regions".

### 7b. DR strategy (by RTO/RPO and cost)

```
   Cheapest, slowest recovery ──────────────────────────► most expensive, fastest
   Backup & Restore   →   Pilot Light   →   Warm Standby   →   Multi-Site Active/Active
   (hours, RPO hrs)      (10s of min)      (minutes)          (near-zero RTO/RPO)
```

| Strategy | RTO/RPO | What's running in DR region | Pick when |
|---|---|---|---|
| **Backup & Restore** | Hours / hours | Nothing; restore from backups | Lowest cost, tolerant of downtime |
| **Pilot Light** | 10s of minutes | Core (DB replicating), app servers off | Critical data live, scale up on failover |
| **Warm Standby** | Minutes | Full stack running at reduced capacity | Low RTO, can't afford long downtime |
| **Multi-Site Active/Active** | Near zero | Full production in both Regions | Mission-critical, near-zero RTO/RPO |

💡 RTO = how long to recover; RPO = how much data you can lose. Lower numbers cost more. Match the cheapest strategy that meets the stated RTO/RPO.

> See detail: [Disaster Recovery Strategies](../14_hybrid_migration_dr/04_disaster_recovery.md) · [Auto Scaling](../07_ha_scaling/03_auto_scaling_groups.md)

---

## 9. Connectivity: VPC Peering vs Transit Gateway vs PrivateLink vs VPN vs Direct Connect

### 8a. VPC-to-VPC and on-prem connectivity

| If you need… | → Choose | Notes |
|---|---|---|
| Connect **two** VPCs privately, simple, 1:1 | **VPC Peering** | Non-transitive; no overlapping CIDRs |
| Connect **many** VPCs + on-prem in a hub-and-spoke | **Transit Gateway** | Transitive routing at scale |
| Expose/consume **one specific service** privately (no full network access) | **PrivateLink / Interface Endpoint** | Service-level, not network-level |
| Encrypted on-prem ↔ AWS over the **internet**, quick to set up | **Site-to-Site VPN** | ~1.25 Gbps/tunnel, internet-dependent |
| **Dedicated**, consistent, high-bandwidth, private on-prem ↔ AWS | **Direct Connect (DX)** | Weeks to provision; not encrypted by itself |
| DX security/encryption | **DX + VPN (IPsec over DX)** | Private *and* encrypted |
| Cheap DR / backup path for Direct Connect | **VPN as backup to DX** | Fail over to internet VPN |

```
   2 VPCs, simple?                      ─► VPC Peering (non-transitive)
   Many VPCs + on-prem, hub/spoke?      ─► Transit Gateway
   Share ONE service privately?         ─► PrivateLink (Interface Endpoint)
   On-prem ↔ AWS over internet, fast?   ─► Site-to-Site VPN
   On-prem ↔ AWS, dedicated/consistent? ─► Direct Connect (+VPN for encryption)
```

✅ "Two VPCs" → **Peering**. "Hundreds of VPCs / centralize" → **Transit Gateway**. "Expose a single app to other accounts privately" → **PrivateLink**.
✅ "Consistent throughput / dedicated / not over internet" → **Direct Connect**. "Quick / encrypted / over internet" → **VPN**.
❌ VPC Peering is **not transitive** — A↔B and B↔C does not give A↔C. That's the classic trap; use **TGW** for transitive routing.

> See detail: [VPC Peering & Endpoints](../03_networking/05_vpc_peering_and_endpoints.md) · [Transit Gateway](../03_networking/06_transit_gateway_and_advanced.md) · [Hybrid Networking](../14_hybrid_migration_dr/01_hybrid_networking.md)

---

### 9b. VPC endpoint chooser

| If you need… | → Choose | Why |
|---|---|---|
| Private S3/DynamoDB from workloads in the same VPC, lowest cost | **Gateway endpoint** | Free route-table endpoint; no ENI/SG |
| Private AWS API access such as Secrets, KMS, SQS, ECR, STS, CloudWatch Logs | **Interface endpoint** | PrivateLink ENI with SG + private DNS |
| Private API Gateway API | **Interface endpoint for `execute-api`** | Private API is reached through PrivateLink |
| Private service exposed by another account/SaaS provider | **PrivateLink endpoint service + interface endpoint** | One-service access, no full VPC routing |
| Inspection appliance path | **Gateway Load Balancer endpoint** | Route traffic through GWLB appliances |

> See detail: [ENIs, Security Groups & Service Networking](../03_networking/07_enis_security_groups_and_service_networking.md) · [Private AWS Service Access with VPC Endpoints](../18_practical_examples/20_vpc_endpoints_private_aws_services.md)

---

## 10. Caching — where to cache

| If you need to cache… | → Choose |
|---|---|
| Static/dynamic web content near users | **CloudFront** (edge) |
| Database query results / sessions / leaderboards in memory | **ElastiCache (Redis/Memcached)** |
| Microsecond reads in front of **DynamoDB** | **DAX** |
| API responses | **API Gateway caching** |
| Frequently read S3 objects globally | **CloudFront + S3 origin (OAC)** |

```
   Where is the hot data?
     At the edge / for browsers? ─► CloudFront
     In front of a SQL/NoSQL DB? ─► ElastiCache  (DynamoDB-specific → DAX)
     API layer?                  ─► API Gateway cache
```

✅ "Reduce read load on RDS/Aurora" → **ElastiCache**. "Reduce latency to DynamoDB to microseconds" → **DAX**. "Reduce origin load / serve globally" → **CloudFront**.

> See detail: [ElastiCache](../06_databases/05_elasticache_and_others.md) · [CloudFront](../08_dns_edge/03_cloudfront.md) · [API Gateway](../10_serverless/02_api_gateway.md)

---

## 11. Secrets: Secrets Manager vs Parameter Store

| If you need… | → Choose |
|---|---|
| **Automatic rotation** of DB credentials/API keys, cross-account sharing | **Secrets Manager** |
| Store config + secrets cheaply (or free standard tier), no built-in auto-rotation | **SSM Parameter Store** |
| Plain config values, feature flags, non-secret parameters | **Parameter Store (String)** |
| Secrets but want to minimize cost and rotation isn't required | **Parameter Store (SecureString + KMS)** |

✅ "**Automatically rotate** credentials" / "managed rotation with RDS" → **Secrets Manager** (this keyword is the discriminator).
✅ "Store configuration / lots of parameters / minimize cost" → **Parameter Store**.
💡 Both encrypt with KMS. Parameter Store standard tier is free; Secrets Manager charges per secret per month but gives you rotation out of the box.

> See detail: [Secrets Manager & ACM](../13_security_services/02_secrets_manager_and_acm.md) · [Encryption & KMS](../13_security_services/01_encryption_and_kms.md)

---

## 12. Putting it together — the question-reading checklist

> **Rule**: Before choosing, underline the single discriminator the question is testing. Then the answer is forced.

| The question stresses… | Lean toward |
|---|---|
| "Least operational overhead" / "fully managed" / "serverless" | Lambda, Fargate, Aurora Serverless, DynamoDB, S3 |
| "Most cost-effective" / "minimize cost" | Spot, S3 Glacier/Intelligent-Tiering, right-size, Parameter Store |
| "Highly available" (within Region) | Multi-AZ |
| "Disaster recovery" / "Region failure" | Multi-Region + a DR strategy matched to RTO/RPO |
| "Decouple" / "buffer" / "smooth spikes" | SQS |
| "Real-time" / "ordered" / "replay" | Kinesis |
| "Single-digit ms" + scale | DynamoDB |
| "Static IP" / "millions of req/s" / "TCP-UDP" | NLB |
| "Route by URL/host" / "WAF" | ALB |
| "Cache content at edge" | CloudFront |
| "Non-HTTP" / "anycast static IP" / "fast Region failover" | Global Accelerator |
| "Automatically rotate secrets" | Secrets Manager |
| "Two VPCs" vs "many VPCs" | Peering vs Transit Gateway |

---

**Next**: [Service Comparison Cheat Sheets](02_service_comparison_cheatsheets.md)
