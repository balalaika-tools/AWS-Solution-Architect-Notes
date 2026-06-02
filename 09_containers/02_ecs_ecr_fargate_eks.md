# ECR, ECS, Fargate & EKS — The AWS Container Stack

> **Who this is for**: Engineers preparing for SAA-C03 who understand the container model
> (images, registries, orchestration) from
> [01_container_fundamentals.md](01_container_fundamentals.md) and now need the AWS services that
> implement it: where images live, what runs them, and how to choose between the options on the
> exam.

---

## 1. The Stack at a Glance

Three layers map directly onto the fundamentals from the previous file:

```
  REGISTRY            ORCHESTRATION (control plane)        COMPUTE (where tasks run)
  ┌──────────┐        ┌──────────────┬──────────────┐      ┌──────────────┬──────────────┐
  │   ECR    │        │     ECS      │     EKS       │      │  EC2 launch  │   Fargate    │
  │ (images) │ ─pull─►│  (AWS-native │ (Kubernetes,  │      │  (you manage │ (serverless, │
  │          │        │ orchestrator)│  managed)     │      │   the nodes) │  no nodes)   │
  └──────────┘        └──────────────┴──────────────┘      └──────────────┴──────────────┘
                          choose ONE orchestrator              choose a compute mode
                                                               (each orchestrator works
                                                                with EC2 or Fargate)
```

> **Key insight**: **registry**, **orchestrator**, and **compute** are three independent choices.
> ECR is almost always the registry. The two big decisions are *ECS vs EKS* (which orchestrator)
> and *EC2 vs Fargate* (who manages the servers). They are orthogonal — Fargate works under both
> ECS and EKS.

---

## 2. Amazon ECR — Elastic Container Registry

**Amazon ECR** is AWS's fully managed, private **container registry** — the AWS implementation of
the registry concept from the previous file. It's where your built images live before ECS, EKS,
Lambda, or App Runner pull them.

Core characteristics:

- **Private by default**, with access controlled by **IAM** (and optional repository policies for
  cross-account access). There is also **ECR Public** for sharing images openly.
- Images stored in **S3** under the hood and encrypted at rest (SSE-S3 or KMS).
- Integrates with the whole AWS container ecosystem — ECS/EKS pull images using the task/pod's
  IAM permissions; no long-lived registry passwords.

**Image scanning** — ECR scans images for known CVEs (Common Vulnerabilities and Exposures):

| Scan type | Engine | When it runs |
|-----------|--------|--------------|
| **Basic scanning** | open-source CVE database (Clair) | on push or manually |
| **Enhanced scanning** | Amazon Inspector | continuously, OS *and* programming-language packages |

**Lifecycle policies** — rules that automatically expire old images so a repository doesn't grow
forever (and rack up storage cost):

```
Example lifecycle policy intent:
  "Keep the last 10 tagged 'prod-*' images; expire untagged images older than 7 days."
```

✅ Use lifecycle policies on every repository — CI pushes images constantly, and untagged image
layers accumulate quickly.

💡 ECR push/pull goes over the public AWS API endpoints by default. To keep traffic private (and
avoid NAT charges for tasks in private subnets), use **VPC interface endpoints** for ECR plus an
**S3 gateway endpoint** for the layer downloads.

For private-subnet ECS/Fargate tasks pulling from ECR without NAT, the practical
minimum is usually:

- interface endpoints for `ecr.api` and `ecr.dkr`,
- an **S3 gateway endpoint** for image layer downloads,
- a CloudWatch Logs interface endpoint if the task writes logs,
- Secrets Manager or SSM Parameter Store interface endpoints if the task injects secrets.

The endpoint security groups must allow inbound HTTPS (`443`) from the task SG.

---

## 3. Amazon ECS — Elastic Container Service

**Amazon ECS** is AWS's own (non-Kubernetes) container orchestrator. It is deeply integrated with
the rest of AWS (IAM, ALB, CloudWatch, VPC) and has no control-plane cost. You describe what you
want to run; ECS schedules, monitors, and heals it.

