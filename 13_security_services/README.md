# Encryption & Security Services

> The data-protection and threat-detection layer of the exam. First the **cryptography prerequisites** (encryption at rest vs in transit, symmetric vs asymmetric, envelope encryption), then **KMS** and **CloudHSM** for key management, **Secrets Manager / Parameter Store / ACM** for secrets and certificates, and finally the **detective & protective** services (WAF, Shield, GuardDuty, Inspector, Macie, Detective, Security Hub). Every note starts from the underlying concept before the AWS service.

[![AWS](https://img.shields.io/badge/AWS-KMS%20%7C%20WAF%20%7C%20GuardDuty-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/security/)
[![Exam](https://img.shields.io/badge/Exam-SAA--C03-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_encryption_and_kms.md](01_encryption_and_kms.md) | KMS & Encryption | Encryption and envelope encryption, KMS authorization/rotation/multi-Region behavior, workload-owned vs centralized enterprise key architecture, cross-account service use, imported material, CloudHSM/custom key stores, quotas, recovery, and audit design. |
| [02_secrets_manager_and_acm.md](02_secrets_manager_and_acm.md) | Secrets & Certificates | Secrets Manager vs Parameter Store, multi-account policies, replica/failover and rotation recovery, caching/PrivateLink, current ACM export and renewal behavior, Regional/account certificate deployment, and a governed Private CA hierarchy. |
| [03_threat_detection_services.md](03_threat_detection_services.md) | Threat Detection & Protection | WAF, current Shield Standard/Advanced distinctions, GuardDuty, Inspector, Macie, Detective, Security Hub, Network Firewall, and Firewall Manager, followed by delegated administration, Regional enrollment, finding aggregation, and automated remediation patterns. |

---

## Reading Order

1. **Encryption & KMS** — every other security service ultimately protects data that KMS encrypts. Start with the cryptography fundamentals, then KMS.
2. **Secrets Manager & ACM** — once keys are understood, see how AWS manages the *secrets* and *certificates* that ride on top of that key material.
3. **Threat Detection Services** — finishes the section with the detective and protective services that watch the account for attacks and misconfiguration.

---

## SAP-C02 Professional Study Path

The professional-level exam expects several services to operate as one multi-account control
system. After the three-file reading order above, work through these decisions in sequence:

1. **Set the organization boundary.** Put workloads in accounts grouped by OU, delegate daily
   security administration away from the Organizations management account, restrict unapproved
   Regions/actions with SCPs, and use Firewall Manager and central-configuration policies to
   enroll new and existing accounts. Remember that delegated administration still needs a
   Region-by-Region deployment strategy.
2. **Design centralized evidence and findings.** Deliver an organization CloudTrail and required
   service/network logs to a protected log-archive account. Aggregate normalized Security Hub
   findings into a home Region while retaining the source findings and logs needed for forensics.
   Decide retention from data classification, regulatory requirements, investigation time, and
   recovery needs rather than from a console's default history window.
3. **Build controlled remediation.** Route high-confidence finding changes with EventBridge to
   Step Functions or Systems Manager Automation. Run cross-account under least-privilege roles,
   preserve evidence first, use idempotency/concurrency/error controls, require approval for
   destructive steps, and verify both containment and recovery.
4. **Classify and retain data.** Use Macie for sensitive S3 discovery, Config and IAM Access
   Analyzer for configuration/access drift, KMS and resource policies for authorization, and S3
   lifecycle/Object Lock where retention requires it. Classification should change encryption,
   access, monitoring, incident priority, and disposal behavior.
5. **Manage vulnerability and patch risk.** Use Inspector for EC2/ECR/Lambda vulnerability
   coverage, Systems Manager patch policies or immutable image replacement for remediation, and
   Config/Security Hub to measure coverage. A patch program includes exception expiry, rollout
   rings, maintenance windows, rollback/replacement, and evidence—not only scanning.
6. **Choose cross-account encryption deliberately.** Default to workload-owned KMS keys and
   centralized governance. Use a central key account only when separation of duties or intentional
   sharing justifies the dependency and the target service supports an external key. Align the key
   policy, caller IAM policy, grants/service role, encryption context, Region, quotas, audit trail,
   deletion recovery, and DR replicas.
7. **Rehearse incident response.** Join GuardDuty/Detective/Security Hub findings to CloudTrail,
   network/WAF logs, KMS history, and resource state. Practice reversible quarantine, credential
   and secret rotation, key-deletion cancellation, forensic preservation, clean rebuild, Regional
   failover, and final restoration with a documented break-glass path.

A strong SAP-C02 answer usually combines **preventive guardrails + detective coverage + an
auditable response/recovery path** and states the account, Region, key, log, and failure
dependencies explicitly.

---

## Prerequisites

- [IAM & Identity](../02_iam/README.md) — KMS key policies, grants, and every security service depend on IAM principals and policy evaluation.
- [Networking](../03_networking/README.md) — WAF/Shield sit in front of ALB/CloudFront; GuardDuty reads VPC Flow Logs.
- [Storage](../05_storage/README.md) and [Databases](../06_databases/README.md) — S3, EBS, and RDS are the main consumers of KMS encryption and the targets of Macie/Inspector.
