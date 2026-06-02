# Container Fundamentals

> **Who this is for**: Engineers preparing for SAA-C03 who need the container vocabulary
> *before* touching ECS, ECR, Fargate, or EKS. No Docker experience assumed. Everything in
> [02_ecs_ecr_fargate_eks.md](02_ecs_ecr_fargate_eks.md) refers back to the concepts defined
> here — read this first even if you "know Docker."

---

## 1. What a Container Is

A **container** is a way of packaging an application together with everything it needs to run —
code, runtime, system libraries, and configuration — into a single, isolated unit that runs the
same way on any machine with a container runtime.

The core trick is **OS-level isolation**, not virtualized hardware. A container shares the host's
**kernel**, but the operating system gives it its own private view of the filesystem, process
tree, network interfaces, and resource limits. To the process inside, it looks like it has the
whole machine to itself; in reality it's just an ordinary process on the host with fences around
it.

```
                    Host operating system (one shared kernel)
  ┌───────────────────────────────────────────────────────────────┐
  │                                                               │
  │   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐      │
  │   │ Container A  │   │ Container B  │   │ Container C  │      │
  │   │  app + libs  │   │  app + libs  │   │  app + libs  │      │
  │   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘      │
  │          │                  │                  │              │
  │      ┌───┴──────────────────┴──────────────────┴───┐          │
  │      │      Container runtime (e.g. Docker)         │         │
  │      └─────────────────────┬───────────────────────┘          │
  │                            │                                  │
  │                    Shared OS kernel                           │
  └───────────────────────────────────────────────────────────────┘
                            Physical / virtual host
```

Those "fences" are built from Linux kernel primitives — **namespaces** (isolate what a process
can *see*: PIDs, mounts, network) and **cgroups** (limit what a process can *use*: CPU, memory).
You don't manage these directly; the runtime does it for you.

> **Key insight**: a container is just a process (or group of processes) on the host kernel,
> wrapped in isolation. That single fact explains why containers start in milliseconds and pack
> densely — there's no second operating system to boot.

---

## 2. Containers vs Virtual Machines

Both isolate workloads, but at different layers. A **virtual machine (VM)** virtualizes
*hardware*: a hypervisor presents fake CPU, memory, and disk to each VM, and every VM runs a
**full guest OS** with its own kernel. A container virtualizes the *operating system*: many
containers share one kernel.

```
        VIRTUAL MACHINES                        CONTAINERS
  ┌────────┐ ┌────────┐ ┌────────┐      ┌────────┐ ┌────────┐ ┌────────┐
  │ App A  │ │ App B  │ │ App C  │      │ App A  │ │ App B  │ │ App C  │
  │ Libs   │ │ Libs   │ │ Libs   │      │ Libs   │ │ Libs   │ │ Libs   │
  │ Guest  │ │ Guest  │ │ Guest  │      └────────┘ └────────┘ └────────┘
  │  OS    │ │  OS    │ │  OS    │      ┌──────────────────────────────┐
  └────────┘ └────────┘ └────────┘      │     Container runtime        │
  ┌──────────────────────────────┐      ├──────────────────────────────┤
  │         Hypervisor           │      │   Shared host OS / kernel    │
  ├──────────────────────────────┤      ├──────────────────────────────┤
  │          Host OS             │      │          Hardware            │
  ├──────────────────────────────┤      └──────────────────────────────┘
  │          Hardware            │
  └──────────────────────────────┘
   3 full guest OSes (heavy)            no guest OS per workload (light)
```

| | **Virtual Machine** | **Container** |
|--|---------------------|----------------|
| Isolation boundary | Hardware (hypervisor) | OS kernel (namespaces + cgroups) |
| Guest OS | Full OS per VM | None — shares host kernel |
| Size | Gigabytes | Megabytes |
| Startup time | Seconds to minutes (boots an OS) | Milliseconds to seconds |
| Density per host | Tens | Hundreds to thousands |
| Isolation strength | Stronger (separate kernels) | Weaker (shared kernel) |
| Portability | OS image tied to hypervisor | Runs anywhere with a runtime |
| Typical AWS form | EC2 instance | ECS task / Kubernetes pod |

⚠️ Because containers share the host kernel, isolation is weaker than a VM's. For
**hard multi-tenant** isolation (running untrusted code from different customers on the same
host), AWS Fargate quietly runs each task in a lightweight micro-VM (Firecracker) so you still
get VM-grade isolation without managing VMs. You don't need to configure this — but it's why
Fargate is positioned as the secure default.

