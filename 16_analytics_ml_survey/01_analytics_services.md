# Analytics Services — Survey

> **Who this is for**: Engineers prepping for SAA-C03 who need **recognition-level** coverage of the AWS analytics family. The exam rarely asks you to *configure* these — it asks you to *pick the right one* from a use-case sentence. Learn the keyword trigger for each. This is a survey, not a deep dive.

> **Survey scope**: One short paragraph + one table row per service. If you can map a question stem ("run SQL directly on S3", "managed Hadoop", "petabyte data warehouse") to the right service, you're exam-ready.

---

## 1. Query & Serverless SQL

**Amazon Athena** — *Serverless, interactive SQL query engine (built on Presto/Trino) that reads data directly in S3.* No infrastructure, no loading: point it at S3, define a schema, query with standard SQL. Pay per TB scanned. Use for ad-hoc querying of logs (VPC Flow Logs, CloudTrail, ALB/ELB access logs), CSV/JSON/Parquet in a data lake, and one-off analysis. Cost drops dramatically when data is **partitioned and stored in columnar Parquet/ORC**.
> 💡 **Exam clue**: "query data **in S3** with SQL, **no servers**, no ETL, pay per query" → **Athena**. Pairs with Amazon Quick Sight for visualization and Glue for the schema.

**AWS Glue** — *Serverless ETL service + a central metadata catalog.* Two parts the exam cares about: (1) **Glue ETL** runs serverless Spark jobs to extract/transform/load data; (2) the **Glue Data Catalog** is a central schema/metadata store, and **crawlers** auto-discover schemas in S3 and populate it. Athena, Redshift Spectrum, and EMR all read the Glue Catalog. Use it to clean/transform data and to give Athena a table definition.
> 💡 **Exam clue**: "**serverless ETL**", "**Data Catalog**", "**crawler** to infer schema" → **Glue**.

**AWS Glue DataBrew** — *Visual, no-code data preparation* for analysts. Point-and-click to clean and normalize data (250+ built-in transforms) without writing Spark. Use when the stem stresses "**no code / visual data prep** for analysts/data scientists."
> 💡 **Exam clue**: "**visual / no-code data preparation**" → **DataBrew** (vs Glue ETL = code-based).

---

## 2. Warehousing & Big Data Processing

**Amazon Redshift** — *Petabyte-scale, columnar data warehouse for OLAP / analytics and complex joins.* A provisioned cluster (or Redshift Serverless) optimized for aggregations and BI reporting over structured data loaded into it. **Redshift Spectrum** lets you query data sitting in S3 *without loading it* — extending the warehouse to the data lake. Use for a dedicated enterprise data warehouse, recurring complex BI queries, and dashboards needing fast joins on huge tables.
> 💡 **Exam clue**: "**data warehouse**", "**OLAP**", "petabyte-scale analytics", "complex joins / BI reporting" → **Redshift**. "Query S3 from the warehouse without loading" → **Redshift Spectrum**.

**Amazon EMR** (Elastic MapReduce) — *Managed cluster platform for big-data frameworks: Hadoop, Spark, Hive, Presto, HBase, Flink.* You run massive distributed processing (TB–PB) on EC2/EKS clusters, with control over the framework and tuning. Use when the workload is explicitly big-data framework-based, needs custom Spark/Hadoop jobs, or machine-learning preprocessing at scale.
> 💡 **Exam clue**: "**managed Hadoop / Spark**", "big data **cluster**", "Hive/Presto/HBase" → **EMR**.

---

## 3. Search, BI & Streaming Ingestion

**Amazon OpenSearch Service** (formerly Elasticsearch Service) — *Managed search and log/analytics engine — the "ELK" stack as a service* (OpenSearch + Dashboards/Kibana). Use for full-text search, real-time log analytics, operational dashboards, and observability over large volumes of semi-structured data. Common pattern: Kinesis Data Streams or Data Firehose → OpenSearch → Dashboards.
> 💡 **Exam clue**: "**search**", "**log analytics**", "**ELK / Kibana**", "near-real-time log dashboards" → **OpenSearch Service**.

**Amazon Quick Sight** (formerly **Amazon QuickSight**, now a core BI component of Amazon Quick Suite) — *Serverless business-intelligence (BI) dashboards and visualizations.* Connects to Athena, Redshift, RDS, S3, etc. to build interactive dashboards; **SPICE** is its fast in-memory engine. Existing QuickSight APIs and integrations continue to work. Use whenever the requirement is "**dashboards / visualize data / BI reports** for business users."
> 💡 **Exam clue**: "**BI dashboards**", "**visualize**", "interactive reports for business users", "SPICE" → **Amazon Quick Sight**.

