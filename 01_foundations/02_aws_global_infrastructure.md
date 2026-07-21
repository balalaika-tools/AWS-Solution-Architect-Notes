# AWS Global Infrastructure

> **Who this is for**: Engineers who understand the cloud service models
> ([01_cloud_computing_fundamentals.md](01_cloud_computing_fundamentals.md)) and now need to know
> *where* AWS physically runs, how that enables high availability, and how to pick a Region. No
> networking background assumed — subnet details come later in
> [Networking](../03_networking/README.md).

---

## 1️⃣ The Building Blocks: Region → AZ → Subnet

AWS is organized as a hierarchy of physical and logical boundaries. The three you must know cold:

- **Region** — a geographic area (e.g., `us-east-1` N. Virginia, `eu-west-1` Ireland). Contains
  multiple, isolated Availability Zones. You choose a Region for your resources.
- **Availability Zone (AZ)** — one or more discrete data centers within a Region, each with
  independent power, cooling, and networking, but connected to sibling AZs by **low-latency,
  high-bandwidth private links**. Named like `us-east-1a`, `us-east-1b`.
- **Subnet** — a slice of your network that lives **inside exactly one AZ**. This is where you
  actually place resources like EC2 instances.

```
                    AWS Region (e.g., us-east-1)
   ┌─────────────────────────────────────────────────────────────┐
   │                                                             │
   │   AZ us-east-1a          AZ us-east-1b        AZ us-east-1c │
   │  ┌──────────────┐      ┌──────────────┐     ┌────────────┐  │
   │  │ Subnet       │      │ Subnet       │     │ Subnet     │  │
   │  │ 10.0.1.0/24  │      │ 10.0.2.0/24  │     │ 10.0.3.0/24│  │
   │  │  [EC2] [EC2] │◄────►│  [EC2] [EC2] │◄───►│  [EC2]     │  │
   │  └──────────────┘      └──────────────┘     └────────────┘  │
   │        ▲   low-latency private links between AZs   ▲        │
   │        └───────────────────────────────────────────┘        │
   └─────────────────────────────────────────────────────────────┘
        (each AZ = 1+ physically separate data centers)
```

> **Rule**: A subnet maps to exactly **one AZ**. To be highly available you spread subnets — and
> the resources in them — across **multiple AZs**.

---

## 2️⃣ How AZs Enable High Availability

AZs are the foundation of fault tolerance on AWS. Because each AZ has independent power, cooling,
and networking and is physically separated (typically km apart), a failure in one AZ — a fire,
power loss, flood — is unlikely to take down another.

✅ **Correct HA pattern**: deploy your application across **two or more AZs** behind a load
balancer. If one AZ fails, traffic shifts to the healthy AZs and the app stays up.

❌ **Anti-pattern**: putting all instances in a single AZ. That AZ becomes a single point of failure.

```
            Load Balancer (spans AZs)
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
   AZ-a [web][web]     AZ-b [web][web]
        │                   │
   AZ-a [db primary] ──► AZ-b [db standby]   (e.g., RDS Multi-AZ)
```

> **Key insight**: The fast private links between AZs make **synchronous replication** (e.g., RDS
> Multi-AZ) practical *within a Region*. Across Regions, latency is too high for synchronous
> replication, so cross-Region designs use asynchronous replication.

---

## 3️⃣ The Edge: Edge Locations & Points of Presence

Beyond Regions and AZs, AWS runs a much larger network of **edge locations** (also called Points
of Presence, PoPs) in hundreds of cities. These are *not* full data centers — they cache content
and terminate connections close to end users.

| Concept | What it is | Used by |
|---------|-----------|---------|
| **Edge location** | Small site that caches data near users to cut latency | **CloudFront** (CDN), **Route 53** (DNS), **Global Accelerator** |
| **Regional edge cache** | Larger cache between edge locations and the origin Region | CloudFront, for less-popular content |

