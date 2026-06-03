# Identity and Access Management (IAM)

> Who controls *who* can do *what* in your AWS account — the security backbone of every architecture on the exam.

[![AWS](https://img.shields.io/badge/AWS-IAM-FF9900.svg?logo=amazonaws&logoColor=white)](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_authn_vs_authz.md](01_authn_vs_authz.md) | AuthN vs AuthZ | Authentication vs authorization, principals, least privilege, how IAM models them. Root user warnings. |
| [02_users_groups_roles.md](02_users_groups_roles.md) | Identities | Users, groups, roles, instance profiles. Temporary vs long-lived credentials. When to use a role vs a user. |
| [03_iam_policies.md](03_iam_policies.md) | Policies | Policy JSON structure, four policy types, evaluation logic, `NotAction`/`NotPrincipal`/policy variables, resource-based policy vs IAM role. |
| [04_organizations_sts_federation.md](04_organizations_sts_federation.md) | Multi-account & federation | Organizations, OUs, SCPs, Tag Policies, STS AssumeRole, SAML/OIDC federation, IAM Identity Center, Control Tower. |
| [05_directory_services.md](05_directory_services.md) | Directory Services | Managed Microsoft AD vs AD Connector vs Simple AD — when to use each, integrations, failure modes. |

---

## Reading Order

1. **AuthN vs AuthZ** — the vocabulary and mental model everything else builds on.
2. **Users, Groups, Roles** — the identities that requests come from.
3. **IAM Policies** — the rules that decide allow or deny, plus advanced elements and resource-based vs role patterns.
4. **Organizations, STS, Federation** — scaling identity across many accounts; Tag Policies, Control Tower.
5. **Directory Services** — bringing Active Directory into AWS or proxying to on-prem AD.

---

## Prerequisites

- Basic understanding of what an AWS account is and the shared responsibility model
- [Foundations](../01_foundations/README.md) — AWS global infrastructure and account basics