💡 They are not mutually exclusive. In AWS the common pattern is **containers running inside
VMs**: ECS tasks running on EC2 instances. You get VM-level tenancy boundaries plus
container-level density and speed.

---

## 3. Images and Layers

A **container image** is the read-only template a container is started from. It is the packaged
filesystem (your app, its dependencies, a minimal userland) plus metadata (the command to run,
exposed ports, environment defaults). A running **container** is an instance of an image — like
an object is an instance of a class.

Images are built in **layers**. Each instruction in a build (install packages, copy code, set
config) produces a new read-only layer stacked on the previous one. Layers are
**content-addressed and cached**: identical layers are stored once and shared across images.

```
  IMAGE = stack of read-only layers + a thin writable layer at runtime

  ┌─────────────────────────────┐  ← writable container layer (per running container)
  ├─────────────────────────────┤
  │  COPY ./app  /app           │  ← layer 4  (changes most often)
  ├─────────────────────────────┤
  │  RUN pip install -r reqs    │  ← layer 3
  ├─────────────────────────────┤
  │  RUN apt-get install python │  ← layer 2
  ├─────────────────────────────┤
  │  FROM ubuntu:22.04          │  ← layer 1  (base image, rarely changes)
  └─────────────────────────────┘
       lower layers shared / cached across images and rebuilds
```

This layering is why containers are cheap to store and ship: if only your app code changed, only
that top layer is rebuilt and re-uploaded; the base OS and dependency layers are reused from
cache.

> **Rule**: order your build from least-changing to most-changing (base OS → dependencies →
> application code) so a code change invalidates only the top layer, not the whole image.

Each image is identified by a **tag** (e.g. `myapp:1.4.0`, `myapp:latest`) and, immutably, by a
content **digest** (a SHA-256 hash). Tags can move; digests can't.

⚠️ `latest` is just a default tag, not a guarantee of "newest." Pin specific versions or digests
in production — relying on `latest` makes deployments non-reproducible.

---

## 4. Registries

A **container registry** is a server that stores and distributes images. You **push** images to
it from your build pipeline and **pull** them onto any host that needs to run them.

```
   developer / CI                 registry                    runtime hosts
  ┌──────────────┐   push       ┌──────────────┐   pull     ┌───────────────┐
  │  build image │ ───────────► │  myapp:1.4.0 │ ─────────► │  run container│
  │  (docker     │              │  myapp:1.3.0 │            │  (ECS task /  │
  │   build)     │              │  base layers │            │   k8s pod)    │
  └──────────────┘              └──────────────┘            └───────────────┘
```

- **Public registries** (e.g. Docker Hub, Amazon ECR Public) host base images and open-source
  software anyone can pull.
- **Private registries** host *your* images with authentication and access control. On AWS this
  is **Amazon ECR** (covered in the next file).

> **Key insight**: the registry is the hand-off point between "build" and "run." Build once, push
> the immutable artifact, then every environment (dev, staging, prod) pulls the *exact same*
> image. This is what makes "works on my machine" stop being a problem.

---

## 5. Why Containers

| Benefit | What it means | Why it matters for architecture |
|---------|---------------|----------------------------------|
| **Portability** | The image bundles all dependencies | Same artifact runs on a laptop, CI, and AWS — no environment drift |
| **Density** | No per-workload OS overhead | Pack many more workloads per host than VMs → lower cost |
| **Fast start / stop** | No OS to boot, just a process | Scale out in seconds, scale to zero, fast deploys and rollbacks |
| **Immutability** | Images don't change after build | Reproducible deploys; roll back by pointing at the previous image |
| **Consistency** | Dev and prod run the identical image | Eliminates "works on my machine" |

These properties are exactly what microservices and elastic, auto-scaling architectures need:
small, independently deployable units that start fast and pack densely.

✅ Good fit: stateless web/API services, microservices, batch and CI jobs, anything you want to
scale horizontally and deploy frequently.

❌ Poor fit (or needs care): workloads needing strong kernel isolation between tenants without
Fargate's micro-VMs, or stateful apps that assume a fixed local disk — containers are ephemeral,
so persistent state must live elsewhere (EBS/EFS/S3/a database).

---

## 6. The Orchestration Problem

Running one container by hand is easy. Running *hundreds* across *many hosts*, reliably, is not.
The moment you go past a single host you hit a set of problems collectively called
**orchestration**:

- **Scheduling / placement** — given a fleet of hosts with finite CPU and memory, *which host*
  should each container run on? How do you bin-pack efficiently and respect constraints?
- **Scaling** — automatically add and remove container copies as load changes, and add/remove
  hosts when the fleet runs out of room.
- **Health & self-healing** — detect crashed or unhealthy containers and restart or reschedule
  them; replace failed hosts.
- **Networking & service discovery** — give containers stable addresses, route traffic to
  healthy ones, and let services find each other even as containers come and go.
- **Rollouts & rollbacks** — deploy new image versions gradually, watch health, and roll back
  cleanly on failure.
- **Configuration & secrets** — inject config and credentials without baking them into images.

```
  desired state                orchestrator                 actual state
  "run 5 copies of    ──────►  ┌─────────────┐   places &   ┌───────────────┐
   myapp:1.4.0 behind          │  scheduler  │   monitors   │ host1: 2 tasks│
   a load balancer"            │  + control  │ ───────────► │ host2: 2 tasks│
                               │    loop     │              │ host3: 1 task │
   one task crashes  ◄─────────│  reconciles │ ◄─────────── │  (1 died → 4) │
   → reschedule a 5th          └─────────────┘   observes   └───────────────┘
```

> **Key insight**: an orchestrator is a control loop. You declare a **desired state** ("5 healthy
> copies behind this load balancer"); it continuously compares that to the **actual state** and
> takes action to close the gap. You stop managing individual containers and start managing intent.

This is the problem **ECS** and **EKS (Kubernetes)** solve on AWS, and the work **Fargate**
removes from you entirely on the host side. That's the subject of the next file.

---

## 7. The Docker Mental Model

**Docker** is the toolchain that popularized containers. You don't need deep Docker skill for the
exam, but you need its vocabulary because AWS reuses it.

```
  Dockerfile  ──build──►  Image  ──push──►  Registry  ──pull──►  Container (running)
  (recipe)               (artifact)        (ECR/Hub)            (process on a host)
```

| Term | Meaning |
|------|---------|
| **Dockerfile** | Text recipe of build steps; each step becomes a layer |
| **Image** | Built, immutable artifact produced from a Dockerfile |
| **Container** | A running instance of an image |
| **Registry** | Stores and serves images (push/pull) |
| **Runtime** | The engine that runs containers from images on a host |

A minimal Dockerfile, just to make the layering concrete:

```dockerfile
# Least-changing first → best layer caching
FROM python:3.12-slim          # base layer
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt   # dependency layer (cached unless reqs change)
COPY . .                              # app code layer (changes most often)
EXPOSE 8080
CMD ["python", "main.py"]             # default command when the container starts
```

> **Note**: AWS doesn't require Docker specifically — ECS and EKS run any OCI-compliant image
> (OCI is the open standard Docker images conform to). "Docker image" and "OCI image" are
> interchangeable for the exam.

---

## Key Exam Points

- A **container shares the host kernel**; a **VM runs a full guest OS** via a hypervisor.
  Containers are smaller, start faster, and pack denser; VMs give stronger isolation.
- Containers are **ephemeral and stateless by design** — persistent state must live in
  EBS/EFS/S3/a database, never assumed on local container disk.
- An **image** is the immutable template; a **container** is a running instance of it. Images are
  built in cached **layers**.
- A **registry** (Amazon **ECR** on AWS) is where images are pushed and pulled — the hand-off
  between build and run.
- **Orchestration** = scheduling, scaling, self-healing, networking/discovery, and rollouts
  across many hosts. AWS solves this with **ECS** and **EKS**; **Fargate** removes host
  management.
- AWS runs any **OCI/Docker image** — the build tool is not exam-relevant; the model is.

---

## Common Mistakes

- ❌ Thinking containers are "lightweight VMs." They isolate at the OS layer and share the kernel —
  a fundamentally different and weaker boundary.
- ❌ Treating containers as durable. Anything written to the container's writable layer is lost
  when it stops or is rescheduled.
- ❌ Relying on the `latest` tag in production — it's mutable and breaks reproducibility. Pin
  versions or digests.
- ❌ Ordering a Dockerfile with app code before dependencies, which busts the cache on every code
  change and slows builds.
- ❌ Assuming you must run your own orchestrator — that's exactly what ECS/EKS/Fargate exist to
  avoid.

---

**Next**: [02_ecs_ecr_fargate_eks.md — ECR, ECS, Fargate & EKS](02_ecs_ecr_fargate_eks.md)