💡 Edge locations far outnumber Regions/AZs. When a question mentions *"low latency for global
users,"* *"cache static content near viewers,"* or *"serve from the nearest location,"* think
**edge** (CloudFront / Global Accelerator), covered in [DNS & Edge](../08_dns_edge/README.md).

---

## 4️⃣ Specialized Infrastructure: Local Zones, Wavelength, Outposts

For workloads that need to be even closer to users or on-premises, AWS extends infrastructure
beyond the standard Region/AZ model:

| Offering | What it is | When to use it |
|----------|-----------|----------------|
| **Local Zone** | An extension of a Region placing compute/storage in a large metro area, closer to users | Single-digit-ms latency for end users in a specific city (gaming, media, live editing) |
| **Wavelength Zone** | AWS infrastructure embedded **inside telecom 5G networks** | Ultra-low-latency mobile apps (AR/VR, connected cars) served via the carrier network |
| **Outposts** | Physical AWS racks/servers installed **in your own data center**, managed by AWS | Hybrid/low-latency needs or data residency where the workload must stay on-prem but use AWS APIs |

> **Rule**: **5G / mobile carrier** latency → **Wavelength**. **Metro-area** low latency →
> **Local Zone**. **Run AWS in *my* data center** → **Outposts**.

---

## 5️⃣ Global vs Regional vs Zonal Services

AWS services operate at different scopes. Knowing a service's scope tells you where it's resilient
and where its endpoints live — a frequent source of exam traps.

| Scope | Meaning | Examples |
|-------|---------|----------|
| **Global** | One logical service across all Regions; not tied to a Region | **IAM**, **Route 53**, **CloudFront**, **WAF** (for CloudFront) |
| **Regional** | Lives in one Region; AWS spreads it across that Region's AZs for resilience | **S3**, **DynamoDB**, **Lambda**, **SQS** — and the *region-level* view of EC2/VPC |
| **Zonal** | Tied to a single AZ; you are responsible for spreading across AZs | **EC2 instances**, **EBS volumes**, **subnets** |

⚠️ **Common trap**: S3 buckets are **Regional** (you pick a Region at creation), even though the
*bucket name* is globally unique. Globally unique name ≠ global service.

⚠️ EBS volumes are **zonal** — a volume in `us-east-1a` cannot attach to an instance in
`us-east-1b`. You'd snapshot it and recreate in the other AZ.

> **Key insight**: AWS already makes **Regional** services multi-AZ for you. For **zonal**
> resources (EC2, EBS), *you* are responsible for distributing across AZs to achieve HA.

---

## 6️⃣ How to Choose a Region

Region selection is a **constraint and trade-off exercise**, not simply "pick the closest or
cheapest Region." First eliminate Regions that cannot satisfy legal or technical requirements.
Then compare the viable candidates against the workload's latency, continuity, cost, and
operational targets.

