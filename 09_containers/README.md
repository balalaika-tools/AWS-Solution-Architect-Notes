# Containers: Associate Foundations and Professional Modernization

> How AWS runs containerized workloads, from the container model through
> production platform selection, deployment safety, security, scaling,
> observability, resilience, and cost. The fundamentals support SAA-C03; the
> architecture path adds the operational trade-offs expected for SAP-C02.

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

## SAP-C02 Container Modernization Path

1. Start with the [architecture checklist](01_container_fundamentals.md#8-architecture-checklist-before-choosing-a-platform): classify state, persistent storage, supply-chain controls, rollout, scaling signals, isolation, observability, recovery, and team skill.
2. Choose ECS or EKS and EC2 or Fargate from requirements, not labels. The [base decision guide](02_ecs_ecr_fargate_eks.md#6-ecs-vs-eks-vs-fargate--decision) separates orchestrator from compute.
3. For an AWS-native service, study the [production ECS design](02_ecs_ecr_fargate_eks.md#9-production-ecs-design): capacity providers, safe deployments, service discovery, task identity, multi-account ECR, Spot handling, and regional recovery.
4. If Kubernetes is a real requirement, use the [EKS architecture decision](02_ecs_ecr_fargate_eks.md#10-eks-architecture-decision): node/Fargate selection, autoscalers, pod identity, network capacity, add-ons/upgrades, storage topology, and cluster boundaries.

For modernization, compare the target with the source workload's licensing,
state, network dependencies, deployment frequency, compliance boundary, and
operator skills. A replatform is successful when it measurably improves rollout
safety, recovery, operations, or cost; moving the same manual process into a
container is only repackaging.

---

## Prerequisites

- [Compute](../04_compute/README.md) — the EC2 launch type runs tasks on EC2 instances you manage; understanding EC2 makes the Fargate trade-off obvious.
- [Networking](../03_networking/README.md) — `awsvpc` mode gives each task its own ENI in a subnet, firewalled by security groups.
- [High Availability & Scaling](../07_ha_scaling/README.md) — containers sit behind an ALB/NLB and scale with the same target-tracking concepts as Auto Scaling groups.
