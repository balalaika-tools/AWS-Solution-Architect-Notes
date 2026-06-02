# Containers

> How AWS runs containerized workloads: the container model itself (images, registries, the orchestration problem), then the AWS stack — ECR for images, ECS and EKS for orchestration, and Fargate for serverless compute under both.

[![AWS](https://img.shields.io/badge/AWS-ECS%20%7C%20EKS%20%7C%20Fargate-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/containers/)
[![Docker](https://img.shields.io/badge/Docker-Container%20Model-2496ED.svg?logo=docker&logoColor=white)](https://docs.docker.com/get-started/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_container_fundamentals.md](01_container_fundamentals.md) | Container fundamentals | What a container is, containers vs VMs, images and layers, registries, why containers, the orchestration problem, the Docker mental model. No AWS specifics. |
| [02_ecs_ecr_fargate_eks.md](02_ecs_ecr_fargate_eks.md) | The AWS container stack | ECR (registry, scanning, lifecycle), ECS (clusters/task defs/services/ALB), EC2 vs Fargate launch types, EKS (managed Kubernetes), the ECS vs EKS vs Fargate decision, awsvpc networking, task IAM roles, service auto scaling, App Runner. |

---

## Reading Order

1. **Container Fundamentals** — the vocabulary (image, layer, registry, orchestration) every AWS service in this section assumes. Read this first even if you "know Docker."
2. **ECS, ECR, Fargate & EKS** — the actual AWS products, how they fit together, and how to choose between them on the exam.

---

## Prerequisites

- [Compute](../04_compute/README.md) — the EC2 launch type runs tasks on EC2 instances you manage; understanding EC2 makes the Fargate trade-off obvious.
- [Networking](../03_networking/README.md) — `awsvpc` mode gives each task its own ENI in a subnet, firewalled by security groups.
- [High Availability & Scaling](../07_ha_scaling/README.md) — containers sit behind an ALB/NLB and scale with the same target-tracking concepts as Auto Scaling groups.
