# EC2 Fundamentals

> **Who this is for**: Engineers prepping for SAA-C03 who want a solid mental model of EC2 — what an instance is, how it's sized, how its lifecycle and storage behave, and how it attaches to a VPC. Networking-light readers welcome; the networking bits are explained inline. Assumes you've skimmed [Networking](../03_networking/README.md) (subnets and security groups).

---

## 1. What EC2 Is and Why

**EC2 (Elastic Compute Cloud)** is AWS's Infrastructure-as-a-Service for renting virtual servers — called **instances** — by the second. You pick an OS image, a hardware size, a network location, and a firewall, and AWS launches a VM on a physical host in one of its data centers.

Why it exists: before cloud, capacity was a CapEx guessing game — buy too many servers and waste money, buy too few and fall over under load. EC2 turns servers into an on-demand, pay-as-you-go resource you can launch in seconds and terminate when done.

```
        ┌──────────────────────────────────────────────┐
        │              Physical Host (AWS)              │
        │   ┌──────────┐  ┌──────────┐  ┌──────────┐    │
        │   │ Instance │  │ Instance │  │ Instance │    │  ← your VM(s),
        │   │  (your   │  │ (someone │  │ (someone │    │    isolated by
        │   │   VM)    │  │  else's) │  │  else's) │    │    the hypervisor
        │   └────┬─────┘  └──────────┘  └──────────┘    │
        │        │  Nitro hypervisor                    │
        └────────┼──────────────────────────────────────┘
                 │ ENI (virtual NIC)
                 ▼
          VPC subnet in one Availability Zone
```

> **Key insight**: EC2 is the "default" compute building block. The exam often tests *when not to use it* — if a workload is event-driven, prefer Lambda; if containerized, prefer ECS/Fargate; if you only run it occasionally, prefer serverless. EC2 wins when you need full OS control, long-running processes, or specific instance hardware.

EC2 lives in **one AZ** at a time. An instance does **not** automatically survive an AZ failure — high availability comes from running instances across multiple AZs behind a load balancer / Auto Scaling group, not from EC2 itself.

---

## 2. Instance Families and the Naming Convention

An instance type name like `m5.large` or `c6gn.xlarge` encodes everything about the hardware:

```
        m   5   g   n   .   xlarge
        │   │   │   │       │
        │   │   │   │       └── size (vCPU/memory tier within the type)
        │   │   │   └────────── extra capability (n = enhanced networking)
        │   │   └────────────── processor family (g = AWS Graviton/ARM)
        │   └────────────────── generation (higher = newer, usually cheaper/faster)
        └────────────────────── family (m = general purpose)
```

| Position | Meaning | Examples |
|----------|---------|----------|
| 1st letter | **Family** — the optimization category | `m`, `c`, `r`, `i`, `g` |
| Number | **Generation** — newer is better/cheaper | `5`, `6`, `7` |
| Extra letters | **Capabilities** | `g`=Graviton/ARM, `a`=AMD, `n`=network-optimized, `d`=local NVMe disk |
| After dot | **Size** | `nano`, `micro`, `small`, `medium`, `large`, `xlarge`, `2xlarge` … |

Each size roughly **doubles** vCPU and memory: `large` → `xlarge` → `2xlarge` doubles each step.

### The five optimization categories

| Category | Letters | Optimized for | Typical workloads |
|----------|---------|---------------|-------------------|
| **General purpose** | `t`, `m` | Balanced CPU/memory/network | Web servers, small DBs, dev/test, code repos |
| **Compute optimized** | `c` | High CPU per dollar | Batch processing, HPC, gaming servers, media transcoding, high-throughput web |
| **Memory optimized** | `r`, `x`, `z` | High RAM | In-memory caches, large/high-performance relational DBs, real-time big-data analytics |
| **Storage optimized** | `i`, `d`, `h` | High local disk IOPS/throughput | NoSQL (Cassandra), data warehouses, distributed file systems, OLTP |
| **Accelerated computing** | `p`, `g`, `inf`, `trn` | GPUs / ML chips | Machine learning training & inference, graphics rendering, HPC |

💡 The `t` family (e.g. `t3.micro`) is **burstable**: it runs at a low baseline CPU and earns *CPU credits* it can spend to burst higher for short periods — ideal for bursty, low-average workloads. Once credits run out it throttles (or charges you in "unlimited" mode).

> **Exam mnemonic**: **C**ompute = **C**PU, **R**AM = `R`, **I**OPS/disk = `I`, **M** = **M**iddle/balanced, **G**PU = `G`/`P`.

---

## 3. Instance Lifecycle

An instance moves through a defined set of states:

```
   launch                 stop                start
     │                     │                    │
     ▼                     ▼                    ▼
 ┌─────────┐   boot    ┌─────────┐   ┌──────────┐   ┌─────────┐
 │ pending │ ────────▶ │ running │ ─▶│ stopping │ ─▶│ stopped │
 └─────────┘           └────┬────┘   └──────────┘   └────┬────┘
                            │                            │
                            │ terminate                  │ start → pending
                            ▼                            ▼
                     ┌─────────────┐                 back to running
                     │ shutting-   │
                     │ down →      │
                     │ terminated  │
                     └─────────────┘
```