**Amazon Kinesis Data Streams + Amazon Data Firehose** — *Real-time streaming ingestion plus near-real-time delivery.* Data Streams is the replayable stream for custom consumers; **Amazon Data Firehose** (formerly Kinesis Data Firehose) is the easiest way to land streaming data into S3/Redshift/OpenSearch without custom consumers. Covered in detail in the Messaging section — only the pointer belongs here.
> 💡 **Exam clue**: "**real-time streaming**", "clickstream", "replayable stream" → **Kinesis Data Streams**. "Ingest then deliver to S3/Redshift/OpenSearch with no code" → **Amazon Data Firehose**. Full notes: **[../11_messaging/05_kinesis.md](../11_messaging/05_kinesis.md)**.

**Amazon MSK** (Managed Streaming for Apache Kafka) — *Fully managed Apache Kafka.* AWS runs the Kafka brokers, Zookeeper, and storage; you keep the open-source Kafka API so existing producers/consumers and tooling work unchanged. **MSK Serverless** removes capacity planning. Use it specifically when the requirement names **Kafka** or the team already has Kafka apps/tooling to migrate. Otherwise Kinesis is the AWS-native default (no servers, tighter AWS integration).
> 💡 **Exam clue**: the word "**Kafka**" → **MSK** (managed Kafka). "Kafka but no capacity to manage" → **MSK Serverless**.

**Amazon Managed Service for Apache Flink** (formerly Kinesis Data Analytics for Apache Flink) — *Serverless real-time stream **processing** with Apache Flink.* Continuously reads from a stream (Kinesis Data Streams, MSK), runs Flink jobs in SQL/Java/Python/Scala to filter, aggregate, window, and enrich events, then writes results onward (S3, OpenSearch, another stream). Distinguish it from ingestion: Kinesis/MSK *move* the stream, Flink *analyzes it in flight*. Older Kinesis Data Analytics SQL applications are discontinued; use Flink for current designs. Use for real-time metrics, anomaly detection, sliding-window aggregations, and ETL on streaming data.
> 💡 **Exam clue**: "**real-time / streaming analytics**", "**Apache Flink**", "windowed aggregations on a live stream", "process data **as it arrives**" → **Managed Service for Apache Flink**.

---

## 4. Data-Lake Governance & Legacy

**AWS Lake Formation** — *Build and govern a secure S3 data lake.* Sits on top of Glue to centralize ingestion, cataloging, and — most importantly for the exam — **fine-grained, centralized permissions** (table/column/row-level access) across analytics services. Use when the stem stresses **central security/governance** for a data lake shared by many teams.
> 💡 **Exam clue**: "**data lake** with **fine-grained / centralized permissions / governance**" → **Lake Formation**.

**AWS Data Pipeline** — *Legacy orchestration service* for moving and transforming data on a schedule. It is in maintenance mode, has no planned new features or Region expansion, and AWS recommends migrating typical workloads to Glue, Step Functions, or Amazon MWAA. Recognize the name; modern designs should not start here.
> ⚠️ **Exam clue**: appears as an older/legacy option. If a newer Glue/Step Functions/MWAA answer fits, prefer it. Mostly recognition-only.

---

## 5. Summary Table

| Service | What it is | When to use | Exam keyword |
|---|---|---|---|
| **Athena** | Serverless SQL on S3 (Presto/Trino) | Ad-hoc SQL on S3 data/logs, no infra | "query S3 with SQL, serverless, pay per query" |
| **Glue** | Serverless ETL + Data Catalog + crawlers | Transform data, central schema catalog | "serverless ETL", "Data Catalog", "crawler" |
| **Glue DataBrew** | Visual/no-code data prep | Analysts cleaning data without code | "visual / no-code data preparation" |
| **Redshift** | Columnar data warehouse (OLAP) | Petabyte BI, complex joins, dashboards | "data warehouse", "OLAP", "petabyte analytics" |
| **Redshift Spectrum** | Query S3 from Redshift | Extend warehouse to data lake, no load | "query S3 without loading into warehouse" |
| **EMR** | Managed Hadoop/Spark/Hive/Presto | Big-data framework jobs at scale | "managed Hadoop / Spark", "big data cluster" |
| **OpenSearch Service** | Managed search + log analytics (ELK) | Full-text search, log dashboards | "search", "log analytics", "Kibana/ELK" |
| **Amazon Quick Sight** | Serverless BI dashboards | Visualize data for business users | "BI dashboards", "visualize", "SPICE" |
| **Kinesis Data Streams / Data Firehose** | Replayable streams / managed delivery | Stream/clickstream → S3/Redshift/OpenSearch | "real-time streaming" / "deliver stream to S3" |
| **MSK** | Managed Apache Kafka (+ Serverless) | Kafka workloads / migrations | "Kafka" |
| **Managed Service for Apache Flink** | Serverless real-time stream processing (Flink) | Analyze/aggregate a live stream in flight | "real-time analytics", "Apache Flink", "windowed" |
| **Lake Formation** | Data-lake build + governance | Central fine-grained S3 lake permissions | "data lake governance / fine-grained access" |
| **Data Pipeline** | Legacy data orchestration | Recognition only; prefer Glue/Step Functions/MWAA | "legacy" data movement |

