# Exam Patterns

> The capstone review section. Two final-review decision guides that synthesize every prior section into "choose X vs Y" logic and dense side-by-side comparison tables. Use these in the last days before the exam to wire keyword triggers directly to services.

[![AWS](https://img.shields.io/badge/AWS-Solutions%20Architect-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)
[![Exam](https://img.shields.io/badge/Exam-SAA--C03-232F3E.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_architecture_decision_guides.md](01_architecture_decision_guides.md) | Decision guides | Scenario-driven "if you need… → choose…" guides with short decision trees and tables across compute, database, storage, load balancing, decoupling, DNS/edge, HA/DR, connectivity, caching, and secrets. |
| [02_service_comparison_cheatsheets.md](02_service_comparison_cheatsheets.md) | Comparison cheat sheets | A 40+ row keyword→service trigger table, the classic pairwise comparisons (Multi-AZ vs Read Replica, SG vs NACL, etc.), and the hard limits worth memorizing. |

---

## Reading Order

1. **Architecture Decision Guides** — train the "given this requirement, pick this service" reflex. Exam questions are decisions in disguise.
2. **Service Comparison Cheat Sheets** — final rapid-fire review: keyword triggers, tie-breaker tables, and memorized limits.

---

## How to use these

> **Rule**: These files assume you have already worked sections 01–16. They are *review*, not first exposure. If a row makes no sense, follow its cross-link back to the detailed guide.

- ✅ Read both files end-to-end the day before the exam.
- 💡 Treat every table row as a flash card: cover the right column, recall the answer.
- ⚠️ The exam rewards the *best* answer, not a *working* one. The decision guides are organized around the discriminators the exam uses (latency, cost, ordering, transitivity, managed vs self-managed).

---

## Prerequisites

- All of sections [01](../01_foundations/README.md)–[16](../16_analytics_ml_survey/README.md). These guides reference services covered in detail there.
