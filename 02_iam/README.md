# Identity and Access Management (IAM)

> Who controls *who* can do *what* in your AWS account — the security backbone of every architecture on the exam.

[![AWS](https://img.shields.io/badge/AWS-IAM-FF9900.svg?logo=amazonaws&logoColor=white)](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_authn_vs_authz.md](01_authn_vs_authz.md) | AuthN vs AuthZ | Authentication vs authorization, principals, least privilege, how IAM models them. Root user warnings. |
| [02_users_groups_roles.md](02_users_groups_roles.md) | Identities | Users, groups, roles, instance profiles. Temporary vs long-lived credentials. When to use a role vs a user. |
| [03_iam_policies.md](03_iam_policies.md) | Policies | Policy JSON structure, identity-based vs resource-based vs boundaries vs SCPs, evaluation logic, real examples. |
| [04_organizations_sts_federation.md](04_organizations_sts_federation.md) | Multi-account & federation | Organizations, OUs, SCPs, STS AssumeRole, SAML/OIDC federation, IAM Identity Center, cross-account patterns. |

---

## Reading Order

1. **AuthN vs AuthZ** — the vocabulary and mental model everything else builds on.
2. **Users, Groups, Roles** — the identities that requests come from.
3. **IAM Policies** — the rules that decide allow or deny.
4. **Organizations, STS, Federation** — scaling identity across many accounts and external identity providers.

---

## Prerequisites

- Basic understanding of what an AWS account is and the shared responsibility model
- [Foundations](../01_foundations/README.md) — AWS global infrastructure and account basics