---

## 6. Athena vs Redshift vs EMR (Decision Blurb)

All three "process big data," but they answer different questions:

```
 Data already in S3, want quick SQL, no infra, occasional/ad-hoc queries?
        └──►  ATHENA  (serverless, pay-per-scan, schema-on-read via Glue Catalog)

 Need a dedicated warehouse for repeated, complex BI queries / joins,
 fast performance, structured data loaded in?
        └──►  REDSHIFT  (provisioned/serverless columnar OLAP; Spectrum to reach S3)

 Need a custom big-data framework — Spark/Hadoop/Hive/HBase — at huge scale
 with control over the cluster and tuning?
        └──►  EMR  (managed cluster running the open-source frameworks)
```

> **Rule of thumb**: *Ad-hoc + serverless* → Athena. *Recurring BI warehouse* → Redshift. *Custom Spark/Hadoop pipelines* → EMR.

---

## 7. Mini-Pattern: Building a Data Lake on S3

The canonical AWS analytics architecture, and a common exam scenario:

```
 Sources ──► Amazon Data Firehose / Glue ETL ──►  S3 (data lake, Parquet)
                                                  │
                       Glue Crawler ─► Glue Data Catalog (schema/metadata)
                                                  │
              ┌───────────────┬───────────────────┼───────────────────┐
              ▼               ▼                    ▼                   ▼
           Athena      Redshift Spectrum         EMR              OpenSearch
         (SQL on S3)   (warehouse → S3)     (Spark/Hadoop)     (search/logs)
                                                  │
                                            Amazon Quick Sight  (BI dashboards)

           Lake Formation  ── governs permissions across all of the above
```

✅ **Storage = S3** (cheap, durable, decoupled), **catalog = Glue**, **query = Athena/Redshift/EMR**, **visualize = Amazon Quick Sight**, **governance = Lake Formation**. Storing data as **partitioned Parquet** cuts Athena/Redshift Spectrum scan cost.

---

## 8. SAP-C02 Analytics Architecture Decisions

Before the recognition table, SAP-C02 candidates should work through the
architecture decisions below. The individual service names are less important
than governance, workload shape, failure behavior, and cost.

### Govern a multi-account data lake

Keep S3 data, Glue catalog metadata, and authorization as separate layers.
**Lake Formation** grants database/table/column/row/cell access and can use LF-Tags
to express scalable policy. IAM still controls access to Lake Formation/Glue APIs,
KMS keys, and underlying infrastructure. Do not leave broad S3 permissions that
bypass the intended Lake Formation governance model.

For cross-account sharing, a producer registers/catalogs governed resources and
grants them to consumer accounts or organization units; AWS RAM participates in
resource sharing where required. The consumer creates links/views and grants its
local principals. Align the S3 bucket policy, KMS key policy/grants, Lake Formation
permissions, Glue Catalog resource policy, and IAM. An allow at one layer cannot
override a deny or missing permission at another.

Centralize CloudTrail, Lake Formation/Glue changes, S3 data events where justified,
and access findings. Separate data owner, catalog administrator, security, and
consumer responsibilities. Use the governed raw/curated zones, retention, object
ownership, and data-quality rules as code; sharing a catalog entry does not make
bad or unclassified data trustworthy.

### Choose the analytics engine from workload shape

| Requirement | Default direction | Trade-off to validate |
|-------------|-------------------|-----------------------|
| Ad-hoc SQL over partitioned S3 | Athena | Bytes scanned, file size/format, concurrency, and unpredictable query cost |
| Stable enterprise warehouse with predictable heavy BI | Redshift provisioned, with RA3/managed storage where it fits | Capacity planning and always-on cluster operations can buy predictable performance/cost |
| Intermittent or unpredictable warehouse demand | Redshift Serverless workgroup/namespace | Set usage limits and monitor base/peak capacity; serverless is not automatically cheapest for continuous load |
| Custom Spark/Hadoop ecosystem and deep cluster control | EMR on EC2 | Node lifecycle, patching, Spot strategy, bootstrap, and tuning |
| Spark jobs without cluster management | EMR Serverless | Application limits/start behavior and per-use cost |
| Run big-data jobs on an existing Kubernetes platform | EMR on EKS | Reuses EKS capacity/governance but inherits Kubernetes operations |

