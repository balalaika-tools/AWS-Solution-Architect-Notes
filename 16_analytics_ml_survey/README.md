# Analytics Architecture & AI/ML Survey

> The SAA-C03 path uses recognition-level service matching. For SAP-C02,
> analytics requires architecture-level decisions about governance, sharing,
> deployment models, streaming, encryption, recovery, performance, and cost.
> Most AI APIs remain recognition-only, with an additional governed Bedrock and
> human-approval scenario for current professional-exam emerging topics.

[![AWS](https://img.shields.io/badge/AWS-Analytics%20%26%20AI%2FML-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/)
[![Athena](https://img.shields.io/badge/Athena-Serverless%20SQL-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/athena/)
[![SageMaker](https://img.shields.io/badge/SageMaker-ML%20Platform-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/sagemaker/)
[![Bedrock](https://img.shields.io/badge/Bedrock-GenAI-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/bedrock/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_analytics_services.md](01_analytics_services.md) | Analytics | Athena, Glue (+ DataBrew), Redshift (+ Spectrum), EMR, OpenSearch, Amazon Quick Sight, Kinesis Data Streams, Amazon Data Firehose, MSK, Managed Service for Apache Flink, Lake Formation, legacy Data Pipeline. Athena vs Redshift vs EMR decision blurb + S3 data-lake pattern. |
| [02_ml_ai_services.md](02_ml_ai_services.md) | AI/ML | Rekognition, Textract, Comprehend, Transcribe, Polly, Translate, Lex, Kendra, Personalize, legacy Forecast/Fraud Detector, SageMaker, Bedrock. Keyword→service matching. |

---

## Reading Order

1. **Analytics services** — the data-processing and querying family (S3 + Athena/Glue/Redshift is the most-tested combo).
2. **AI/ML services** — the managed AI APIs; the exam loves "which service for *this* use case" matching.

## Depth by Certification Track

| Capability | SAA-C03 | SAP-C02 |
|------------|---------|---------|
| Athena, Glue, Redshift, EMR, OpenSearch, streams, Lake Formation | Recognize the service and basic data-lake pattern | Design cross-account governance, choose compute/deployment modes, plan streaming semantics, encryption, DR, and cost/performance |
| Rekognition, Textract, Comprehend, speech/language/search APIs | Recognition only | Recognition only unless a scenario explicitly makes one a workload dependency |
| SageMaker | Recognize custom-model platform | Compare only when modernization requires custom model ownership; full ML engineering is outside these notes |
| Bedrock | Recognize foundation-model use | Apply Guardrails, least-privilege model/data/tool access, logging/privacy controls, and human approval for high-impact agent actions |

Follow [Analytics architecture decisions](01_analytics_services.md#8-sap-c02-analytics-architecture-decisions)
after the base survey, then the [governed generative/agentic scenario](02_ml_ai_services.md#7-sap-c02-emerging-scenario-governed-generative-and-agentic-ai).
Do not promote every AI service to deep study; the professional depth here is
analytics platform architecture plus secure, controlled use of Bedrock.

---

## Prerequisites

- [Storage](../05_storage/README.md) — S3 is the substrate for nearly every analytics service here.
- [Messaging & Events](../11_messaging/README.md) — Kinesis (the streaming ingestion path into analytics) lives there; this section only points to it.

---

**Next**: [01_analytics_services.md — Analytics Services](01_analytics_services.md)