### Core objects

| Object | What it is |
|--------|------------|
| **Cluster** | A logical grouping of compute (EC2 instances and/or Fargate capacity) where tasks run |
| **Task definition** | The blueprint (JSON): which image(s), CPU/memory, ports, env vars, IAM roles, log config, volumes. Like a `docker run` spec, versioned. |
| **Task** | A running instance of a task definition — one or more containers scheduled together |
| **Service** | Keeps a desired number of tasks running, replaces failed ones, and integrates with a load balancer and auto scaling |

```
  ECS CLUSTER
  ┌────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │   SERVICE "web"  (desired count = 3, behind ALB)                 │
  │   ┌───────────────────────────────────────────────────────┐    │
  │   │   Task        Task        Task     ◄── all from the     │    │
  │   │ ┌───────┐   ┌───────┐   ┌───────┐     same task         │    │
  │   │ │ nginx │   │ nginx │   │ nginx │     definition rev    │    │
  │   │ │  app  │   │  app  │   │  app  │                       │    │
  │   │ └───────┘   └───────┘   └───────┘                       │    │
  │   └─────▲─────────▲─────────────▲──────────────────────────┘    │
  │         │         │             │   target group (health checks)│
  └─────────┼─────────┼─────────────┼───────────────────────────────┘
            │         │             │
        ┌───┴─────────┴─────────────┴───┐
        │   Application Load Balancer    │ ◄── internet / clients
        └────────────────────────────────┘
```

### The ALB integration

An ECS **service** registers and deregisters its tasks in an **ALB target group** automatically
as tasks start, stop, or get rescheduled. Combined with **dynamic port mapping** (on EC2 launch
type) the ALB routes only to healthy tasks and supports path/host-based routing to multiple
services. This is the canonical web-app pattern.

For Fargate or any service using `awsvpc`, the target group type is usually
**IP**, because the task has its own ENI and private IP. The common pattern is:

```
internet -> public ALB -> private Fargate task ENIs

ALB SG:  allow 443 from clients
Task SG: allow container port from ALB SG
```

Do not open the task SG to the internet. The ALB is public; the tasks stay in
private subnets.

See the load-balancer details in
[../07_ha_scaling/02_elb_alb_nlb_gwlb.md](../07_ha_scaling/02_elb_alb_nlb_gwlb.md), and a full
worked deployment in
[../18_practical_examples/13_ecs_fargate_behind_alb.md](../18_practical_examples/13_ecs_fargate_behind_alb.md).

> **Rule**: ECS handles *placement and health* (the orchestration loop); the **ALB** handles
> *traffic distribution and health checks at the request layer*. They cooperate but solve
> different problems.

---

## 4. Launch Types — EC2 vs Fargate

ECS (and EKS) must run tasks on *some* compute. The **launch type** decides who owns and manages
that compute.

- **EC2 launch type** — you run a fleet of EC2 instances (the ECS-optimized AMI runs the ECS
  agent) and register them with the cluster. ECS packs tasks onto your instances. **You** patch,
  scale, and secure the instances.
- **Fargate launch type** — **serverless containers**. You never see or manage a server. You
  specify CPU and memory per task; AWS provisions the right-sized, isolated compute (each task in
  its own Firecracker micro-VM), runs it, and bills per task per second.

```
   EC2 LAUNCH TYPE                          FARGATE
  ┌──────────────────────────┐            ┌──────────────────────────┐
  │ EC2 instance (you own)   │            │      (no instance you     │
  │  ┌──────┐ ┌──────┐       │            │       can see)            │
  │  │ task │ │ task │ ◄ ECS │            │   ┌──────┐                │
  │  └──────┘ └──────┘  packs│            │   │ task │ ◄ AWS provisions│
  │   ECS agent, you patch   │            │   └──────┘   micro-VM,     │
  │   the OS, you scale nodes│            │   right-sized, isolated    │
  └──────────────────────────┘            └──────────────────────────┘
```

