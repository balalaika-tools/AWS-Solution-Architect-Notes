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

**Amazon OpenSearch Service** (formerly Elasticsearch Service) — *Managed search and log/analytics engine — the "ELK" stack as a service* (OpenSearch + Dashboards/Kibana). Use for full-text search, real-time log analytics, operational dashboards, and observability over large volumes of semi-structured data. Common pattern: Kinesis/Firehose → OpenSearch → Dashboards.
> 💡 **Exam clue**: "**search**", "**log analytics**", "**ELK / Kibana**", "near-real-time log dashboards" → **OpenSearch Service**.

**Amazon Quick Sight** (formerly commonly written **QuickSight**, now part of Amazon Quick/Quick Suite) — *Serverless business-intelligence (BI) dashboards and visualizations.* Connects to Athena, Redshift, RDS, S3, etc. to build interactive dashboards; **SPICE** is its fast in-memory engine. Existing QuickSight APIs and integrations continue to work. Use whenever the requirement is "**dashboards / visualize data / BI reports** for business users."
> 💡 **Exam clue**: "**BI dashboards**", "**visualize**", "interactive reports for business users", "SPICE" → **Amazon Quick Sight / QuickSight**.

**Amazon Kinesis** — *Real-time streaming ingestion and processing* (Data Streams, Firehose, Managed Flink). Firehose is the easiest way to land streaming data into S3/Redshift/OpenSearch. Covered in detail in the Messaging section — only the pointer belongs here.
> 💡 **Exam clue**: "**real-time streaming**", "clickstream", "ingest then deliver to S3/Redshift" → **Kinesis**. Full notes: **[../11_messaging/05_kinesis.md](../11_messaging/05_kinesis.md)**.

**Amazon MSK** (Managed Streaming for Apache Kafka) — *Fully managed Apache Kafka.* Use it specifically when the requirement names **Kafka** or the team already has Kafka apps/tooling to migrate. Otherwise Kinesis is the AWS-native default.
> 💡 **Exam clue**: the word "**Kafka**" → **MSK** (managed Kafka).

---

## 4. Data-Lake Governance & Legacy

**AWS Lake Formation** — *Build and govern a secure S3 data lake.* Sits on top of Glue to centralize ingestion, cataloging, and — most importantly for the exam — **fine-grained, centralized permissions** (table/column/row-level access) across analytics services. Use when the stem stresses **central security/governance** for a data lake shared by many teams.
> 💡 **Exam clue**: "**data lake** with **fine-grained / centralized permissions / governance**" → **Lake Formation**.

**AWS Data Pipeline** — *Legacy orchestration service* for moving and transforming data on a schedule. Largely superseded by Glue, Step Functions, and managed alternatives. Recognize the name; modern designs prefer Glue/Step Functions/MWAA.
> ⚠️ **Exam clue**: appears as an older/legacy option. If a newer Glue/Step Functions answer fits, prefer it. Mostly recognition-only.

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
| **Amazon Quick Sight / QuickSight** | Serverless BI dashboards | Visualize data for business users | "BI dashboards", "visualize", "SPICE" |
| **Kinesis** | Real-time streaming ingestion | Stream/clickstream → S3/Redshift/OpenSearch | "real-time streaming" (see messaging notes) |
| **MSK** | Managed Apache Kafka | Kafka workloads / migrations | "Kafka" |
| **Lake Formation** | Data-lake build + governance | Central fine-grained S3 lake permissions | "data lake governance / fine-grained access" |
| **Data Pipeline** | Legacy data orchestration | Recognition only; prefer Glue/Step Functions | "legacy" data movement |

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
 Sources ──► Kinesis Firehose / Glue ETL ──►  S3 (data lake, Parquet)
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

## 8. Key Exam Points (Keyword → Service)

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
| "**BI dashboards** / visualize for business users" | Amazon Quick Sight / QuickSight |
| "**Real-time streaming** ingestion" | Kinesis |
| "**Apache Kafka**" | MSK |
| "**Data lake** with **fine-grained / centralized permissions**" | Lake Formation |

> ⚠️ **Common trap**: Athena vs Redshift. If data is *already in S3* and queries are *ad-hoc/occasional* with *no infra to manage* → Athena. If it's a *persistent warehouse* for *repeated complex BI* → Redshift. The phrases "serverless" and "pay per query" point to Athena; "warehouse" and "cluster" point to Redshift.

---

**Next**: [02_ml_ai_services.md — AI/ML Services](02_ml_ai_services.md)