Use compression, columnar formats, partition pruning, compaction, statistics, and
workload management before scaling hardware. Test with representative data skew
and concurrency, not one small query.

### Search and streaming resilience

For OpenSearch, use dedicated cluster-manager capacity where the domain size and
criticality warrant it, data nodes across AZs, Multi-AZ with Standby where its
availability model fits, snapshots, and controlled index/Shard design. Too many
small shards waste heap; hot shards and unbalanced nodes break latency before
total storage fills. Encrypt at rest/in transit, restrict domain/network access,
and test snapshot restore or cross-cluster/Region recovery rather than assuming
replicas are backups.

Streaming architecture separates ingestion, processing, and delivery:

```
producers -> Kinesis Data Streams or MSK -> Flink/Lambda/consumers -> curated stores
                                 \-> Data Firehose -> S3/Redshift/OpenSearch
```

Size shards/partitions and consumers from records and bytes, choose a replay
retention window, handle duplicates/checkpoints/poison records, and alarm on
iterator lag. Data Firehose reduces delivery operations; it does not provide the
same custom consumer/replay model as a stream. Multi-Region usually requires a
second stream/cluster and an application or replication pipeline with explicit
ordering, duplicate, and RPO behavior.

### Encryption, recovery, and cost

Use KMS keys whose policies support the analytics service and every participating
account. Encrypt transport, S3 zones, catalogs/metadata where supported, streams,
warehouses/search domains, logs, and snapshots. A copied snapshot or shared table
is unusable in recovery if the target principal cannot use the key.

Define recovery per layer: catalog export/recreation, S3 versioning/replication,
Redshift snapshots and restore, EMR job/config recreation, OpenSearch snapshot or
replicated index, and replayable stream retention. Pre-provision quotas and test
data integrity plus dashboard/application reconnection.

Measure cost per TB ingested/stored/scanned and per successful report or stream
outcome. The biggest levers are file layout and scan reduction, warehouse duty
cycle, EMR Spot/graceful decommissioning, shard count/retention, cross-AZ/Region
transfer, KMS/log requests, and data duplicated for recovery. Preserve the SLO
and RPO when optimizing.

---

## 9. Recognition Table (Keyword → Service)

| Question stem keyword | Pick |
|---|---|
| "Query data **in S3** with SQL, **serverless**" | Athena |
| "Analyze **VPC Flow Logs / CloudTrail / ALB logs** with SQL" | Athena |
| "**Serverless ETL**" / "**crawler**" / "**Data Catalog**" | Glue |
| "**Visual / no-code** data prep" | Glue DataBrew |
| "**Data warehouse**" / "**OLAP**" / "petabyte BI" | Redshift |
| "Warehouse query of **S3 without loading**" | Redshift Spectrum |
| "Managed **Hadoop / Spark / Hive / Presto** cluster" | EMR |
| "**Search**" / "**log analytics**" / "**Kibana / ELK**" | OpenSearch Service |
| "**BI dashboards** / visualize for business users" | Amazon Quick Sight |
| "**Real-time streaming** ingestion" | Kinesis Data Streams |
| "Deliver streaming data to S3/Redshift/OpenSearch, no code" | Amazon Data Firehose |
| "**Apache Kafka**" | MSK |
| "**Real-time analytics / processing**", "**Apache Flink**", "windowed aggregation on a stream" | Managed Service for Apache Flink |
| "**Data lake** with **fine-grained / centralized permissions**" | Lake Formation |

> ⚠️ **Common trap**: Athena vs Redshift. If data is *already in S3* and queries are *ad-hoc/occasional* with *no infra to manage* → Athena. If it's a *persistent warehouse* for *repeated complex BI* → Redshift. The phrases "serverless" and "pay per query" point to Athena; "warehouse" and "cluster" point to Redshift.

> **Professional checkpoint**: for SAP-C02, also identify the producer/consumer
> account boundary, Lake Formation/Glue/S3/KMS permission path, engine duty cycle,
> streaming replay semantics, AZ/Region recovery method, quota, and unit-cost KPI.

---

**Next**: [02_ml_ai_services.md — AI/ML Services](02_ml_ai_services.md)
