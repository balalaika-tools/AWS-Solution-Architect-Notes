# Load Balancing Concepts

> **Who this is for**: Engineers prepping for SAA-C03 who want the mental model behind load balancing *before* meeting the AWS products. Networking-light readers welcome — OSI layers, listeners, and health checks are all explained from scratch. Before reading this, understand what an instance is: **[EC2 Fundamentals](../04_compute/01_ec2_fundamentals.md)**.

---

## 1. Why Distribute Traffic at All

A single server is three bad things at once:

- A **single point of failure** — if it dies, the whole app is down.
- A **capacity ceiling** — it can only handle so many requests before it slows or crashes.
- A **deployment bottleneck** — you can't update it without downtime.

The fix is to run **multiple identical servers** and put something in front that spreads requests across them. That "something" is a **load balancer (LB)**.

```
                    ┌──────────────┐
   clients ───────▶ │ Load Balancer│
                    └──────┬───────┘
            ┌──────────────┼──────────────┐
            ▼              ▼               ▼
        ┌────────┐    ┌────────┐     ┌────────┐
        │ Server │    │ Server │     │ Server │   ← if one dies, the LB
        │   A    │    │   B    │     │   C    │     stops sending to it
        └────────┘    └────────┘     └────────┘
```

A load balancer buys you three properties at once:

| Property | What it means | How the LB delivers it |
|----------|---------------|------------------------|
| **Availability** | The app stays up despite failures | Stops routing to unhealthy targets; spreads across Availability Zones |
| **Scalability** | Handle more load by adding servers | New servers register behind the LB and start receiving traffic |
| **No single point of failure** | No one box whose death = outage | Many targets; the LB itself is a managed, redundant fleet |

> **Key insight**: A load balancer is the *seam* that lets you treat a fleet of servers as one logical service. Clients see one address; behind it, servers come and go freely.

---

## 2. Horizontal vs Vertical Scaling

There are two ways to give an app more capacity. Load balancing only makes sense once you understand the difference.

```
   VERTICAL (scale up)              HORIZONTAL (scale out)
   ─────────────────────           ──────────────────────
   ┌──────┐   ┌────────┐           ┌──┐    ┌──┐ ┌──┐ ┌──┐
   │ t3.  │ → │ m5.    │           │t3│ →  │t3│ │t3│ │t3│
   │ micro│   │ 4xlarge│           └──┘    └──┘ └──┘ └──┘
   └──────┘   └────────┘
   bigger box                      more boxes
```

| | Vertical (scale up) | Horizontal (scale out) |
|---|---|---|
| **How** | Resize to a bigger instance | Add more instances |
| **Ceiling** | Limited by the largest instance type | Effectively unlimited |
| **Downtime** | Usually requires a stop/start | None — add instances live |
| **Resilience** | Still a single point of failure | Survives losing any one node |
| **Needs a load balancer?** | No | **Yes** — something must spread traffic |

> **Rule**: The exam strongly favors **horizontal scaling** for availability and elasticity. Vertical scaling is the answer mainly for workloads that *can't* be parallelized (e.g., a single large relational database primary). Load balancers exist to make horizontal scaling possible.

⚠️ Horizontal scaling only works if your servers are **stateless** — any server can handle any request. State (sessions, uploads) must live in a shared store (a database, ElastiCache, S3), not on the server's local disk. See [sticky sessions](#6-sticky-sessions) for the workaround when you can't.

---

## 3. What a Load Balancer Actually Is

A load balancer is a device (in AWS, a **managed, auto-scaling fleet** of them) that:

1. **Accepts** incoming connections on one or more addresses/ports.
2. **Chooses** a healthy backend server for each connection or request.
3. **Forwards** the traffic, often rewriting or terminating the connection.
4. **Monitors** backends and removes unhealthy ones from rotation.

The clients never talk to the servers directly. They talk to the load balancer, which acts as a **reverse proxy** (or, at lower layers, a packet/connection router) standing in front of the fleet.

💡 In AWS the load balancer is a **managed service** — you don't patch it, scale it, or worry about its own availability. You configure it; AWS runs the redundant infrastructure across multiple AZs.

---

## 4. OSI Layer 4 vs Layer 7

Load balancers operate at one of two levels of the network stack. This single distinction drives almost every AWS load-balancer decision, so it's worth getting right.

The OSI model is a 7-layer abstraction of networking. You only need two of them:

```
  Layer 7  APPLICATION   ── HTTP, HTTPS, gRPC, WebSocket
  Layer 6  Presentation                         ▲
  Layer 5  Session                               │  "Layer 7" LBs see this
  Layer 4  TRANSPORT     ── TCP, UDP, TLS  ◀──────┤
  Layer 3  Network       ── IP packets            │  "Layer 4" LBs see this
  Layer 2  Data Link                              ▼
  Layer 1  Physical
```

