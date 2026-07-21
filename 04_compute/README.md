# Compute — EC2

> Virtual servers in the cloud: how EC2 works, what it costs, how you boot and image instances, and how to physically place them for performance or resilience.

[![AWS](https://img.shields.io/badge/AWS-EC2-FF9900.svg?logo=amazonaws&logoColor=white)](https://docs.aws.amazon.com/ec2/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_ec2_fundamentals.md](01_ec2_fundamentals.md) | EC2 basics | What EC2 is, instance families & naming, vCPU/memory, lifecycle (stop vs terminate), instance store vs EBS-backed, ENIs, key pairs, status checks. |
| [02_ec2_pricing_models.md](02_ec2_pricing_models.md) | Pricing | On-Demand, Reserved Instances, Savings Plans, Spot + Spot Fleet, Dedicated Hosts vs Instances, Capacity Reservations. Workload → model decision table. |
| [03_amis_userdata_metadata.md](03_amis_userdata_metadata.md) | Imaging & bootstrap | AMIs and golden images, creating an AMI, snapshots, User Data bootstrap scripts, Instance Metadata Service (IMDSv1 vs IMDSv2). |
| [04_placement_groups.md](04_placement_groups.md) | Placement | Cluster, spread, and partition placement groups — controlling where instances run for latency or fault isolation. |

---

## Reading Order

1. **EC2 Fundamentals** — what an instance is and how it plugs into your VPC and EBS.
2. **EC2 Pricing Models** — the single biggest cost lever in the exam; pick the right purchasing option per workload.
3. **AMIs, User Data & Metadata** — how instances boot, get configured, and discover themselves.
4. **Placement Groups** — advanced physical placement for HPC and fault tolerance.

## Professional Track: Fleet and Modernization Decisions

For SAP-C02, use the same files as an architecture path rather than a service
tour:

1. [EC2 Fundamentals](01_ec2_fundamentals.md#8-operating-and-modernizing-a-fleet)
   — govern managed nodes with Systems Manager, stage patching, evaluate licenses
   and quotas, and decide when host ownership justifies EC2.
2. [EC2 Pricing Models](02_ec2_pricing_models.md#9-enterprise-purchasing-and-capacity-planning)
   — separate price discounts from capacity assurance, evaluate organization-wide
   coverage and utilization, diversify Spot, and align commitments with the
   modernization roadmap.
3. [AMIs, User Data and Instance Metadata](03_amis_userdata_metadata.md#6-controlled-image-delivery-at-enterprise-scale)
   — build and distribute tested encrypted images, promote launch-template
   versions, and choose immutable replacement versus controlled in-place state.
4. [Placement Groups](04_placement_groups.md#7-capacity-and-recovery-scenario)
   — combine topology, quotas, reservations, failure domains, and workload
   recovery instead of selecting a placement strategy from a keyword.

The professional decision is end to end: establish the business SLO, choose
the platform, prove capacity, automate deployment and operations, model the
commitment, and test rollback. If OS control is not a real requirement, compare
EC2 with a managed container, serverless, batch, or database service before
optimizing the fleet you already have.

---

## Recognition-Level Compute Services

These appear in SAA-C03 scope less deeply than EC2/Lambda/Fargate, but you
should recognize the exam clue.

| Service | Pick it when the question says... |
|---------|-----------------------------------|
| **Elastic Beanstalk** | Deploy a web app with less ops than raw EC2, while still using familiar services like EC2, ALB, ASG, RDS. |
| **AWS Batch** | Run queued batch/HPC jobs, array jobs, or Spot-friendly compute fleets without managing schedulers. |
| **AWS Outposts** | Run AWS infrastructure/services on-premises for low latency or data residency. |
| **AWS Wavelength** | Place compute/storage at 5G provider edge locations for ultra-low-latency mobile apps. |
| **VMware Cloud on AWS** | Move/extend VMware workloads using familiar VMware tooling. |
| **Serverless Application Repository** | Find/deploy prebuilt serverless applications and patterns. |

---

## Prerequisites

- [Networking](../03_networking/README.md) — VPCs, subnets, and security groups (an EC2 instance lives in a subnet and is firewalled by a security group)
- [IAM](../02_iam/README.md) — instance profiles / roles for granting AWS permissions to an instance