| | **EC2 launch type** | **Fargate** |
|--|---------------------|-------------|
| Server management | You manage EC2 (patching, scaling, AMIs, security) | None — fully serverless |
| Billing | Per EC2 instance (running 24/7, even if idle) | Per task vCPU + memory, per second |
| Right-sizing | You bin-pack tasks onto instances | AWS sizes each task |
| Isolation | Tasks may share an instance | Each task in its own micro-VM |
| Best for | Steady, high-density, cost-tuned workloads; need GPU/special instances/daemon access | Spiky/variable load, ops-light teams, fewer/larger tasks, "just run my container" |
| Control | Full host control (instance type, GPUs, host volumes) | No host access; only what the task definition exposes |

✅ Default to **Fargate** unless you have a concrete reason (cost at scale with high steady
density, GPU/Inferentia, special instance types, or workloads needing host-level access) to
manage EC2.

⚠️ Fargate is *not always cheaper*. At high, predictable, densely-packed utilization, EC2 (with
Savings Plans/Spot) can win because you pay for the instance, not per-task overhead. The exam
favors Fargate for "minimize operational overhead," and EC2 for "maximize cost efficiency at
steady high scale / need specific hardware."

💡 Both launch types support **Spot** for big savings on fault-tolerant workloads (Fargate Spot,
EC2 Spot capacity providers).

---

## 5. Amazon EKS — Elastic Kubernetes Service

**Amazon EKS** is managed **Kubernetes**. If your team already runs Kubernetes, wants the
Kubernetes API and its huge ecosystem (Helm, operators, CRDs), or needs portability across
clouds/on-prem, EKS gives you upstream-conformant Kubernetes without operating the control plane
yourself.

- **Control plane** — AWS runs the Kubernetes API servers and `etcd` across multiple AZs, manages
  upgrades and patching, and bills a flat hourly fee per cluster. (Unlike ECS, the EKS control
  plane is **not free**.)
