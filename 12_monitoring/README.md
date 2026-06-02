# Observability & Governance

> The three AWS services the exam asks you to tell apart: **CloudWatch** (is it healthy? — metrics, logs, alarms), **CloudTrail** (who did what? — API audit log), and **AWS Config** (how is it configured / is it compliant? — resource state and change history). Each note starts from the underlying concept (observability, audit logging, configuration management) before the AWS service.

[![AWS](https://img.shields.io/badge/AWS-CloudWatch%20%7C%20CloudTrail%20%7C%20Config-FF9900.svg?logo=amazoncloudwatch&logoColor=white)](https://aws.amazon.com/cloudwatch/)
[![Exam](https://img.shields.io/badge/Exam-SAA--C03-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_cloudwatch.md](01_cloudwatch.md) | CloudWatch | Observability basics (metrics/logs/traces), then CloudWatch: namespaces & dimensions, standard vs custom metrics, basic vs detailed monitoring, alarms (states/actions), composite alarms, Logs (groups/streams, metric & subscription filters, Logs Insights), dashboards, EventBridge, the CloudWatch Agent, Synthetics canaries, and the X-Ray/ServiceLens pointer. |
| [02_cloudtrail.md](02_cloudtrail.md) | CloudTrail | Audit-logging concept, then CloudTrail: management vs data vs Insights events, 90-day event history vs trails to S3, multi-region & organization trails, log file integrity validation, integration with CloudWatch Logs and Athena, and what CloudTrail is *not*. |
| [03_config.md](03_config.md) | AWS Config | Configuration-management concept, then AWS Config: configuration items, the recorder, managed & custom rules, conformance packs, SSM auto-remediation, the configuration timeline, and multi-account aggregators. Includes the **CloudWatch vs CloudTrail vs Config** comparison table. |

---

## Reading Order

1. **CloudWatch** — the broadest service and the one every other AWS service emits to. Start here.
2. **CloudTrail** — once you know how CloudWatch surfaces *performance*, learn how CloudTrail records *API activity*.
3. **AWS Config** — closes the loop with *configuration state and compliance*, and ends with the three-way comparison that ties the section together.

---

## Governance Services to Recognize

| Service | Exam clue |
|---------|-----------|
| **CloudFormation** | Infrastructure as code, repeatable stacks, drift detection. |
| **Systems Manager** | Patch Manager, Run Command, Session Manager, inventory, Parameter Store. |
| **Service Catalog** | Let teams launch approved products/stacks with governance. |
| **Control Tower** | Create and govern a multi-account landing zone. |
| **Organizations** | Multi-account structure, consolidated billing, SCP guardrails. |
| **Trusted Advisor** | Best-practice checks for cost, security, fault tolerance, performance, service limits. |
| **Compute Optimizer** | Right-size EC2/EBS/Lambda/ECS based on utilization. |
| **Health Dashboard** | AWS service/account events that affect your resources. |
| **Service Quotas** | View/request quota increases and avoid limit-related scaling failures. |

---

## Prerequisites

- [IAM & Identity](../02_iam/README.md) — CloudTrail records the IAM principal behind every API call; Config and CloudWatch use service roles.
- [Messaging & Events](../11_messaging/README.md) — alarms and events fan out through **SNS** and **EventBridge**.
- [Storage](../05_storage/README.md) — CloudTrail trails and Config snapshots land in **S3**; Athena queries them in place.