| State | Billed for compute? | Notes |
|-------|--------------------|-------|
| `pending` | No | Booting up, being placed on a host |
| `running` | **Yes** | Fully operational |
| `stopping` / `stopped` | No (compute) | You still pay for **attached EBS volumes** and any Elastic IP not in use |
| `terminated` | No | Instance is gone permanently |

### Stop vs Terminate — and what happens to the root volume

This is a classic exam question.

| Action | Root EBS volume | Public IP | Private IP | Data |
|--------|-----------------|-----------|------------|------|
| **Stop** | Preserved (by default) | Released, new one on start | Kept | Persists |
| **Terminate** | **Deleted by default** (`DeleteOnTermination=true`) | Released | Released | **Gone** unless flag changed |

- ✅ **Stop** when you want to pause and resume later without losing the root disk. On restart the instance may land on a different host and gets a *new public IP* (unless you use an Elastic IP).
- ❌ **Terminate** is permanent. By default the root volume is deleted. Additional (non-root) EBS volumes default to *not* being deleted on termination — but always check the `DeleteOnTermination` flag.
- ⚠️ There's also **stop-hibernate**: RAM contents are written to the root EBS volume so the instance resumes faster with state intact. Requires an encrypted root volume and has size limits.

> **Rule**: Stop = pause (disk kept). Terminate = destroy (root disk deleted by default).

---

## 4. Instance Store (Ephemeral) vs EBS-Backed Storage

EC2 instances get their disks from one of two sources:

| | **Instance Store** | **EBS-backed** |
|--|---------------------|----------------|
| What it is | Physical disk **attached to the host** the instance runs on | Network-attached block storage (a separate service) |
| Persistence | **Ephemeral** — wiped on stop or terminate | **Persists** independently of the instance lifecycle |
| Performance | Very high IOPS / low latency (local NVMe) | High, but over the network |
| Survives stop? | ❌ No — data lost when you stop or terminate | ✅ Yes |
| Survives host failure? | ❌ No | ✅ Yes (replicated within the AZ) |
| Detach / reattach? | No | Yes — can move volume to another instance in same AZ |
| Snapshots | No | Yes (to S3) |

```
 EBS-backed:                          Instance Store:
   Instance ─── network ──▶ EBS         Instance ─── local bus ──▶ NVMe on host
   (volume lives on, survives           (disk dies with the host /
    stop, terminate, host failure)       on every stop or terminate)
```

- ✅ Use **EBS-backed** for almost everything — it's the default, supports stop/start, snapshots, and survives failures.
- 💡 Use **instance store** only for scratch data, caches, buffers, or replicated data (e.g. a node in a cluster that re-replicates) where speed matters and loss is acceptable.
- ⚠️ Instance-store-backed instances **cannot be stopped** — only rebooted or terminated. Stopping isn't an option because there's no persistent root volume to return to.

For the full EBS deep-dive (volume types, IOPS, snapshots), see [../05_storage/01_ebs.md](../05_storage/01_ebs.md).

---

## 5. How EC2 Connects to a VPC — ENIs

Every instance reaches the network through an **Elastic Network Interface (ENI)** — a virtual network card that lives in **one subnet** (and therefore one AZ).

```
   ┌────────────────────────── VPC ───────────────────────────┐
   │   Subnet (AZ-a, 10.0.1.0/24)                              │
   │     ┌─────────────────────────────────────────┐          │
   │     │  EC2 Instance                            │          │
   │     │   ┌──────────────┐    ┌──────────────┐   │          │
   │     │   │ eth0 (ENI)   │    │ eth1 (ENI)   │   │          │
   │     │   │ 10.0.1.25    │    │ 10.0.1.40    │   │          │
   │     │   │ + Security   │    │ + Security   │   │          │
   │     │   │   Group(s)   │    │   Group(s)   │   │          │
   │     │   └──────────────┘    └──────────────┘   │          │
   │     └─────────────────────────────────────────┘          │
   └──────────────────────────────────────────────────────────┘
```

Key facts about ENIs:
- Each ENI has a **private IPv4** from the subnet's range; can also have a **public IP**, an **Elastic IP**, MAC address, and one or more **security groups**.
- The **primary ENI (eth0)** can't be detached. Secondary ENIs **can** be detached and moved to another instance **in the same AZ** — useful for failover (move the ENI = move the IP/MAC to a standby).
- The number of ENIs and private IPs per instance is **capped by instance type** (bigger instances allow more).
- ENIs consume private IPs from the subnet, so subnet sizing limits scale for EC2 and for many managed services that also create ENIs.
- EC2 ENIs have a **source/destination check**. Leave it on for normal instances; disable it only for instances that forward traffic, such as NAT instances or firewall/router appliances.

