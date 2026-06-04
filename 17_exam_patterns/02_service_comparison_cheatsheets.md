# Service Comparison Cheat Sheets

> **Who this is for**: An engineer doing rapid-fire final review before SAA-C03. This is the densest file in the repo — a keyword→service trigger table, the classic pairwise comparisons every exam relies on, and the hard limits worth memorizing. Cover the right column and recall the answer.

> **Key insight**: SAA-C03 questions are pattern-matching exercises. A handful of keywords reliably point at one service. Burn these triggers into memory and most questions answer themselves before you finish reading the options.

---

## 1. Keyword → Service Trigger Table

| Keyword / phrase in the question | → Service / Answer |
|---|---|
| Millions of requests/sec + static IP + TCP/UDP | **NLB** |
| Route by URL path / hostname / HTTP header, WAF, OIDC auth | **ALB** |
| Insert third-party firewall / IDS/IPS appliances inline | **Gateway Load Balancer (GWLB)** |
| Extract text/data from scanned docs, forms, tables | **Textract** |
| Detect objects/faces/inappropriate content in images & video | **Rekognition** |
| Sentiment / entities / key phrases / PII in text (NLP) | **Comprehend** |
| Text-to-speech (lifelike voice) | **Polly** · Speech-to-text → **Transcribe** |
| Translate language | **Translate** · Chatbot → **Lex** |
| Build, train, deploy custom ML models | **SageMaker** |
| Exactly-once, strictly ordered messages | **SQS FIFO** |
| Decouple / buffer work, retries, dead-letter queue | **SQS (Standard)** |
| Fan-out one message to many subscribers (push) | **SNS** |
| Durable fan-out (each consumer gets every message) | **SNS → multiple SQS** |
| Route events by content, schedule (cron), SaaS sources | **EventBridge** |
| Real-time ordered stream, replay, clickstream/IoT/telemetry | **Kinesis Data Streams** |
| Load streaming data into S3/Redshift/OpenSearch, no code | **Amazon Data Firehose** (formerly Kinesis Data Firehose) |
| Sub-millisecond NoSQL key-value at any scale | **DynamoDB** |
| Microsecond reads in front of DynamoDB | **DAX** |
| In-memory cache, session store, leaderboard, pub/sub | **ElastiCache (Valkey/Redis OSS)** |
| Petabyte data warehouse / OLAP / BI dashboards | **Redshift** |
| Query data in S3 with SQL, serverless, pay-per-scan | **Athena** |
| Serverless ETL, data catalog, crawlers | **AWS Glue** |
| Managed Hadoop/Spark/Presto big-data clusters | **EMR** |
| Managed search / log analytics / Kibana dashboards | **OpenSearch Service** |
| Managed Apache Kafka | **Amazon MSK** |
| BI dashboards / visualizations | **Amazon Quick Sight** |
| DDoS protection (L3/L4 automatic; L7 advanced) | **Shield (Standard/Advanced)** |
| Block SQLi/XSS, rate-limit, geo-block at L7 | **WAF** |
| Stateful/stateless managed firewall inside a VPC | **AWS Network Firewall** |
| Centrally manage WAF/Shield/SG policies across accounts | **Firewall Manager** |
| Who did what API call (audit trail) | **CloudTrail** |
| Metrics, alarms, logs, dashboards (how is it performing) | **CloudWatch** |
| Is the resource configured compliantly / change history | **AWS Config** |
| Detect threats from logs (VPC Flow/DNS/CloudTrail) via ML | **GuardDuty** |
| Scan EC2/ECR/Lambda for CVEs & vulnerabilities | **Inspector** |
| Discover & classify PII/sensitive data in S3 | **Macie** |
| Centralized security findings aggregation | **Security Hub** |
| Petabyte-scale offline data transfer in older exam wording (truck/box) | **Snowball / Snowmobile** legacy recognition; new customers evaluate DataSync/Data Transfer Terminal/partners |
| Online data transfer to/from AWS over network, scheduled | **DataSync** |
| Hybrid storage: cache S3 on-prem (file/volume/tape) | **Storage Gateway** |
| Migrate databases (homogeneous & heterogeneous) | **DMS** (+ SCT for schema) |
| Lift-and-shift / rehost servers to EC2 | **Application Migration Service (MGN)** |
| Infrastructure as code / repeatable stacks | **CloudFormation** |
| Patch, run commands, inventory, Session Manager | **Systems Manager** |
| Prebuilt app platform for web apps, less ops than raw EC2 | **Elastic Beanstalk** |
| Managed queues of batch jobs, Spot-friendly compute fleets | **AWS Batch** |
| GraphQL API / managed realtime app backend | **AppSync** |
| Managed ActiveMQ/RabbitMQ migration | **Amazon MQ** |
| SaaS data transfer/integration flows | **AppFlow** |
| Automatically rotate DB credentials / API keys | **Secrets Manager** |
| Store config + secrets cheaply, no auto-rotation needed | **SSM Parameter Store** |
| Issue/manage free public TLS certificates | **ACM** |
| Encryption key management, envelope encryption | **KMS** (HSM-dedicated → **CloudHSM**) |
| Centrally apply guardrails across accounts (allow/deny) | **Organizations SCPs** |
| Set up a multi-account landing zone with best practices | **Control Tower** |
| Federate corporate identities / SSO to AWS | **IAM Identity Center** |
| Grant temporary cross-account/role credentials | **STS (AssumeRole)** |
| Web/mobile app user sign-up & sign-in (user directory) | **Cognito User Pools** |
| Exchange tokens for temporary AWS creds (app users) | **Cognito Identity Pools** |
| Orchestrate multi-step workflows / state machine | **Step Functions** |
| Cache content at the edge / CDN / static website + HTTPS | **CloudFront** |
| Non-HTTP (TCP/UDP), static anycast IPs, fast Region failover | **Global Accelerator** |
| Lowest-latency Region routing via DNS | **Route 53 Latency policy** |
| Active-passive DR via DNS | **Route 53 Failover policy** |
| Data residency / geo-block by country via DNS | **Route 53 Geolocation policy** |
| Run a query of huge dataset only occasionally, no infra | **Athena** |
| Shared Linux file system (NFS) across instances/AZs | **EFS** |
| Windows file shares (SMB) + Active Directory | **FSx for Windows** |
| High-performance compute (HPC/ML) file system over S3 | **FSx for Lustre** |
| Recommendations to cut cost / cost analysis | **Cost Explorer / Trusted Advisor / Compute Optimizer** |
| Cheapest interruptible compute (up to 90% off) | **EC2 Spot / Fargate Spot** |
| Process up to 15 minutes, event-driven, scale to zero | **Lambda** |