- **Worker nodes** — where your **pods** run. Three options:
  - **Self-managed nodes** — you run and manage the EC2 instances yourself.
  - **Managed node groups** — AWS provisions and lifecycle-manages EC2 nodes for you (still EC2
    you can see, but with automation).
  - **Fargate on EKS** — run pods on serverless Fargate compute, no nodes at all (with some
    constraints, e.g. DaemonSets and certain pod features aren't supported).

```
  EKS CLUSTER
  ┌──────────────────────────────────────────────────────────────┐
  │  CONTROL PLANE  (AWS-managed, multi-AZ, flat hourly fee)       │
  │   ┌──────────────┐  ┌──────┐                                  │
  │   │ kube-apiserver│  │ etcd │   ← you never touch these        │
  │   └──────────────┘  └──────┘                                  │
  └───────────────────────────┬──────────────────────────────────┘
                              │ schedules pods onto:
        ┌────────────────────┴─────────────────────┐
        │                                            │
  ┌─────────────┐   ┌─────────────┐         ┌──────────────────┐
  │ Managed node│   │ Self-managed│   OR    │  Fargate on EKS  │
  │ group (EC2) │   │ nodes (EC2) │         │  (no nodes)      │
  │  ┌────┐┌───┐│   │  ┌────┐     │         │   ┌────┐         │
  │  │pod ││pod││   │  │pod │     │         │   │pod │         │
  │  └────┘└───┘│   │  └────┘     │         │   └────┘         │
  └─────────────┘   └─────────────┘         └──────────────────┘
```

> **Choose EKS over ECS when**: you already use Kubernetes, need multi-cloud/on-prem portability,
> require the Kubernetes ecosystem (Helm, operators, advanced scheduling), or have a team with
> Kubernetes skills. Otherwise the operational simplicity of ECS usually wins.

---

## 6. ECS vs EKS vs Fargate — Decision

A frequent source of confusion: **ECS and EKS are orchestrators; Fargate is a compute mode.**
Fargate is *not* an alternative to ECS/EKS — it runs *under* them. The real fork is:

```
        Need to run containers on AWS
                    │
        ┌───────────┴────────────┐
   Need Kubernetes API /     No Kubernetes
   ecosystem / portability?  requirement?
        │                         │
       EKS                       ECS         ◄── pick the ORCHESTRATOR
        │                         │
        └───────────┬─────────────┘
                    │
        Want to manage servers?
        ┌───────────┴────────────┐
       Yes                        No
        │                         │
     EC2 nodes                 Fargate        ◄── pick the COMPUTE MODE
   (cost/control)           (serverless,
                            low ops)
```

| | **ECS** | **EKS** | **Fargate** |
|--|---------|---------|-------------|
| Category | AWS-native orchestrator | Managed Kubernetes orchestrator | Serverless compute mode (used by ECS or EKS) |
| Control-plane cost | None | Flat hourly fee per cluster | n/a (you pay per task/pod resources) |
| Learning curve | Low (AWS concepts) | High (Kubernetes) | n/a |
| Portability | AWS-only | Portable (standard k8s) | AWS-only |
| Server management | EC2 or Fargate | EC2 or Fargate | None |
| Pick it when | Simple, AWS-centric, low ops | Already on k8s / need its ecosystem / portability | You don't want to manage nodes under either |

💡 Exam shorthand: "least operational overhead, AWS-only" → **ECS on Fargate**. "Existing
Kubernetes / multi-cloud" → **EKS**. "Minimize cost at steady high scale / need GPUs" → **EC2
launch type** (under ECS or EKS).

---

## 7. ECS Networking, IAM, and Auto Scaling

### Networking modes (and why `awsvpc` matters)

ECS tasks can use different network modes; the one to know is **`awsvpc`**:

| Mode | Behavior |
|------|----------|
| **`awsvpc`** | Each task gets its **own ENI** with a private IP in your VPC subnet and its **own security group**. Required for Fargate; recommended for EC2. |
| `bridge` | Docker bridge network on the host (EC2 only); uses dynamic port mapping behind an ALB. |
| `host` | Task uses the host's network interface directly (EC2 only). |
| `none` | No external networking. |

✅ **`awsvpc`** gives each task first-class VPC networking — it's addressable, firewalled by its
own security group, and shows up like an EC2 ENI. This is mandatory on Fargate and the standard
choice everywhere.

For Fargate, the service or run-task request must provide subnets and security
groups. Each task gets one task ENI, so subnet free IPs and ENI-related quotas
can become scaling limits before CPU or memory do.

### EKS networking notes

EKS on EC2 normally uses the **Amazon VPC CNI**. Pods receive VPC IP addresses
from node ENIs, so pod density is constrained by subnet IP space and the ENI/IP
limits of the node instance type. By default, pod traffic uses the node's
security groups. With **Security Groups for Pods**, selected pods can get more
granular SGs through additional VPC CNI plumbing.

The EKS cluster API endpoint can be public, private, or both. Public/private
endpoint configuration controls access to the Kubernetes API, not whether pods
themselves are in public or private subnets.

For the broader ENI/security-group map, see
[ENIs, Security Groups & Service Networking](../03_networking/07_enis_security_groups_and_service_networking.md).

### Task roles — two distinct IAM roles

| Role | Who uses it | Purpose |
|------|-------------|---------|
| **Task execution role** | The ECS agent / Fargate | Pull the image from **ECR**, fetch secrets, write logs to **CloudWatch** — infrastructure-level permissions |
| **Task role** | Your application code inside the container | Grants the app's containers AWS permissions (e.g. read an S3 bucket, write to DynamoDB) |

> **Rule**: never bake AWS credentials into an image or env var. Assign a **task role** so the
> container gets temporary, automatically-rotated credentials scoped to exactly what it needs —
> the container equivalent of an EC2 instance profile.

### Service Auto Scaling

ECS services scale on two axes:

- **Service auto scaling** — adjusts the **task count** using Application Auto Scaling. Policies:
  **target tracking** (e.g. keep average CPU at 60%, or ALB requests-per-target at N), **step
  scaling**, and **scheduled scaling** — the same model as EC2 Auto Scaling groups (see
  [../07_ha_scaling/README.md](../07_ha_scaling/README.md)).
- **Cluster capacity (EC2 launch type only)** — **capacity providers** add/remove EC2 instances
  so there's room for the tasks. On **Fargate this layer disappears** — there are no instances to
  scale.

💡 This two-layer scaling is a key Fargate selling point: you scale tasks and AWS handles the
underlying capacity automatically.

---

## 8. AWS App Runner (Brief Mention)

**AWS App Runner** is the most hands-off option: point it at a container image in **ECR** (or even
source code) and it builds, deploys, load-balances, auto-scales (including scale-to-zero), and
serves it over HTTPS — no clusters, task definitions, load balancers, or networking to configure.

✅ Best for simple, stateless **web apps and APIs** where you want a fully managed
"container-to-URL" experience. ❌ Not for complex multi-service orchestration, batch, or anything
needing fine-grained control — reach for ECS/EKS then.

> Mental model of operational overhead, lowest to highest:
> **App Runner → ECS on Fargate → ECS on EC2 → EKS on Fargate → EKS on EC2.**

---

## Key Exam Points

- **ECR** = private image registry; supports **image scanning** (basic via Clair, enhanced via
  Inspector) and **lifecycle policies** to expire old images. Access via IAM. Use VPC endpoints to
  keep pulls private.
- **ECS** objects: **cluster** → **service** → **task** ← **task definition**. Services keep
  desired task count, self-heal, and register tasks in an **ALB target group**.
- **EC2 launch type** = you manage instances (more control, can be cheaper at steady high scale,
  GPUs). **Fargate** = serverless, no servers, pay per task — choose for **least operational
  overhead**.
- **ECS vs EKS is the orchestrator choice; Fargate is a compute mode under either.** EKS for
  existing Kubernetes / portability / ecosystem (and it has a control-plane fee); ECS for simple,
  AWS-native, low-ops.
- **`awsvpc`** mode gives each task its own ENI + security group (mandatory on Fargate).
- Private Fargate image pulls need NAT or the ECR API/ECR DKR interface endpoints plus an S3
  gateway endpoint; add logs/secrets endpoints as needed.
- EKS VPC CNI makes pods consume VPC IPs; default pod security is usually node SGs unless SG for
  Pods is enabled.
- Two IAM roles: **task execution role** (pull image, secrets, logs) vs **task role** (app's AWS
  permissions). Never embed credentials in the image.
- **Service auto scaling** scales task count (target tracking / step / scheduled); **capacity
  providers** scale EC2 nodes (not needed on Fargate).
- **App Runner** = simplest path for a single containerized web app/API.

---

## Common Mistakes

- ❌ Treating Fargate as an alternative *to* ECS/EKS. It's a compute mode that runs *under* them.
- ❌ Confusing the **task execution role** (infrastructure: pull image, write logs) with the
  **task role** (your app's AWS access). The exam tests this distinction.
- ❌ Assuming Fargate is always cheapest — at high steady density with Savings Plans/Spot, EC2 can
  be cheaper.
- ❌ Reaching for **EKS** when there's no Kubernetes requirement — you pay a control-plane fee and
  take on Kubernetes complexity for nothing. Default to ECS.
- ❌ Baking AWS credentials into images or env vars instead of using a task role.
- ❌ Forgetting ECR **lifecycle policies** and letting untagged image layers (and storage cost)
  pile up.
- ❌ Picking `bridge`/`host` networking on Fargate — only **`awsvpc`** is supported there.

---

**Next**: [../10_serverless/01_lambda.md — AWS Lambda](../10_serverless/01_lambda.md)
