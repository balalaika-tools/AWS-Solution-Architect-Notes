# Service Comparison Cheat Sheets

> **Who this is for**: Engineers doing final review for SAA-C03 or SAP-C02.
> The first half supports rapid service discrimination; the professional sections
> compare policy semantics, governance, operations, migration, connectivity, and
> capacity without treating volatile numbers as timeless facts.

> **Key insight**: Keyword triggers are useful for SAA-C03 recall. In SAP-C02,
> use them only to generate candidates, then decide from combined requirements
> and explain why each plausible alternative fails.

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
| Migrate databases (homogeneous & heterogeneous) | **DMS** (+ DMS Schema Conversion where supported, or SCT fallback) |
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

## 10. SAP-C02 Governance and Operations Comparisons

Before the limits section, use these professional-level comparisons. They differ
by operating model and policy semantics, not a single keyword.

### Organizations vs Control Tower

| | **AWS Organizations** | **AWS Control Tower** |
|--|-----------------------|-----------------------|
| Core job | Account hierarchy, consolidated billing, organization policies, trusted access | Opinionated landing-zone orchestration built on Organizations and other services |
| Gives you | OUs/accounts, SCPs/RCPs and other policy types, delegated administration | Shared accounts, controls, Account Factory, dashboards, drift visibility, baseline automation |
| Choose when | You need primitives or a highly custom governance implementation | You want a prescriptive multi-account baseline and governed account vending |
| Does not do | Build the full landing zone/account factory by itself | Remove need to design OUs, IAM, networking, logging, operations, exceptions, and costs |

### SCP vs permissions boundary vs resource policy

| Policy | Applies to | Can grant? | Best use |
|--------|------------|------------|----------|
| **SCP** | IAM principals in member accounts under its organization root/OU/account scope | No; maximum permission guardrail | Organization-wide deny/allow ceiling |
| **Permissions boundary** | One IAM user or role | No; maximum identity permission | Delegate role creation without allowing privilege beyond a boundary |
| **Resource policy** | Access to one resource, including supported cross-account principals | Yes, subject to full evaluation | Bucket/key/queue/role trust and other resource-scoped access |

An explicit deny wins. Cross-account access also needs a trusted principal and a
valid grant path; SCPs/boundaries/session policies can still restrict it. A role
trust policy is a resource policy that says who may assume the role, not what the
resulting session may do.

### StackSets vs Service Catalog

| | **CloudFormation StackSets** | **AWS Service Catalog** |
|--|------------------------------|-------------------------|
| Consumer | Central platform deploys stacks across accounts/Regions | End users provision approved products from shared portfolios |
| Purpose | Organization-wide baseline/application rollout | Governed self-service and product lifecycle |
| Control | Delegated admin, OU targets, concurrency/failure tolerance | Constraints, versions, access, launch roles |
| Common use | IAM roles, Config baseline, logging resources | Approved application/environment/database products |

They can combine: a StackSet distributes portfolios/constraints, while a Service
Catalog product uses CloudFormation underneath.

### Config vs CloudFormation drift

| | **AWS Config** | **CloudFormation drift detection** |
|--|----------------|------------------------------------|
| Scope | Supported resources across the account/aggregator, whether or not created by CloudFormation | Declared properties of resources in one stack/StackSet |
| Answers | What configuration changed, when, relationships, and whether a rule/conformance pack considers it compliant | Does actual stack resource configuration differ from its template expectation? |
| Remediation | Config rule + Systems Manager Automation or other workflow | Update/reconcile stack/template through deployment process |

### MGN vs DMS vs DataSync

| | **MGN** | **DMS (+ schema conversion)** | **DataSync** |
|--|---------|-----------------|--------------|
| Moves | Server block storage/whole machine into EC2 | Database rows/changes; DMS Schema Conversion or SCT fallback converts heterogeneous schema/code | Files/objects between storage systems/services |
| Low-downtime method | Continuous block replication then test/cutover launch | Full load plus CDC then validation/promotion | Repeated incremental transfers then final delta |
| Does not replace | Database/application-specific consistency testing | Application/server migration | Database CDC/schema conversion or bootable server migration |

### Direct Connect gateway and VIF chooser

| Component | Reach | Pick it when |
|-----------|-------|--------------|
| **Private VIF** | Private VPC address space through a virtual private gateway, or multiple VPCs through a Direct Connect gateway association | Private workload connectivity |
| **Transit VIF** | Transit Gateway(s) through a Direct Connect gateway | Hub-and-spoke connectivity to many VPCs/networks |
| **Public VIF** | Public AWS service prefixes using public IP/BGP requirements | Reach public AWS endpoints over Direct Connect, not private VPC CIDRs |
| **Direct Connect gateway** | Global association construct connecting VIFs to supported gateways | Connect locations to VPCs/TGWs across supported Regions; it is not a packet-inspection or transitive VPC router by itself |

Direct Connect is private transport, not encryption by default. Add MACsec where
supported or VPN over the connection when encryption is required, and design
redundant connections/locations plus BGP paths.

### Centralized vs distributed network/security

| Model | Strength | Cost/risk |
|-------|----------|-----------|
| **Centralized egress, DNS, inspection, endpoints** | Consistent control and fewer duplicated resources | Shared failure/operations domain, TGW/cross-AZ/processing charges, routing symmetry and team bottleneck |
| **Distributed per workload/AZ/Region** | Failure isolation, local ownership/path, simpler regional independence | More hourly resources, policy drift, duplicated tooling, fragmented visibility |

Choose per capability. Centralize policy and evidence even when data-plane
resources are distributed; use PrivateLink for one-service exposure instead of
granting routed network reachability.

---

## 11. Durable Limits and Volatile Values

Memorize durable semantics; verify quotas, prices, availability, and throughput
at design time. Many values below the old exam-review line changed as services
evolved.

### Durable architectural constraints

| Constraint | Why it matters |
|------------|----------------|
| Lambda invocation maximum is 15 minutes | Longer work must be split/orchestrated or moved to another compute service |
| DynamoDB item maximum is 400 KB | Large objects belong in S3 with a reference |
| VPC peering is non-transitive and rejects overlapping CIDRs | It cannot become an organization-scale routed hub |
| Security groups are stateful allow rules | Use other controls when an explicit network deny is required |
| An EBS volume belongs to one AZ | AZ recovery requires snapshot/copy/replicated application state, not reattachment across AZs |

### Verify instead of memorizing

Check current values and regional availability in [AWS Service Quotas](https://docs.aws.amazon.com/servicequotas/latest/userguide/intro.html),
the relevant service's official quotas page, and the [AWS pricing pages](https://aws.amazon.com/pricing/)
before design/cutover. This includes:

- account/Regional vCPU, ENI, VPC, Elastic IP, load balancer, API, and scaling quotas;
- SQS message size, retention, FIFO throughput, and in-flight limits;
- S3 object/request limits and storage/data-transfer/request prices;
- Kinesis shard/on-demand throughput and retention;
- Lambda concurrency, payload, package, and ephemeral-storage limits;
- EBS type performance, database replicas/failover behavior, service-specific
  timeout, and every quoted discount percentage.

Service Quotas shows defaults, applied values, adjustability, and increase
requests for integrated services. It does not prove scarce AZ capacity exists.
Pair quota checks with subnet/IP headroom, launch tests or reservations where
applicable, downstream quotas, and game-day scale/recovery tests.

When a practice question supplies an exact quota or price, use the value in that
question. For real designs and current-study flash cards, attach the official
source and a checked date. Treat exact throughput, count, storage size, discount,
and default quota as volatile unless the architectural constraint depends on it.

---

## 12. Fast tie-breakers

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
