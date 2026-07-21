# Security Groups vs Network ACLs — The Two Firewall Layers

> **Who this is for**: Engineers who understand [VPCs and subnets](02_vpc_subnets_route_tables.md)
> and the [stateful vs stateless firewall distinction](01_networking_primer.md#8️⃣-firewalls--stateful-vs-stateless)
> and [ephemeral ports](01_networking_primer.md#6️⃣-ports-and-protocols) from the primer.
> Those two primer concepts are the *entire* basis of this file — if they're fuzzy, reread them now.

---

## 1. Two Layers, Two Behaviors

AWS gives you **two independent firewall layers** around your resources. A packet must pass **both** to reach an instance:

```
   Internet / other subnet
            │
            ▼
   ┌───────────────────┐
   │  NETWORK ACL      │   ← subnet boundary, STATELESS, allow + deny
   └─────────┬─────────┘
             ▼
   ┌───────────────────┐
   │  SECURITY GROUP   │   ← instance/ENI boundary, STATEFUL, allow only
   └─────────┬─────────┘
             ▼
        EC2 instance (ENI)
```

- A **Network ACL (NACL)** sits at the **subnet** boundary — it filters traffic entering or leaving the whole subnet.
- A **Security Group (SG)** sits at the **instance** boundary (technically the ENI — elastic network interface) — it filters traffic to/from that specific resource.

They behave fundamentally differently, and that difference is the most heavily tested networking topic on the exam.

---

## 2. Security Groups — Stateful, Instance-Level, Allow-Only

A Security Group is a **stateful** firewall attached to an instance's network interface.

- **Stateful**: if you allow an inbound request, the **response is automatically allowed** out — and vice versa. You never write return-traffic rules. (This is the stateful behavior from the primer.)
- **Allow rules only**: there is **no deny rule** in a security group. Anything not explicitly allowed is implicitly denied. You cannot block a specific IP with an SG — you can only *not allow* it.
- **Default behavior**: all inbound **denied**, all outbound **allowed**.
- Rules can reference **another security group** as the source (e.g., "allow from the web-tier SG"), which is powerful — the rule follows the instances, not their IPs.
- An instance can have **multiple SGs**; rules are **evaluated as the union** (most permissive wins among allows).

```
Web-tier SG                          DB-tier SG
─ Inbound: allow 443 from 0.0.0.0/0  ─ Inbound: allow 3306 from web-tier-SG
─ Outbound: allow all (default)      ─ Outbound: allow all (default)

The DB rule references the SG, not an IP — every web instance can reach the DB
on 3306 automatically, no matter how many there are or what IPs they get.
```

> **Key insight**: Because SGs are stateful, you almost never think about ephemeral return ports. Allow inbound 443, and the reply just works.

---

## 3. What Actually Gets a Security Group

Security groups are not just for EC2. They attach to ENIs created by many VPC
resources.

| Has security groups | No security group |
|---------------------|-------------------|
| EC2 ENIs, ALB, NLB when attached at creation, RDS/Aurora, ElastiCache, EFS mount targets, ECS/Fargate task ENIs, VPC Lambda Hyperplane ENIs, interface VPC endpoints, Route 53 Resolver endpoints | Subnets, route tables, IGW, VPC peering, Transit Gateway, gateway VPC endpoints, NAT Gateway, Gateway Load Balancer |

The "NAT Gateway" row is a favorite trap: a zonal NAT Gateway creates a managed
ENI and has a private IP in a subnet, while regional mode manages networking at
VPC scope. You **cannot** attach a security group to either. A NAT Instance is an
EC2 instance, so it **does** use security groups.

For the full resource-by-resource map, see
[ENIs, Security Groups & Service Networking](07_enis_security_groups_and_service_networking.md).

---

## 4. Network ACLs — Stateless, Subnet-Level, Allow + Deny, Ordered

A Network ACL is a **stateless** firewall at the subnet boundary.

- **Stateless**: each packet is judged independently. Allowing inbound traffic does **not** allow the response back out — you must add a **separate outbound rule** for the return traffic, on the **ephemeral port range**. (This is the stateless behavior from the primer, and the source of most NACL bugs.)
- **Allow and deny rules**: unlike SGs, NACLs can **explicitly deny** — useful for blocking a known-bad IP across an entire subnet.
- **Numbered, ordered rules**: rules are evaluated **lowest number first**, and the **first match wins** (allow or deny), then evaluation stops. There's a final implicit `*` rule that **denies** anything unmatched.
- **Default NACL**: allows all inbound and all outbound. A **custom NACL** denies everything until you add rules.

```
Inbound NACL rules (evaluated top to bottom, first match wins)
┌──────┬──────────┬───────────────┬───────┬────────┐
│ Rule │ Protocol │ Port range    │ Source│ Allow? │
├──────┼──────────┼───────────────┼───────┼────────┤
│ 100  │ TCP      │ 443           │ 0/0   │ ALLOW  │
│ 200  │ TCP      │ 1024–65535    │ 0/0   │ ALLOW  │ ← return traffic for outbound conns
│  *   │ all      │ all           │ 0/0   │ DENY   │ ← implicit catch-all
└──────┴──────────┴───────────────┴───────┴────────┘
```

> **Rule**: NACL rule order matters. A `DENY` at rule 100 beats an `ALLOW` at rule 200 for the same traffic, because the lower number is evaluated first and stops the search.

---

## 5. The Big Comparison Table

| Feature | Security Group | Network ACL |
|---------|----------------|-------------|
| Operates at | **Instance / ENI** level | **Subnet** level |
| State | **Stateful** (return traffic auto-allowed) | **Stateless** (must allow return traffic explicitly) |
| Rule types | **Allow only** | **Allow and Deny** |
| Rule evaluation | All rules evaluated; union of allows | **Numbered, first match wins, then stops** |
| Default (custom) | Deny all inbound, allow all outbound | Deny all (custom); allow all (default NACL) |
| Can reference | Other security groups, IPs | IP CIDR ranges only |
| Applies to | Whatever ENIs you attach it to | Automatically all instances in the subnet |
| Ephemeral ports | Handled automatically (stateful) | **You must allow them manually** |
| Typical use | Primary firewall — fine-grained per resource | Coarse subnet guardrail; explicit IP blocking |

> **Mental model**: Security Group = a bouncer who **remembers** who's already inside and lets them leave freely. NACL = a bouncer with **amnesia** who checks every person in *both* directions against a numbered list.

---

## 6. Worked Example — Why NACLs Need Return-Traffic Rules

A web server in a subnet must serve HTTPS (port 443) to the internet. Walk through both firewalls.

**With a Security Group only (stateful) — this works:**

```
SG inbound:  allow TCP 443 from 0.0.0.0/0
SG outbound: allow all (default)

Client → server:443         ✅ allowed by inbound rule
server → client:ephemeral   ✅ auto-allowed (stateful remembers the connection)
DONE. No return rule needed.
```

**With a custom NACL (stateless) — the naive attempt FAILS:**

```
NACL inbound:  100  allow TCP 443  from 0.0.0.0/0
NACL outbound: (nothing)

Client → server:443                 ✅ allowed by inbound rule 100
server → client:ephemeral (e.g. 52000)  ❌ DROPPED — no outbound rule!
RESULT: connection hangs. Client gets no response.
```

The reply leaves on a **random ephemeral port** (1024–65535, from the primer), *not* port 443. The stateless NACL doesn't remember the inbound connection, so it evaluates the outbound reply on its own and finds no matching allow.

**Correct NACL — you must add the ephemeral return rule:**

```
NACL inbound:   100  allow TCP 443         from 0.0.0.0/0
NACL outbound:  100  allow TCP 1024–65535  to   0.0.0.0/0   ← the fix
```

Now the reply on the ephemeral port matches the outbound rule. And don't forget: if instances *initiate* outbound connections (e.g., to download updates), you'll also need an **inbound** ephemeral rule for *those* replies.

> **Key insight**: Every NACL allow rule usually needs a matching ephemeral-port rule in the *opposite* direction. Stateless means you account for both legs of every conversation.

---

## 7. Common Exam Traps

- ⚠️ **"Can a Security Group block a specific malicious IP?"** → **No.** SGs are allow-only. Use a **NACL deny** rule (subnet-level) to block an IP.
- ⚠️ **"Traffic reaches the subnet but not the instance"** → the NACL allowed it but a Security Group didn't (or vice versa). A packet must pass **both**.
- ⚠️ **"HTTPS works with SG but breaks after adding a custom NACL"** → missing ephemeral-port return rule on the NACL.
- ⚠️ **SGs reference SGs; NACLs reference only CIDRs.** If a question says "allow the app tier regardless of its IPs," that's an SG-referencing-SG answer.
- ⚠️ **NACL rule numbering**: a lower-numbered DENY overrides a higher-numbered ALLOW. Evaluation stops at the first match.
- ⚠️ **NACLs are stateless even for outbound you initiate** — return traffic for *your own* outbound connections needs an inbound ephemeral allow.
- ⚠️ **Not every managed ENI has an editable SG** — NAT Gateway and Gateway Load Balancer are the big contrasts; interface endpoints do have SGs.

---

## 8. Organization-Scale Segmentation and Enforcement

In one account, a careful security-group chain is manageable. Across hundreds of
accounts, manual review drifts: a team adds `0.0.0.0/0` to an admin port, a new
resource launches without the shared-services rules, or an unused SG remains
attached. Treat segmentation as an organization policy with workload-level and
network-level controls:

```
Organization / OU boundary
  -> account and VPC boundary
     -> TGW route-table or shared-subnet boundary
        -> workload SG references
           -> optional NACL / Network Firewall guardrail
```

Use accounts and VPCs for strong trust boundaries, Transit Gateway route tables
for which networks may route to one another, and SG references for the exact
application flows inside those boundaries. NACLs remain a coarse subnet control;
they are not a scalable replacement for workload policy.

### AWS Firewall Manager security-group policies

**AWS Firewall Manager** applies security-group policy across in-scope AWS
Organizations accounts and Regions. Scope policies by OU/account, resource type,
and tags instead of copying rules into each account.

Set up an Organizations member account as a Firewall Manager administrator and
enable continuous AWS Config recording in every in-scope account and Region;
Firewall Manager uses that configuration inventory to evaluate compliance.

| Policy type | What it solves |
|-------------|----------------|
| **Common security group** | Distributes centrally defined SGs and associates them with in-scope resources, so every workload receives required baseline access. |
| **Content audit** | Finds rules that violate an allow/deny policy, such as SSH or database ports open to the internet; optional remediation removes noncompliant rules. |
| **Usage audit** | Finds unused or redundant security groups that should be cleaned up. |

Start a content or usage audit with automatic remediation disabled, inspect the
findings, and then enable remediation once exclusions are correct. A broad
remediation policy can delete a rule that an application depends on just as
quickly as it can remove a dangerous one. Keep application-owned SGs narrow even
when a common baseline SG is attached: SG rules are a union, so one permissive
group can undo the intended boundary.

### Validate the paths, not only the rules

A policy can make every individual rule look reasonable while their combined
routes, peering, TGW attachments, SGs, and NACLs still create an unintended path.
**Network Access Analyzer** performs static configuration analysis against a
**Network Access Scope** and reports paths that match the scope without sending
packets.

Example scope: find any path from an internet gateway, peering connection, or
Transit Gateway attachment to database ENIs, excluding the approved path through
the application load balancer. Run it after topology and policy changes and
investigate each finding. Use [Reachability Analyzer](08_dhcp_prefix_lists_sharing_analyzer.md#5-reachability-analyzer--network-access-analyzer--proving-the-path)
for one expected source-to-destination path; use Network Access Analyzer to find
all paths that should not exist.

> **Key insight**: Firewall Manager enforces rule hygiene across the organization;
> Network Access Analyzer tests the effective connectivity created by all the
> network controls together. You need both policy and path validation.

---

## 9. Key Exam Points

- **SG = stateful, instance-level, allow-only.** Return traffic is automatic.
- **NACL = stateless, subnet-level, allow + deny, numbered/ordered, first-match-wins.** Return traffic is **manual** on ephemeral ports.
- A packet must be permitted by **both** the NACL **and** the SG.
- Only a **NACL** can explicitly **block** an IP; an SG can't deny.
- Default SG: deny inbound / allow outbound. Default NACL: allow all; **custom** NACL: deny all.
- SG rules can reference other SGs; NACL rules use CIDRs only.
- Ephemeral return ports are the #1 NACL trap.
- SGs are common on ENI-backed resources, but gateway-style resources usually do not have SGs.
- **Firewall Manager SG policies** distribute baseline groups and audit or
  remediate noncompliant/unused rules across AWS Organizations.
- **Network Access Analyzer** evaluates Network Access Scopes to find unintended
  paths created by the combined routing and security configuration.

---

## Common Real-World Misconfigurations

- ❌ Switching to a custom NACL and forgetting outbound ephemeral allow rules — half-open, hanging connections.
- ❌ Trying to ban an attacker IP with a security group (impossible) instead of a NACL deny.
- ❌ Over-broad NACL rules that defeat the purpose; the SG should do fine-grained filtering.
- ❌ Assuming an SG change protects the whole subnet — it only affects the ENIs it's attached to.
- ❌ Misordered NACL rules where a broad early ALLOW shadows a later, more specific DENY.
- ❌ Copying a baseline SG into every account by hand and assuming it stays
  consistent — enforce and audit it centrally with Firewall Manager.
- ❌ Reviewing SG rules in isolation without checking the effective routes and
  paths through Network Access Analyzer.

---

**Next**: [05_vpc_peering_and_endpoints.md — Connecting VPCs and Reaching Services Privately](05_vpc_peering_and_endpoints.md)
