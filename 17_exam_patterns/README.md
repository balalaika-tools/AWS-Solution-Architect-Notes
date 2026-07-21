# Exam Patterns: SAA-C03 and SAP-C02 Final Review

> The capstone review section has two tracks. SAA-C03 uses compact service
> discriminators. SAP-C02 uses multi-step scenarios that combine organizational,
> security, operations, migration, performance, reliability, and cost constraints
> and require explaining why plausible alternatives fail.

[![AWS](https://img.shields.io/badge/AWS-Solutions%20Architect-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)
[![Exam](https://img.shields.io/badge/Exam-SAA--C03-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)
[![Exam](https://img.shields.io/badge/Exam-SAP--C02-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-professional/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_architecture_decision_guides.md](01_architecture_decision_guides.md) | Decision guides | Associate service selection plus SAP-C02 landing-zone, deployment/configuration/capacity, migration/modernization, and long-form constraint drills. |
| [02_service_comparison_cheatsheets.md](02_service_comparison_cheatsheets.md) | Comparison cheat sheets | Fast SAA comparisons plus SAP-C02 governance/operations comparisons and a durable-limit versus volatile-quota method. |

---

## Reading Order

1. **Architecture Decision Guides** — review base service choices, then work the organizational, deployment, migration, and long-form SAP-C02 sections without looking at the answer.
2. **Service Comparison Cheat Sheets** — use fast comparisons for SAA-C03; for SAP-C02, explain evaluation semantics, operating ownership, failure domain, rollback, quota, and cost rather than reciting one keyword.

---

## SAP-C02 Domain Map and Coverage Tracker

Current scored weights are **26% / 29% / 25% / 20%**. Use the task rows as a
final-review checklist; "covered" means you can solve a mixed-constraint scenario,
not that you recognize the service name.

| Domain / task | Weight | Primary review coverage |
|---------------|--------|-------------------------|
| **1. Design Solutions for Organizational Complexity** | **26%** | |
| 1.1 Architect network connectivity strategies | | [Networking](../03_networking/README.md), [hybrid networking](../14_hybrid_migration_dr/01_hybrid_networking.md), [connectivity guide](01_architecture_decision_guides.md#9-connectivity-vpc-peering-vs-transit-gateway-vs-privatelink-vs-vpn-vs-direct-connect) |
| 1.2 Prescribe security controls | | [IAM](../02_iam/README.md), [Security](../13_security_services/README.md), [policy comparison](02_service_comparison_cheatsheets.md#scp-vs-permissions-boundary-vs-resource-policy) |
| 1.3 Design reliable and resilient architectures | | [HA/scaling](../07_ha_scaling/README.md), [DR](../14_hybrid_migration_dr/04_disaster_recovery.md) |
| 1.4 Design a multi-account AWS environment | | [Organizations/STS](../02_iam/04_organizations_sts_federation.md), [landing-zone guide](01_architecture_decision_guides.md#13-multi-account-landing-zone-and-governance) |
| 1.5 Determine cost optimization and visibility strategies | | [Cost allocation/optimization](../15_cost_well_architected/01_cost_optimization.md) |
| **2. Design for New Solutions** | **29%** | |
| 2.1 Design a deployment strategy | | [Deployment/configuration guide](01_architecture_decision_guides.md#14-deployment-configuration-and-capacity-decisions), compute/container/serverless release sections |
| 2.2 Ensure business continuity | | [Backup and DR](../14_hybrid_migration_dr/README.md), Route 53/ARC and service-specific recovery notes |
| 2.3 Determine security controls | | [Security services](../13_security_services/README.md), networking/IAM controls |
| 2.4 Meet reliability requirements | | [HA/scaling](../07_ha_scaling/README.md), messaging failure semantics, Service Quotas method |
| 2.5 Meet performance objectives | | [Compute](../04_compute/README.md), [Databases](../06_databases/README.md), [Analytics](../16_analytics_ml_survey/README.md), caching/edge guides |
| 2.6 Determine cost optimization strategy | | [Cost](../15_cost_well_architected/README.md), purchasing and data-transfer investigations |
| **3. Continuous Improvement for Existing Solutions** | **25%** | |
| 3.1 Improve operational excellence | | [Monitoring/operations](../12_monitoring/README.md), [Well-Architected workflow](../15_cost_well_architected/02_well_architected_framework.md#6-a-professional-review-and-improvement-workflow) |
| 3.2 Improve security | | [Security path](../13_security_services/README.md), Config/CloudTrail/central findings and automated remediation |
| 3.3 Improve performance | | SLO/KPI-based compute, load balancing, database, edge, and analytics investigations |
| 3.4 Improve reliability | | [HA/scaling](../07_ha_scaling/README.md), [tested DR](../14_hybrid_migration_dr/04_disaster_recovery.md), quota/capacity planning |
| 3.5 Identify cost optimizations | | [Measured optimization investigation](../15_cost_well_architected/01_cost_optimization.md#8-run-an-optimization-investigation) |
| **4. Accelerate Workload Migration and Modernization** | **20%** | |
| 4.1 Select workloads/processes for migration | | [Migration services](../14_hybrid_migration_dr/02_migration_services.md), [assessment and 7Rs](01_architecture_decision_guides.md#15-migration-and-modernization-decision-guide) |
| 4.2 Determine the optimal migration approach | | Migration waves, MGN/DMS/DataSync, networking/identity/governance/cutover coverage |
| 4.3 Determine a new architecture | | Compute/container/storage/database decision guides and practical examples |
| 4.4 Identify modernization/enhancement opportunities | | Container/serverless, purpose-built databases, decoupling and integration paths |

## Scenario Drill Method

> **Rule**: These files assume you have already worked sections 01–16. They are
> review, not first exposure. If a row makes no sense, follow its cross-link back
> to the detailed guide.

For SAA-C03, table rows can be flash cards. For SAP-C02, rewrite each scenario
into this decision record before viewing the answer:

1. Rank hard requirements: business outcome, compliance, RTO/RPO/SLO, deadline,
   compatibility, operations, performance, and cost.
2. Draw the account, Region/AZ, trust, network, data, and control-plane boundaries.
3. Select a multi-step architecture and state deployment, observability,
   capacity/quota, failure, rollback/failback, and ownership.
4. Reject every distractor against a named requirement. "Less managed" or "more
   expensive" is incomplete; state the concrete operational, security,
   performance, reliability, cost, or migration failure.
5. Change one constraint and solve again. Examples: overlapping CIDRs, one-hour
   RTO, no downtime, host-bound license, data residency, no Kubernetes skill, or
   a six-month modernization plan.

The [long-form drills](01_architecture_decision_guides.md#16-sap-c02-scenario-drills-requirements-beat-keywords)
are the starting set. Add practice questions until every task above has been
exercised under at least two competing constraints.

---

## Prerequisites

- All of sections [01](../01_foundations/README.md)–[16](../16_analytics_ml_survey/README.md). These guides reference services covered in detail there.
