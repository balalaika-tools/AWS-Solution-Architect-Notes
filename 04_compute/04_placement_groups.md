# Placement Groups

> **Who this is for**: SAA-C03 candidates. Placement groups control *where on AWS's physical hardware* your instances land — to get either the lowest possible network latency, the strongest fault isolation, or controlled fault domains for big distributed systems. The exam tests "which placement strategy for *this* workload." Assumes you've read [01_ec2_fundamentals.md](01_ec2_fundamentals.md).

---

## 1. Why Placement Groups Exist

Normally AWS decides which physical host runs your instance, spreading things out for its own balancing. A **placement group** lets you *influence that placement* to optimize for one of two competing goals:

- **Pack tightly** → lower network latency, higher throughput between instances (but a single hardware fault can take out many instances).
- **Spread apart** → if one rack/host fails, the others survive (but no latency benefit).

```
   Cluster (pack):          Spread (isolate):        Partition (group + isolate):
   ┌──────────────┐         host1: [i]               Partition 1: host-set A [i][i]
   │ one rack     │         host2: [i]               Partition 2: host-set B [i][i]
   │ [i][i][i][i] │         host3: [i]               Partition 3: host-set C [i][i]
   └──────────────┘         (distinct hardware)      (each partition = own racks)
   low latency,             max isolation,           balance: isolation + scale
   shared fate              max 7/AZ                  up to 7 partitions/AZ
```

---

## 2. The Three Strategies

| Strategy | Placement | Optimizes for | Fault isolation | Spans AZs? | Key limit |
|----------|-----------|---------------|-----------------|------------|-----------|
| **Cluster** | Packed close together on the **same rack** in a **single AZ** | **Lowest latency, highest throughput** (10/25/100 Gbps) | ❌ Poor — shared hardware/rack | No — single AZ | All instances should be same type, launched together |
| **Spread** | Each instance on **distinct hardware** (separate racks, separate power/network) | **Maximum fault isolation** for a small set of critical instances | ✅ Best | Yes — can span AZs | **Max 7 instances per AZ** per group |
| **Partition** | Instances grouped into **partitions**, each partition on its **own set of racks** | Large distributed systems that are **rack-aware** | ✅ Per-partition (a partition failure ≠ whole cluster) | Yes — across AZs | **Up to 7 partitions per AZ** |

---

## 3. Cluster — Low Latency, Single AZ

Packs instances physically close so they get the highest bandwidth and lowest network latency AWS can offer (best with enhanced networking / EFA).

- ✅ Use for: **HPC**, tightly-coupled parallel compute (MPI), low-latency clusters, big-data jobs that shuffle lots of data between nodes.
- ⚠️ **Shared fate**: because instances are on the same rack, a single hardware failure can take down the whole group. Not for resilience.
- 💡 Launch all instances **at the same time** with the **same instance type** to maximize the chance AWS can satisfy the tight placement (else you may hit `InsufficientCapacity`).
- All instances must be in a **single AZ** (a cluster group cannot span AZs).

> **Trigger phrase**: "low network latency" / "high throughput between nodes" / "HPC" → **Cluster**.

---

## 4. Spread — Critical Instances on Distinct Hardware

Places each instance on physically separate hardware (distinct racks, each with its own network and power source), so no two instances share a single point of failure.

- ✅ Use for: a **small number of critical instances** that must not fail together — e.g. primary/secondary nodes, individual critical app servers.
- ⚠️ Hard limit: **7 instances per AZ** per spread group. (Spread across N AZs = up to 7×N.)
- Best when you have a handful of important instances and want them as isolated as possible.

> **Trigger phrase**: "critical instances must not share hardware" / "maximum availability for a few servers" → **Spread**.

---

## 5. Partition — Large Distributed, Rack-Aware Systems

Divides instances into **partitions**; each partition sits on its **own set of racks** with no shared hardware between partitions. AWS exposes which partition an instance is in (via metadata), so partition-aware software can place replicas in different partitions.

- ✅ Use for: large distributed/replicated systems that are **rack-aware** — **HDFS, HBase, Cassandra, Kafka**. They place data replicas in separate partitions so a rack failure loses at most one replica.
- Up to **7 partitions per AZ**; can span multiple AZs in a region; scales to **hundreds of instances**.
- A failure in one partition's hardware doesn't affect instances in other partitions.

> **Trigger phrase**: "Cassandra / HDFS / Kafka" / "large distributed system, isolate failures by partition" → **Partition**.

---

## 6. Decision Guide

```
            ┌──────────────────────────────────────────────┐
            │ What do you need most?                        │
            └───────┬───────────────┬──────────────┬────────┘
              lowest latency   isolate a few   large distributed,
              (HPC, tight      critical        replicated, rack-
               coupling)       instances        aware system
                  │                │                 │
               CLUSTER          SPREAD           PARTITION
            (1 AZ, packed)   (≤7/AZ, distinct  (≤7 partitions/AZ,
                              hardware)          own racks each)
```

| If the question says… | Choose |
|------------------------|--------|
| "lowest network latency", "highest throughput", "HPC", "tightly coupled" | **Cluster** |
| "critical", "must not share hardware", small set, "maximize availability" | **Spread** |
| "HDFS / HBase / Cassandra / Kafka", "large distributed system", "partition failures" | **Partition** |

💡 You can **move** an existing instance into a placement group (it must be in the `stopped` state), and you change groups only while stopped.

---

## Key Exam Points

- ✅ **Cluster** = pack tight in **one AZ** for **lowest latency/highest throughput**; poor fault isolation; great for HPC.
- ✅ **Spread** = each instance on **distinct hardware**; best isolation; **max 7 instances per AZ**; for a few critical instances.
- ✅ **Partition** = instances grouped into partitions each on their own racks; for **large rack-aware distributed systems** (Cassandra, HDFS, Kafka); **up to 7 partitions per AZ**.
- ✅ Spread and Partition can span AZs; **Cluster cannot** (single AZ).
- ✅ An instance must be **stopped** to be added to or moved between placement groups.
- ✅ Placement groups are free — you pay only for the instances.

---

## Common Mistakes

- ❌ Using **Cluster** for high availability — it concentrates fault risk, it doesn't reduce it.
- ❌ Trying to put more than **7 instances per AZ** in a Spread group.
- ❌ Choosing **Spread** for a large (hundreds of instances) distributed system — that's what **Partition** is for; Spread's 7/AZ cap won't fit.
- ❌ Expecting a **Cluster** group to span AZs — it can't.
- ❌ Mixing many instance types in a Cluster group and then hitting capacity errors — use the same type and launch together.

---

## Relevant Limits

- **Spread**: maximum **7 instances per AZ** per placement group.
- **Partition**: up to **7 partitions per AZ**; can span AZs within a region; supports hundreds of instances.
- **Cluster**: single AZ; recommend identical instance types launched simultaneously.
- An instance can belong to **only one** placement group at a time, and must be **stopped** to change it.

---

**Next**: [EBS — Elastic Block Store](../05_storage/01_ebs.md)