---

## 2. Multi-AZ vs Read Replica (RDS / Aurora)

| | **Multi-AZ** | **Read Replica** |
|---|---|---|
| Purpose | **High availability / failover** | **Scale reads** |
| Replication | Synchronous | Asynchronous |
| Readable? | ❌ Classic RDS Multi-AZ instance standby is not readable; Multi-AZ DB cluster has readable standbys | ✅ Serves read traffic |
| Failover | Automatic, ~60–120s, same endpoint | Manual promotion (then own endpoint) |
| Cross-Region? | No (standby is same Region) | ✅ Yes (also serves DR) |
| Cost | 2× (standby idle) | Per replica |

✅ "Survive AZ failure / automatic failover" → **Multi-AZ**. "Offload reporting / read-heavy" → **Read Replica**. If the question explicitly says **Multi-AZ DB cluster**, readable standbys are available; classic Multi-AZ instance questions still mean HA, not read scaling. "DR to another Region" → **cross-Region Read Replica**.

---

## 3. Security Group vs NACL

| | **Security Group** | **Network ACL** |
|---|---|---|
| Level | Instance / ENI | Subnet |
| State | **Stateful** (return traffic auto-allowed) | **Stateless** (must allow return explicitly) |
| Rules | Allow only | Allow **and** Deny |
| Evaluation | All rules evaluated | Rules in **number order**, first match wins |
| Default | Deny all inbound, allow all outbound | Default NACL allows all; custom denies all |