| | Layer 4 (Transport) | Layer 7 (Application) |
|---|---|---|
| **Sees** | IP addresses + ports, TCP/UDP segments | Full HTTP request: URL, headers, cookies, host |
| **Can route on** | Destination port only | Path (`/api`), hostname, headers, method |
| **Understands content?** | No — it's just moving bytes | Yes — it parses the request |
| **Latency** | Lower (no parsing) | Slightly higher (parses each request) |
| **Throughput** | Extremely high (millions/sec) | High, but does more work per request |
| **Example feature** | "Send TCP port 443 to this pool" | "Send `GET /images/*` to the image fleet" |
| **AWS product** | Network Load Balancer (NLB) | Application Load Balancer (ALB) |

> **Key insight**: **Layer 4 = dumb and fast** (moves connections by IP/port). **Layer 7 = smart and feature-rich** (reads HTTP and routes on its contents). If a question mentions routing by URL path or hostname, it's Layer 7 / ALB. If it mentions TCP/UDP, static IPs, or extreme performance, it's Layer 4 / NLB.

---

## 5. The Building Blocks: Listeners, Target Groups, Health Checks

Modern AWS load balancers (ALB, NLB, GWLB) share the same three core objects. Learn them once.

```
   ┌──────────────────── Load Balancer ────────────────────┐
   │                                                        │
   │   LISTENER (port 443, HTTPS)                           │
   │      │                                                 │
   │      │  rule: if path = /api  ──▶ TARGET GROUP "api"   │
   │      │  rule: default         ──▶ TARGET GROUP "web"   │
   │                                                        │
   └────────────────────────┬───────────────────┬──────────┘
                             ▼                   ▼
                    TARGET GROUP "web"   TARGET GROUP "api"
                     ┌──┐ ┌──┐ ┌──┐        ┌──┐ ┌──┐
                     │EC2│ │EC2│ │EC2│       │EC2│ │EC2│
                     └──┘ └──┘ └──┘        └──┘ └──┘
                       ▲ health checks ▲
```

### Listener

A **listener** is the front door: a protocol + port the LB listens on (e.g., HTTPS:443). It holds **rules** that decide where matching traffic goes. An LB can have multiple listeners (e.g., one for 80, one for 443).

### Target group

A **target group** is a named pool of backends that receive traffic. Targets can be:

| Target type | What it is | Used by |
|-------------|-----------|---------|
| **Instance** | EC2 instances (by instance ID) | ALB, NLB |
| **IP** | Any IP in the VPC (or on-prem via VPN/DX) | ALB, NLB |
| **Lambda** | A Lambda function | ALB only |
| **ALB** | Another ALB (NLB forwards to an ALB) | NLB only |

A target group also defines the **health check** and the **deregistration delay** (below).

### Health checks

The LB periodically probes each target. If a target fails the configured number of checks, the LB marks it **unhealthy** and stops sending it traffic — automatically, with no human involved. When it passes again, it returns to rotation.

```
  LB ──"GET /health"──▶ target
        │
        ├─ 200 OK   ×2 in a row  ──▶ Healthy   (gets traffic)
        └─ timeout  ×3 in a row  ──▶ Unhealthy (no traffic)
```

| Health check setting | Meaning |
|----------------------|---------|
| **Protocol / port / path** | What to probe (e.g., `HTTP :80 /health`) |
| **Healthy threshold** | Consecutive passes to mark healthy |
| **Unhealthy threshold** | Consecutive fails to mark unhealthy |
| **Interval** | Seconds between checks |
| **Timeout** | How long to wait for a response |
| **Success codes** | HTTP codes that count as healthy (e.g., `200-299`) |

💡 Point health checks at a **dedicated lightweight endpoint** (e.g., `/health`) that verifies the app's real dependencies, not just that the web server is up. A naive check on `/` can report healthy while the database connection is broken.

Health checks are still network traffic. For an ALB, the ALB security group must
allow outbound traffic to the target's listener/health-check port, and the
target security group should allow inbound traffic **from the ALB security
group** on those ports. Do not open private targets to `0.0.0.0/0` just because
the load balancer is public.

---

## 6. Connection Draining / Deregistration Delay

When a target is removed — because it failed a health check, you're deploying, or Auto Scaling is terminating it — the LB doesn't cut its in-flight requests off mid-stream. Instead it **stops sending new requests** but lets existing ones finish, up to a timeout.

```
   t=0   instance marked for removal
         │  LB stops routing NEW requests to it
         │  existing requests keep draining ────────┐
   t=30  deregistration delay (default 300s) expires │
         │  any still-open connections are closed    ▼
         └─ instance fully deregistered / can terminate
```

