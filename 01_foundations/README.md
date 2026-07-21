# Foundations

> The mental models that frame everything else: what cloud computing *is*, where AWS physically runs, and who is responsible for what.

[![AWS](https://img.shields.io/badge/AWS-Solutions%20Architect-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)
[![Exam](https://img.shields.io/badge/Exam-SAA--C03-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_cloud_computing_fundamentals.md](01_cloud_computing_fundamentals.md) | Cloud Computing Fundamentals | IaaS/PaaS/SaaS, public/private/hybrid, CapEx vs OpEx, the six advantages, how AWS bills |
| [02_aws_global_infrastructure.md](02_aws_global_infrastructure.md) | AWS Global Infrastructure | Regions, AZs, edge locations, Local Zones, Wavelength, Outposts; choosing a Region; global vs regional vs zonal |
| [03_shared_responsibility_and_account_basics.md](03_shared_responsibility_and_account_basics.md) | Shared Responsibility & Account Basics | Security OF vs IN the cloud, root vs IAM, billing, Organizations and multi-account governance bridge, support plans, Trusted Advisor |

---

## Reading Order

1. **Cloud Computing Fundamentals** — the vocabulary (IaaS/PaaS/SaaS, OpEx) every later topic assumes.
2. **AWS Global Infrastructure** — where your resources physically live and why that drives availability and latency decisions.
3. **Shared Responsibility & Account Basics** — the security contract and account scaffolding before you touch IAM.

---

## SAP-C02 Readiness

These notes are reusable foundations for both associate- and professional-level architecture:
the service model, infrastructure fault boundaries, Region selection, shared responsibility, and
the AWS account boundary remain the same. They deliberately stop before the organizational
complexity expected in SAP-C02 scenarios.

For professional-level multi-account design, continue to
[Organizations, STS & Federation](../02_iam/04_organizations_sts_federation.md). That chapter
develops OU and SCP design, cross-account roles, IAM Identity Center, tag policies, and Control
Tower landing zones and account vending. The
[account-governance bridge](03_shared_responsibility_and_account_basics.md#4%EF%B8%8F%E2%83%A3-aws-organizations-and-the-multi-account-bridge)
in this section first explains where delegated administration and centralized logging/security
accounts fit in that model.

---

## Prerequisites

- None. This is the entry point — start here if you are new to AWS.
- A general sense of what a server and a data center are is helpful but not required.

**Next section**: [IAM & Identity](../02_iam/README.md)