⚠️ To **block a specific IP**, you need a **NACL** (SGs can't deny). Stateful vs stateless is the most-tested distinction.

---

## 4. CloudTrail vs CloudWatch vs Config

| | **CloudTrail** | **CloudWatch** | **AWS Config** |
|---|---|---|---|
| Answers | **Who** made which **API call**, when | **How** is it performing (metrics/logs) | **What** is the configuration & did it drift |
| Data | API/management events | Metrics, logs, alarms | Resource config snapshots + history |
| Use case | Audit, security forensics, compliance trail | Monitoring, alarming, dashboards | Compliance rules, change tracking |
| Keyword | "audit", "who deleted", "API history" | "alarm", "CPU > 80%", "metric", "log" | "compliant", "config change", "drift" |

💡 "Who did it" → CloudTrail. "How is it doing" → CloudWatch. "Is it configured correctly / what changed" → Config.

---

## 5. SQS vs SNS vs Kinesis

| | **SQS** | **SNS** | **Kinesis Data Streams** |
|---|---|---|---|
| Model | Queue (pull) | Pub/sub (push) | Stream (pull, ordered) |
| Consumers | One logical consumer | Many subscribers | Many, each reads independently |
| Retention | Up to 14 days | Not stored (push then gone) | 1–365 days, replayable |
| Ordering | FIFO option | FIFO topic option | Per-shard ordering |
| Replay | ❌ (deleted on consume) | ❌ | ✅ |
| Typical use | Decouple, buffer work | Fan-out notifications | Real-time analytics, IoT, clickstream |

---

## 6. S3 Storage Classes

| Class | Use case | Retrieval | Min duration |
|---|---|---|---|
| **Standard** | Frequent access, hot data | ms | — |
| **Intelligent-Tiering** | Unknown/changing access patterns | ms | — |
| **Standard-IA** | Infrequent, rapid when needed | ms | 30 days |
| **One Zone-IA** | Infrequent, re-creatable, single AZ | ms | 30 days |
| **Glacier Instant Retrieval** | Archive, occasional instant access | ms | 90 days |
| **Glacier Flexible Retrieval** | Archive, retrieve minutes–hours | min–hrs | 90 days |
| **Glacier Deep Archive** | Coldest, cheapest, retrieve in hours | hours | 180 days |

✅ "Unknown / changing access pattern, no retrieval fees" → **Intelligent-Tiering**. "Re-creatable, lowest cost, OK in one AZ" → **One Zone-IA**. "Cheapest long-term archive, 12-hr retrieval OK" → **Deep Archive**.

---

## 7. EC2 Pricing Models

| Model | Discount | Commitment | Best for |
|---|---|---|---|
| **On-Demand** | — | None | Spiky, short-term, unpredictable |
| **Savings Plans** | up to ~72% | 1 or 3 yr ($/hr) | Steady usage, flexible across families |
| **Reserved (RI)** | up to ~72% | 1 or 3 yr (instance) | Steady, known instance type |
| **Spot** | up to ~90% | None (interruptible) | Fault-tolerant, batch, stateless |
| **Dedicated Host** | — | Optional | BYOL licensing, compliance, isolation |

⚠️ Spot can be reclaimed with a **2-minute** warning — only for interruption-tolerant workloads.

---

## 8. VPC Endpoint Types

| | **Gateway Endpoint** | **Interface Endpoint (PrivateLink)** | **Gateway Load Balancer Endpoint** |
|---|---|---|---|
| Services | **S3 and DynamoDB only** | Most AWS services + your own/3rd-party | GWLB endpoint services for appliances |
| Mechanism | Route table entry | ENI with private IP in your subnet | Endpoint ENI + route table target |
| Security group | No | Yes, on endpoint ENI | Not like interface endpoints; route/NACL/appliance controls |
| Cost | Free | Hourly + per-GB | Hourly + per-GB |
| Cross-Region/on-prem | No direct on-prem/TGW/peered access | Reachable via DX/VPN/peering with DNS/routing design | Inspection path |

💡 If the question is S3 or DynamoDB and says "free / no extra cost" → **Gateway Endpoint**. Anything else (or "private IP / on-prem access") → **Interface Endpoint**. Appliance inspection path → **Gateway Load Balancer Endpoint**.

---

## 9. CloudFront vs Global Accelerator

| | **CloudFront** | **Global Accelerator** |
|---|---|---|
| Layer / protocol | HTTP/HTTPS (caches content) | TCP/UDP (no caching) |
| Static IPs | No (uses DNS) | ✅ 2 static anycast IPs |
| Caches content? | ✅ Yes | ❌ No |
| Failover | Origin failover | Instant cross-Region failover |
| Best for | CDN, websites, streaming, WAF at edge | Non-HTTP apps, gaming, fixed IPs, fast Region failover |

---

## 10. Limits & Numbers to Memorize

| Limit | Value |
|---|---|
| Lambda max execution time | **15 minutes** |
| Lambda /tmp storage | 512 MB – 10 GB |
| Lambda memory | 128 MB – 10,240 MB |
| Lambda deployment package (zipped, direct) | 50 MB (250 MB unzipped) |
| SQS max message size | **256 KB** (use S3 pointer for larger via Extended Client) |
| SQS message retention | up to **14 days** (default 4) |
| SQS visibility timeout max | 12 hours |
| SQS Standard delivery | at-least-once, best-effort order |
| SQS FIFO throughput | Default FIFO: 300 API calls/s per action, 3,000 messages/s with batching; high-throughput FIFO can scale much higher with many message groups |
| S3 max object size | **5 TB** (single PUT max 5 GB; multipart > 100 MB) |
| S3 durability | 11 nines (99.999999999%) |
| Kinesis Data Streams retention | 24 hr default, up to 365 days |
| Kinesis shard throughput | 1 MB/s or 1,000 records/s in; 2 MB/s out |
| EBS gp3 max IOPS / throughput | 16,000 IOPS / 1,000 MB/s |
| VPC peering | **non-transitive**; no overlapping CIDRs |
| Security group rules | Allow-only, stateful |
| Default VPCs / Region | 5 (soft limit) |
| EIPs per Region | 5 (soft limit) |
| RDS Multi-AZ failover | ~60–120 seconds |
| DynamoDB item size | **400 KB** max |
| EFS | scales to petabytes, thousands of clients |
| IAM | global service (not Region-scoped) |
| EBS volume | single AZ; can't span AZs |

⚠️ The most-tested limits: **Lambda 15 min**, **SQS 256 KB**, **S3 object 5 TB**, **DynamoDB item 400 KB**, **VPC peering non-transitive**.

---

## 11. Fast tie-breakers

| Confusion | Tie-breaker |
|---|---|
| EFS vs FSx | Linux/NFS → EFS; Windows/SMB → FSx for Windows; HPC → FSx for Lustre |
| SQS vs Kinesis | Need replay/multiple readers/ordered stream → Kinesis; else SQS |
| SNS vs EventBridge | Content-based routing / SaaS / schedule → EventBridge; simple fan-out → SNS |
| ALB vs NLB | HTTP routing → ALB; static IP / TCP-UDP / extreme perf → NLB |
| RDS vs Aurora | Specific engine/lowest cost → RDS; max perf/HA AWS-native → Aurora |
| Secrets Manager vs Parameter Store | Auto-rotation → Secrets Manager; cheap config → Parameter Store |
| GuardDuty vs Inspector vs Macie | Threats from logs → GuardDuty; CVEs on resources → Inspector; PII in S3 → Macie |
| Shield vs WAF | DDoS (L3/4) → Shield; L7 SQLi/XSS/rate-limit → WAF |
| Snowball vs DataSync | Older exam: offline/no-bandwidth → Snowball; current new-customer design: DataSync, Data Transfer Terminal, or partners |

---

**Next**: [Practical: VPC with Public & Private Subnets](../18_practical_examples/01_vpc_public_private_subnets.md)