| Scenario driver | What to establish | Region decision | Easy-to-miss consequence |
|-----------------|-------------------|-----------------|--------------------------|
| **Latency** | Where users and dependent systems are, and the workload's measured latency target | Place latency-sensitive compute near the dominant users or dependencies. For a global audience, test whether edge caching or acceleration is enough before duplicating the application across Regions. | A nearby frontend does not fix a backend that makes synchronous calls to a distant database or on-premises system. Measure the whole request path. |
| **Data sovereignty and residency** | Where data may be stored, processed, backed up, logged, and decrypted; which jurisdictions and AWS partitions are permitted | Treat the approved geography as a hard filter. Keep replicas, backups, logs, and encryption keys inside it unless the policy explicitly permits export. | Choosing an EU Region does not by itself prove compliance. The application's replication, support, encryption, and data-handling controls must also meet the requirement. |
| **Service and feature availability** | Whether every required service, feature, instance type, quota, and integration exists in both the primary and recovery Regions | Remove any candidate that lacks a mandatory capability, or choose an explicit substitute and test it before committing. | A service name may be available while a needed feature, hardware family, or quota is not. Verify the current regional service list rather than assuming parity. |
| **Inter-Region data transfer** | How much traffic replication, cross-Region API calls, centralized inspection, and data access will generate | Co-locate tightly coupled components. If multiple Regions are required, make cross-Region interactions coarse-grained and include transfer charges in the cost model. | Inter-Region transfer commonly incurs charges from the source Region. A chatty architecture adds both cost and latency, even when each service is inexpensive. |
| **Business continuity** | The business impact, failure scope, **RTO** (maximum acceptable recovery time), and **RPO** (maximum acceptable data loss) | Use multi-AZ in one Region for most workloads. Add a second Region only when the recovery objective includes a Regional impairment or another explicit geographic requirement. | A second Region is not a recovery plan by itself. Data replication, routing, dependencies, failover, and failback must all be designed and tested. |
| **Operational complexity** | Whether the team can keep deployments, configuration, quotas, observability, security controls, and runbooks consistent across Regions | Prefer a single-Region, multi-AZ design when it meets the requirements. Choose multi-Region only when its benefit justifies the extra failure modes and operating burden. | Untested failover, configuration drift, asymmetric quotas, or a dependency that still lives in one Region can make a nominally multi-Region system less reliable. |
| **Regional price** | The full workload cost in each viable Region, including duplicate capacity, transfer, support services, and recovery resources | Compare total cost only after compliance and capability requirements are satisfied. | Unit prices vary by Region, but a lower compute price can be outweighed by data transfer or an additional recovery footprint. |

### A practical decision sequence

1. **Set hard constraints**: approved jurisdictions, required certifications, services, features,
   and integrations.
2. **Measure placement needs**: user latency, dependency latency, data volume, and where data may
   cross a boundary.
3. **Define resilience targets**: decide whether multi-AZ availability is sufficient or whether
   the workload needs a tested cross-Region recovery strategy to meet its RTO and RPO.
4. **Price the complete design**: include replicas, standby capacity, backups, observability, and
   inter-Region transfer—not only the primary compute resources.
5. **Validate operations**: confirm quota parity, deployment automation, monitoring, and tested
   failover/failback runbooks in every selected Region.

> **Default**: Start with one compliant Region and a multi-AZ architecture. Add edge services for
> global delivery when they solve the latency problem. Adopt multi-Region only for a stated
> latency, sovereignty, or business-continuity requirement that the simpler design cannot meet.

---

## Key Exam Points

- **Region** = geographic area; contains **≥3 AZs** (most). **AZ** = 1+ isolated data centers.
  **Subnet** = lives in **exactly one AZ**.
- **HA = spread resources across multiple AZs**; a single AZ is a single point of failure.
- AZs have **low-latency private links** → synchronous replication within a Region (e.g., RDS Multi-AZ).
- **Edge locations / PoPs** cache content near users → CloudFront, Route 53, Global Accelerator.
- **Wavelength = 5G/telecom**, **Local Zone = metro proximity**, **Outposts = AWS in your data center**.
- Scope: **IAM, Route 53, CloudFront = global**; **S3, DynamoDB, Lambda = regional**;
  **EC2, EBS, subnets = zonal**.
- Choosing a Region: apply **sovereignty and service constraints first**, then weigh latency,
  inter-Region transfer, continuity targets, total cost, and operational complexity.

---

## Common Mistakes

- ❌ Thinking an Availability Zone is a single data center — an AZ is **one or more** discrete data centers.
- ❌ Believing a subnet can span multiple AZs. It cannot — **one subnet = one AZ**.
- ❌ Treating S3 as a global service because bucket names are globally unique. S3 is **Regional**.
- ❌ Expecting an **EBS volume** to attach across AZs. EBS is **zonal**; use a snapshot to move it.
- ❌ Picking a Region purely on price when the scenario states a **data-residency/compliance** requirement.
- ❌ Confusing **edge locations** (cache, no full compute) with **Regions/AZs** (full infrastructure).

---

**Next**: [03_shared_responsibility_and_account_basics.md — Shared Responsibility & Account Basics](03_shared_responsibility_and_account_basics.md)
