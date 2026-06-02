# High Availability & Scaling

> How to keep an application up and responsive as load changes and components fail: distributing traffic across many targets with load balancers, and adding/removing capacity automatically with Auto Scaling groups.

[![AWS](https://img.shields.io/badge/AWS-ELB-FF9900.svg?logo=amazonaws&logoColor=white)](https://docs.aws.amazon.com/elasticloadbalancing/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_load_balancing_concepts.md](01_load_balancing_concepts.md) | Load balancing fundamentals | Why distribute traffic, horizontal vs vertical scaling, OSI layer 4 vs 7, listeners, target groups, health checks, connection draining, sticky sessions, request flow. |
| [02_elb_alb_nlb_gwlb.md](02_elb_alb_nlb_gwlb.md) | The ELB family | Classic vs ALB vs NLB vs GWLB, big comparison table, cross-zone load balancing, SSL/TLS termination & SNI, decision guide. |
| [03_auto_scaling_groups.md](03_auto_scaling_groups.md) | Auto Scaling Groups | Launch templates, desired/min/max, scaling policies (target tracking, step, scheduled, predictive), health checks, instance refresh, lifecycle hooks. |

---

## Reading Order

1. **Load Balancing Concepts** — the vocabulary (listeners, targets, health checks, L4 vs L7) every later file assumes.
2. **ELB: ALB, NLB & GWLB** — the actual AWS products and how to choose between them.
3. **Auto Scaling Groups** — adding and removing capacity automatically, and wiring it to a load balancer.

---

## Prerequisites

- [Compute](../04_compute/README.md) — EC2 instances are the things being balanced and scaled.
- [Networking](../03_networking/README.md) — load balancers live in subnets across multiple Availability Zones and are firewalled by security groups.