> **This is where the firewall attaches**: a **Security Group** is attached to the ENI, not the instance directly. It's a *stateful* firewall — allow rules only, return traffic is automatically permitted. See [Security Groups vs NACLs](../03_networking/04_security_groups_vs_nacls.md).

For the cross-service ENI map, see
[ENIs, Security Groups & Service Networking](../03_networking/07_enis_security_groups_and_service_networking.md).

### Public vs private EC2 access

Public internet access to an EC2 instance needs **all three**:

1. The subnet route table has `0.0.0.0/0 -> IGW`.
2. The instance ENI has a public IPv4 address or Elastic IP.
3. SG and NACL rules allow the traffic.

Private EC2 instances should usually be reached through an **ALB**, **SSM
Session Manager**, a tightly controlled bastion, or private connectivity
(VPN/DX). For outbound access from private subnets, use a **NAT Gateway** for
internet/third-party APIs or **VPC endpoints** for supported AWS services.

---

## 6. Key Pairs and SSH Access

To log into a Linux instance you use an **SSH key pair** — public/private key cryptography:

- AWS stores the **public key**; you keep the **private key** (`.pem` file). AWS never has your private key, so **if you lose it, you cannot SSH in** with that key again.
- The public key is injected into the instance at first boot (into `~/.ssh/authorized_keys`) via the metadata service.
- Linux: SSH on **port 22** using the private key. Windows: the key decrypts the **administrator password** for RDP on **port 3389**.

```bash
chmod 400 my-key.pem
ssh -i my-key.pem ec2-user@<public-ip-or-dns>
```

⚠️ The security group must allow inbound 22 (SSH) or 3389 (RDP) from your IP, or the connection times out.

💡 Modern alternative: **EC2 Instance Connect** (browser/CLI, pushes a temporary key) and **SSM Session Manager** (no SSH port, no key pair, no bastion — access via the SSM agent + IAM). SSM Session Manager is the exam-preferred answer for "secure shell access without opening port 22 / without a bastion host."

---

## 7. Status Checks

EC2 runs two automated health checks every minute:

| Check | What it tests | Fix |
|-------|---------------|-----|
| **System status check** | The **underlying AWS host / infrastructure** (power, network, hardware) | **Stop and start** the instance → it moves to healthy hardware. (A reboot does *not* move hosts.) |
| **Instance status check** | The **instance's own software/config** (OS reachable, correct networking config, no exhausted memory/corrupt filesystem) | **Reboot or fix the OS** — it's your responsibility |

```
  System check  = AWS's problem  → stop/start to relocate the VM
  Instance check = your problem   → reboot / fix the guest OS
```

💡 You can create a **CloudWatch alarm** with an **EC2 recovery action** that automatically recovers (stop/start onto new hardware) an instance whose *system* status check fails — preserving the instance ID, private IP, Elastic IP, and metadata.

---

## Key Exam Points

- ✅ EC2 = IaaS virtual servers, billed per second (Linux/Windows), priced by instance type + size + region.
- ✅ Naming: `m5.large` = family `m` (general), gen `5`, size `large`. Know the five categories and what they optimize.
- ✅ **Stop** keeps the root EBS volume; **Terminate** deletes it by default.
- ✅ **Instance store** is ephemeral (lost on stop/terminate); **EBS** persists. Instance-store-backed instances can't be stopped.
- ✅ Security Groups attach to the **ENI** and are **stateful**; an instance lives in exactly one subnet/AZ per ENI.
- ✅ Public reachability requires an IGW route, a public IP/EIP, and firewall rules. A public IP alone is not enough.
- ✅ **System status check** failure → stop/start to relocate; **Instance status check** failure → reboot/fix OS.
- ✅ Lose the private key = lose SSH access via that key. SSM Session Manager = no key, no open port 22.
- ✅ Stopping changes the public IP (use an **Elastic IP** for a stable public address).

---

## Common Mistakes

- ❌ Thinking a **reboot** relocates an instance to new hardware — it does **not**. Only **stop/start** moves it to a new host.
- ❌ Assuming a single EC2 instance is highly available — it lives in one AZ; HA requires multiple AZs + load balancer/ASG.
- ❌ Forgetting that **stopped** instances still bill you for **attached EBS volumes** and idle Elastic IPs.
- ❌ Expecting data on **instance store** to survive a stop — it won't; only EBS does.
- ❌ Confusing the two status checks (system = AWS, instance = your OS).
- ❌ Putting the security group rule in but forgetting it's attached per-ENI, or that NACLs (stateless) might also block return traffic.

---

## Relevant Limits

- **vCPU-based On-Demand limits** (per region, per instance-family group) replace old per-instance-count limits; default is a few hundred vCPUs and is adjustable via Service Quotas.
- **ENIs and private IPs per instance**: capped by instance type (e.g. small types ~2–3 ENIs; large types many more).
- **Security groups per ENI**: up to **5** by default (raisable to 16); up to **60 inbound + 60 outbound** rules per SG.
- Instance sizes scale in powers of two: each size up roughly doubles vCPU and memory.

---

**Next**: [Part 2: EC2 Pricing Models](02_ec2_pricing_models.md)
