# Cost Optimization & the Well-Architected Framework

> The two "design and govern" topics the exam expects every architect to know: **how to keep the bill down** (cost drivers, billing tools, and pricing levers) and **the Well-Architected Framework** — AWS's six-pillar lens for reviewing any design. Each note starts from the underlying concept (CapEx vs OpEx, what a "good architecture" even means) before the AWS tooling.

[![AWS](https://img.shields.io/badge/AWS-Cost%20%7C%20Well--Architected-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/architecture/well-architected/)
[![Exam](https://img.shields.io/badge/Exam-SAA--C03-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_cost_optimization.md](01_cost_optimization.md) | Cost Optimization | CapEx vs OpEx recap and the four cloud cost drivers (compute, storage, data transfer — **egress!**, requests). Billing & cost tools: **Budgets, Cost Explorer, CUR, Cost Allocation Tags, billing alarms, Pricing Calculator, Compute Optimizer, Trusted Advisor**. Pricing levers: right-sizing, Savings Plans/RIs/Spot, S3 classes & lifecycle, choosing the right DB, killing idle resources, minimizing data transfer, consolidated billing volume discounts. Plus "how to reduce cost for X" decision points. |
| [02_well_architected_framework.md](02_well_architected_framework.md) | Well-Architected Framework | What the WAF is and why it exists (design principles, the review process, the WA Tool). The **six pillars** — for each: goal, key design principles, and the services that support it. The pillar comparison table, the general design principles (stop guessing capacity, automate, loosely couple, design for failure, scale horizontally), and how the framework shows up in exam question framing. |

---

## Reading Order

1. **Cost Optimization** — concrete and tool-driven; it doubles as the deep-dive for the Cost Optimization pillar referenced in the next file.
2. **Well-Architected Framework** — the umbrella that ties every prior section (security, reliability, performance, cost, operations, sustainability) into one review lens. Read it last so the pillars map onto services you already know.

---

## Prerequisites

- [Compute](../04_compute/README.md) — EC2 pricing models (On-Demand, Reserved, Spot, Savings Plans) are the heart of compute cost optimization.
- [Storage](../05_storage/README.md) — S3 storage classes and lifecycle policies are the main storage cost lever.
- [HA & Scaling](../07_ha_scaling/README.md) and [Networking](../03_networking/README.md) — Auto Scaling (right-sizing), VPC endpoints, and same-AZ traffic all feed the cost and reliability pillars.