- On **ALB/NLB** this is the **deregistration delay** (target group setting, default **300 s**, range 0–3600).
- On the legacy Classic LB it's called **connection draining** (same idea).

⚠️ Set this **lower** for fast-cycling stateless apps (so scale-in is quick) and **higher** for long-lived connections (file uploads, WebSockets) so you don't kill active users.

---

## 7. Sticky Sessions

By default the LB may send a given client's successive requests to **different** targets. If your app stores session state in a server's local memory, the user gets logged out when routed elsewhere. **Sticky sessions** (a.k.a. **session affinity**) pin a client to one target.

```
  Without stickiness:           With stickiness:
  req1 ─▶ Server A              req1 ─▶ Server A
  req2 ─▶ Server C              req2 ─▶ Server A   (cookie pins it)
  req3 ─▶ Server B              req3 ─▶ Server A
```

- **ALB** uses a cookie. Either an **LB-generated cookie** (`AWSALB`) or an **application-defined cookie**. Duration is configurable.
- **NLB** can do stickiness based on the **source IP** (no cookies at layer 4).

| ✅ Good use | ❌ Anti-pattern |
|------------|----------------|
| Legacy app that *must* keep session in local memory | Using stickiness as a permanent design instead of externalizing state |

> **Rule**: Stickiness is a **crutch**, not a feature to design around. The exam-preferred architecture is **stateless** servers with session state in **ElastiCache / DynamoDB**, so any target can serve any request and scaling is frictionless. Reach for stickiness only when refactoring isn't an option.

---

## 8. The Request Flow End to End

Putting it together — a client request through an HTTPS Application Load Balancer:

```
  ┌────────┐    1. DNS resolves the LB's DNS name to an IP
  │ Client │
  └───┬────┘
      │ 2. opens TCP, TLS handshake
      ▼
  ┌──────────────── Load Balancer (multi-AZ) ────────────────┐
  │  3. LISTENER (HTTPS:443) terminates TLS                   │
  │  4. RULES match the request (host / path) → target group │
  │  5. picks a HEALTHY target from that group                │
  └───────────────────────────┬──────────────────────────────┘
                              │ 6. forwards request
                              ▼
                         ┌────────┐
                         │ Target │  7. processes, returns response
                         └────────┘
                              │
                              ▼
   8. LB relays the response back to the client over the same connection
```

> **Key insight**: The client only ever knows the load balancer's address. Targets can scale, get replaced, fail health checks, and move AZs — entirely invisible to the client. That decoupling is the whole point.

---

## 9. Key Exam Points

- **Load balancers enable horizontal scaling and remove the single point of failure.** They are the front door to a fleet of stateless servers.
- **Layer 4 (NLB) = TCP/UDP, fast, dumb. Layer 7 (ALB) = HTTP/HTTPS, smart routing on path/host/headers.** This is the most-tested distinction.
- **Listener → rules → target group → targets** is the universal object chain. Health checks live on the **target group**.
- **Unhealthy targets are automatically removed** from rotation; this is how the LB delivers availability.
- **Deregistration delay / connection draining** lets in-flight requests finish before a target is removed (default 300 s).
- **Prefer stateless servers**; use sticky sessions only when state can't be externalized. ALB stickiness = cookie; NLB = source IP.
- **Health-check a real `/health` endpoint**, not just port liveness.

---

## 10. Common Mistakes

- **❌ Storing session state on the instance's local disk/memory.** Loses sessions on scale-in or rerouting. Externalize to ElastiCache/DynamoDB.
- **❌ Confusing scale up with scale out.** "Add more servers behind an LB" is scale *out* (horizontal); "use a bigger instance" is scale *up* (vertical).
- **❌ Thinking the LB protects a single instance.** One target behind an LB is still a single point of failure — you need targets in **multiple AZs**.
- **❌ Health-checking `/` and assuming the app is healthy.** A 200 on the home page can hide a dead database connection.
- **❌ Leaving deregistration delay too high for fast-scaling apps**, making scale-in sluggish — or too low for long uploads, cutting users off.

---

## Limits & Defaults (concept-level)

| Thing | Default / limit |
|-------|-----------------|
| Deregistration delay (ALB/NLB) | 300 s default, 0–3600 s |
| Health check interval | Configurable (commonly 30 s) |
| Healthy / unhealthy threshold | Small integer counts (e.g., 2–3) |
| Cross-AZ requirement for HA | Targets in **≥2 AZs** |

(Per-LB connection, listener, and target-group quotas are product-specific — covered in the next file.)

---

**Next**: [02_elb_alb_nlb_gwlb.md — The ELB Family: ALB, NLB & GWLB](02_elb_alb_nlb_gwlb.md)
