# Identity and Access Management (IAM)

> Who controls *who* can do *what* in your AWS account — the security backbone of every architecture on the exam.

[![AWS](https://img.shields.io/badge/AWS-IAM-FF9900.svg?logo=amazonaws&logoColor=white)](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_authn_vs_authz.md](01_authn_vs_authz.md) | AuthN vs AuthZ | Authentication vs authorization, principals, least privilege, root safety, and the distinct jobs of grants and guardrails. |
| [02_users_groups_roles.md](02_users_groups_roles.md) | Identities | Users, groups, roles, instance profiles, role chaining, session duration, and enterprise session attributes. |
| [03_iam_policies.md](03_iam_policies.md) | Policies | Policy JSON, evaluation logic, multi-account evaluation, boundaries/session policies/SCPs, advanced elements, and direct resource sharing vs roles. |
| [04_organizations_sts_federation.md](04_organizations_sts_federation.md) | Multi-account & federation | Organizations, STS/federation, Identity Center, Control Tower landing zones, delegated administration, and centralized operations. |
| [05_directory_services.md](05_directory_services.md) | Directory Services | Managed Microsoft AD vs AD Connector vs Simple AD, hybrid migration, account/Region sharing, and identity-source failure modes. |

---

## Reading Order

1. **AuthN vs AuthZ** — the vocabulary and mental model everything else builds on.
2. **Users, Groups, Roles** — the identities that requests come from.
3. **IAM Policies** — the rules that decide allow or deny, plus advanced elements and resource-based vs role patterns.
4. **Organizations, STS, Federation** — scaling identity across many accounts; Tag Policies, Control Tower.
5. **Directory Services** — bringing Active Directory into AWS or proxying to on-prem AD.

---

## SAP-C02 Domain Map

IAM questions on SAP-C02 rarely ask for a definition in isolation. They describe an
organization, an identity path, and a failure or governance constraint. Map the scenario to the
exam domain first, then follow the request through authentication, role assumption, policy
evaluation, and the target resource.

| Scenario | Primary SAP-C02 domain | What you must decide | Notes |
|----------|------------------------|----------------------|-------|
| **Cross-account access** | Domain 1: Design Solutions for Organizational Complexity (26%) | Direct resource policy or `AssumeRole`; both-account authorization; which SCP, boundary, session policy, and explicit deny applies before and after STS | [01](01_authn_vs_authz.md), [02](02_users_groups_roles.md), [03](03_iam_policies.md), [04](04_organizations_sts_federation.md) |
| **Third-party access and federation** | Domain 1, especially security controls | Workforce SAML/OIDC vs vendor `AssumeRole`; confused-deputy protection; source attribution, ABAC tags, and session lifetime | [02](02_users_groups_roles.md), [03](03_iam_policies.md), [04](04_organizations_sts_federation.md) |
| **Multi-account governance** | Domain 1, especially multi-account environments | OU structure, SCP scope, Control Tower vs Organizations, account vending, delegated administrators, and organization-wide logging/security coverage | [03](03_iam_policies.md), [04](04_organizations_sts_federation.md) |
| **Centralized workforce identity** | Domain 1 and Domain 2: Design for New Solutions (29%) | IAM Identity Center permission sets and identity source; temporary access across accounts; when AD is or is not required | [02](02_users_groups_roles.md), [04](04_organizations_sts_federation.md), [05](05_directory_services.md) |
| **Directory and migration dependencies** | Domain 4: Accelerate Workload Migration and Modernization (20%) | Managed AD vs AD Connector vs trust; authoritative identity location; WAN/DNS failure; directory sharing and multi-Region behavior during migration | [05](05_directory_services.md) |
| **Security improvement and drift** | Domain 3: Continuous Improvement for Existing Solutions (25%) | Reduce standing credentials, analyze effective permissions, centralize evidence/findings, and select preventive/proactive/detective controls plus remediation | [01](01_authn_vs_authz.md), [03](03_iam_policies.md), [04](04_organizations_sts_federation.md) |

### Advanced scenario checklist by file

- **01 — AuthN vs AuthZ**: explain why a successfully authenticated session is denied, and
  separate what identity/resource policies can grant from what boundaries, session policies,
  and SCPs can only cap.
- **02 — Users, groups, and roles**: design an enterprise cross-account role session with a
  correct maximum duration, the 1-hour role-chaining limit, vendor `ExternalId`, immutable
  source attribution, and constrained transitive session tags.
- **03 — Policies**: evaluate a request account by account, split the path at `AssumeRole`, and
  eliminate distractors by locating the missing allow or winning explicit deny.
- **04 — Organizations, STS, and federation**: design an OU/account structure and account
  vending path; choose Control Tower or Organizations; delegate operations without using the
  management account daily; prove account and Region coverage for audit/security services.
- **05 — Directory Services**: choose an identity source and workload directory separately;
  compare proxy, trust, migration, directory sharing, and multi-Region replication while naming
  what fails when connectivity or an authoritative directory is lost.

---

## Prerequisites

- Basic understanding of what an AWS account is and the shared responsibility model
- [Foundations](../01_foundations/README.md) — AWS global infrastructure and account basics
