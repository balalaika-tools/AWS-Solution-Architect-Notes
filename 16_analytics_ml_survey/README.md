# Analytics & AI/ML — Survey

> **Survey / recognition-level only.** The SAA-C03 exam tests these services at the "match the use case to the right managed service" level — *not* deep configuration. These two files are deliberately shallow: a tight paragraph and a table row per service, plus keyword→service cheat sheets. Don't study these to operate the services; study them to recognize the trigger word in a question stem.

[![AWS](https://img.shields.io/badge/AWS-Analytics%20%26%20AI%2FML-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/)
[![Athena](https://img.shields.io/badge/Athena-Serverless%20SQL-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/athena/)
[![SageMaker](https://img.shields.io/badge/SageMaker-ML%20Platform-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/sagemaker/)
[![Bedrock](https://img.shields.io/badge/Bedrock-GenAI-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/bedrock/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_analytics_services.md](01_analytics_services.md) | Analytics | Athena, Glue (+ DataBrew), Redshift (+ Spectrum), EMR, OpenSearch, Amazon Quick Sight / QuickSight, MSK, Managed Service for Apache Flink, Lake Formation, Data Pipeline. Athena vs Redshift vs EMR decision blurb + S3 data-lake pattern. |
| [02_ml_ai_services.md](02_ml_ai_services.md) | AI/ML | Rekognition, Textract, Comprehend, Transcribe, Polly, Translate, Lex, Kendra, Personalize, Forecast, Fraud Detector, SageMaker, Bedrock. Keyword→service matching. |

---

## Reading Order

1. **Analytics services** — the data-processing and querying family (S3 + Athena/Glue/Redshift is the most-tested combo).
2. **AI/ML services** — the managed AI APIs; the exam loves "which service for *this* use case" matching.

---

## Prerequisites

- [Storage](../05_storage/README.md) — S3 is the substrate for nearly every analytics service here.
- [Messaging & Events](../11_messaging/README.md) — Kinesis (the streaming ingestion path into analytics) lives there; this section only points to it.

---

**Next**: [01_analytics_services.md — Analytics Services](01_analytics_services.md)
