# Encryption & Security Services

> The data-protection and threat-detection layer of the exam. First the **cryptography prerequisites** (encryption at rest vs in transit, symmetric vs asymmetric, envelope encryption), then **KMS** and **CloudHSM** for key management, **Secrets Manager / Parameter Store / ACM** for secrets and certificates, and finally the **detective & protective** services (WAF, Shield, GuardDuty, Inspector, Macie, Detective, Security Hub). Every note starts from the underlying concept before the AWS service.

[![AWS](https://img.shields.io/badge/AWS-KMS%20%7C%20WAF%20%7C%20GuardDuty-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/security/)
[![Exam](https://img.shields.io/badge/Exam-SAA--C03-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_encryption_and_kms.md](01_encryption_and_kms.md) | KMS & Encryption | Encryption at rest vs in transit, symmetric vs asymmetric keys, **envelope encryption** (DEK + KEK) with diagram, then **KMS**: key types (AWS-managed / customer-managed / AWS-owned), key policies vs IAM, grants, rotation, multi-region keys, the `GenerateDataKey` flow, and **KMS vs CloudHSM**. |
| [02_secrets_manager_and_acm.md](02_secrets_manager_and_acm.md) | Secrets & Certificates | **Secrets Manager** (rotation via Lambda, RDS integration, versioning) **vs SSM Parameter Store** (SecureString, free, no auto-rotation) comparison, then **ACM**: free public TLS certs, auto-renewal, the `us-east-1` CloudFront rule, regional certs for ALB, and a Private CA note. |
| [03_threat_detection_services.md](03_threat_detection_services.md) | Threat Detection & Protection | **WAF** (L7), **Shield** Standard vs Advanced, **GuardDuty** (anomaly detection), **Inspector** (vuln scanning), **Macie** (S3 PII discovery), **Detective** (root-cause), **Security Hub** (aggregation), Firewall Manager. Includes the **"which service for which problem"** picker table. |

---

## Reading Order

1. **Encryption & KMS** — every other security service ultimately protects data that KMS encrypts. Start with the cryptography fundamentals, then KMS.
2. **Secrets Manager & ACM** — once keys are understood, see how AWS manages the *secrets* and *certificates* that ride on top of that key material.
3. **Threat Detection Services** — finishes the section with the detective and protective services that watch the account for attacks and misconfiguration.

---

## Prerequisites

- [IAM & Identity](../02_iam/README.md) — KMS key policies, grants, and every security service depend on IAM principals and policy evaluation.
- [Networking](../03_networking/README.md) — WAF/Shield sit in front of ALB/CloudFront; GuardDuty reads VPC Flow Logs.
- [Storage](../05_storage/README.md) and [Databases](../06_databases/README.md) — S3, EBS, and RDS are the main consumers of KMS encryption and the targets of Macie/Inspector.
