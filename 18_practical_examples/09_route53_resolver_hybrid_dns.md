# Practical: Hybrid DNS with Route 53 Resolver Endpoints

> **Who this is for**: Engineers connecting on-prem (data center) and AWS over VPN or Direct Connect
> who need DNS to work **both directions**: on-prem servers resolving AWS private names
> (`db.internal`), and AWS instances resolving on-prem names (`erp.corp.local`). We cover **inbound**
> and **outbound** resolver endpoints, resolver rules, and the path over the tunnel.
>
> Prerequisites: **[08_route53_private_hosted_zone.md](08_route53_private_hosted_zone.md)** (PHZs)
> and **[Route 53](../08_dns_edge/02_route_53.md)**.

---

## 1. The Problem

The Amazon-provided resolver (`.2` / `169.254.169.253`) lives **inside** a VPC and answers only from
within it. It can't be reached from on-prem, and it doesn't know about your on-prem zones. So:

- On-prem DNS server queries `db.internal` → it has no idea where AWS's resolver is → **fails**.
- An EC2 instance queries `erp.corp.local` → Amazon resolver isn't authoritative for `corp.local`
  and has no forwarding rule → **fails**.

**Route 53 Resolver endpoints** bridge this. They are ENIs in your VPC that expose / forward DNS.
They are DNS resolver endpoints, not VPC service endpoints like gateway/interface endpoints for
S3, DynamoDB, or AWS APIs.

| Endpoint type | Direction | Solves | Mental model |
|---------------|-----------|--------|--------------|
| **Inbound**  | on-prem → AWS | on-prem resolves **AWS** private names | "A door *in* to the Amazon resolver" |
| **Outbound** | AWS → on-prem | AWS resolves **on-prem** names | "A door *out*, plus rules saying which domains to forward" |

> **Key insight**: Inbound = others query *into* the Amazon resolver. Outbound = the Amazon resolver
> forwards selected domains *out* to your DNS. Most hybrid setups need **both**.

---

## 2. Inbound Endpoint — On-Prem Resolves AWS Private Zones

An **inbound endpoint** creates resolver ENIs (with private IPs, in 2+ subnets/AZs) that your on-prem
DNS server can forward to. Once it forwards to those IPs, on-prem can resolve any PHZ associated with
that VPC.

```
 ON-PREM                          VPN / Direct Connect           AWS VPC (vpc-app)
 ┌───────────────┐                       │                ┌──────────────────────────┐
 │ corp DNS       │── forward zone ───────┼───────────────►│ Inbound Resolver ENIs    │
 │ "internal" →   │   internal → 10.0.0.10│                │  10.0.0.10 / 10.0.1.10   │
 │  10.0.0.10,    │◄─────── 10.0.5.20 ────┼────────────────│  → Amazon resolver (.2)  │
 │  10.0.1.10     │   (db.internal answer)│                │  → PHZ "internal"        │
 └───────────────┘                       │                └──────────────────────────┘
```

```bash
aws route53resolver create-resolver-endpoint \
  --name inbound-from-onprem \
  --direction INBOUND \
  --security-group-ids sg-resolver \
  --ip-addresses SubnetId=subnet-a,Ip=10.0.0.10 SubnetId=subnet-b,Ip=10.0.1.10
```

Then on the **on-prem DNS server** (BIND example), conditionally forward the AWS zone to those ENI IPs:

```
# named.conf — forward only the internal zone to the AWS inbound endpoint
zone "internal" {
    type forward;
    forward only;
    forwarders { 10.0.0.10; 10.0.1.10; };
};
```

⚠️ The resolver ENI **security group must allow inbound UDP/TCP 53** from your on-prem CIDR, and the
route over VPN/DX must reach those subnets.
If the subnet NACL is restrictive, remember it is stateless: allow DNS and return traffic in the
opposite direction. Security groups are stateful; NACLs are not.

---

## 3. Outbound Endpoint + Resolver Rules — AWS Resolves On-Prem

An **outbound endpoint** lets the Amazon resolver forward queries *out*. You attach **resolver rules**
that say "for domain X, forward to these on-prem DNS IPs," then **associate the rule with VPCs**.

```
 AWS VPC (vpc-app)                          VPN / Direct Connect      ON-PREM
 ┌───────────────────────────┐                     │            ┌───────────────┐
 │ EC2 → "erp.corp.local?"    │                     │            │ corp DNS      │
 │   → Amazon resolver (.2)   │                     │            │ 192.168.1.53  │
 │   rule: corp.local →        ── forward ──────────┼───────────►│ authoritative │
 │     192.168.1.53            │                     │            │ for corp.local│
 │   via Outbound ENIs         │◄──── 192.168.1.40 ─┼────────────│               │
 │   10.0.0.20 / 10.0.1.20    │   (erp answer)      │            └───────────────┘
 └───────────────────────────┘                     │
```

```bash
# 1. Outbound endpoint (ENIs that forward out)
aws route53resolver create-resolver-endpoint \
  --name outbound-to-onprem \
  --direction OUTBOUND \
  --security-group-ids sg-resolver \
  --ip-addresses SubnetId=subnet-a SubnetId=subnet-b

# 2. Forwarding rule: corp.local -> on-prem DNS servers
aws route53resolver create-resolver-rule \
  --name fwd-corp-local \
  --rule-type FORWARD \
  --domain-name corp.local \
  --resolver-endpoint-id rslvr-out-abc123 \
  --target-ips Ip=192.168.1.53,Port=53 Ip=192.168.2.53,Port=53

# 3. Associate the rule with the VPC(s) whose instances should use it
aws route53resolver associate-resolver-rule \
  --resolver-rule-id rslvr-rr-xyz789 \
  --vpc-id vpc-app
```

💡 Resolver rules can be **shared across accounts via AWS RAM**, so one networking account owns the
rules and many app-account VPCs associate them — a common multi-account hybrid pattern.

---

## 4. Resilient Multi-Account Design

A full hybrid DNS setup uses **inbound + outbound** in a dedicated
shared-services VPC in each Region:

```
        ON-PREM  corp.local                         AWS  internal (PHZ)
   ┌────────────────────┐                       ┌────────────────────────┐
   │ corp DNS           │── forward "internal" ─►│ INBOUND endpoint       │
   │ 192.168.1.53       │◄── PHZ answers ────────│  10.0.0.10/10.0.1.10   │
   │                    │                        │                        │
   │ authoritative for  │◄── forward "corp.local"│ OUTBOUND endpoint      │
   │ corp.local         │── on-prem answers ────►│  + resolver rule       │
   └────────────────────┘   over VPN / DX        └────────────────────────┘
```

Build it as a platform, not as an endpoint in every application VPC:

1. The network/DNS account owns the endpoints, their subnets, security groups,
   forwarding rules, and the connection to on-premises networks.
2. Put at least two endpoint IPs in different AZs. Assign the inbound IPs
   explicitly so an endpoint restored after accidental deletion can reuse the
   addresses already configured on-premises.
3. Share outbound rules through **AWS RAM**. A workload account accepts the
   share and associates the rule with its VPC; it does not create its own
   outbound endpoint. Sharing a rule also makes its outbound endpoint serve the
   consumer VPC's queries.
4. Associate centrally served PHZs with the VPC containing the inbound endpoint
   (and with workload VPCs that query them directly). An inbound endpoint can
   only expose names the resolver in that endpoint VPC can resolve.
5. Repeat the design per Region. Resolver endpoints and rules are Regional;
   another Region is a separately deployed recovery path, not an automatic
   failover target.

On-premises DNS must list **every** inbound endpoint IP and have at least two
recursive resolvers, ideally split across sites. The AWS outbound rule should
target at least two on-premises resolvers. Make the routes redundant too: two
DNS IPs are not resilient if both depend on one VPN tunnel, Direct Connect
virtual interface, router, or on-premises site.

### Prevent forwarding loops

Publish a namespace ownership map before creating rules:

| Suffix | Authority | On-prem action | AWS action |
|--------|-----------|----------------|------------|
| `aws.example.com` | Route 53 PHZ | Forward to AWS inbound IPs | Resolve locally |
| `corp.example.com` | On-prem DNS | Resolve locally | Forward through outbound rule |
| `example.com` public names | Public DNS | Recurse normally | Recurse normally |

Do not configure both sides to forward the same suffix back to each other. A
root (`.`) forwarder is especially dangerous. Use the narrowest suffix, record
the exception owner, and test a nonexistent name: a loop often appears as high
latency/SERVFAIL rather than an obvious configuration error.

### Central policy and evidence

- Enable Resolver query logging for all participating VPCs and centralize it in
  a security/log-archive account. Retain the VPC, source resource, query,
  response, and DNS Firewall action needed for investigations.
- Associate shared **Resolver DNS Firewall** rule groups for known-malicious
  domains and organization allow/block policy. Decide explicitly whether each
  VPC should fail open or fail closed if Firewall cannot return a filtering
  decision, and monitor blocks before enforcing a new rule broadly.
- Use CloudTrail for endpoint, rule, RAM-share, logging, and firewall control
  plane changes. Separate platform administrators from application teams that
  only associate an approved shared rule.

### Capacity and failover tests

Endpoint capacity scales per endpoint IP. Graph `InboundQueryVolume` and
`OutboundQueryAggregateVolume` **per IP**, enable the detailed capacity/target
name-server metrics where useful, alarm before the current quota is reached,
and add endpoint IPs when load or maintenance headroom requires it. Also measure
TCP DNS and large responses; a UDP-only test misses truncation and fallback.

Run these tests from both directions at least quarterly:

- Stop advertising one VPN/DX route or block one endpoint IP; clients should
  continue through the other AZ/link without a long outage.
- Make one on-premises target resolver unavailable; verify the outbound rule
  still resolves through the other target and measure timeout impact.
- Query known AWS, on-premises, public, blocked, and nonexistent names. Capture
  latency, SERVFAIL/NXDOMAIN rate, endpoint-IP load balance, and firewall logs.
- Restore an endpoint from infrastructure as code using the documented inbound
  IPs, then verify conditional forwarders. Cached answers can hide a broken
  path, so include uncached names or wait for TTL expiry.

Assign one team to own endpoint/rule health and another clear escalation owner
for on-premises DNS and connectivity. A dashboard without an operator and a
runbook is not a recovery plan.

---

## 5. Inbound vs Outbound — When to Use Which

| You want… | Use | Plus |
|-----------|-----|------|
| On-prem to resolve `*.internal` (AWS PHZ) | **Inbound** endpoint | on-prem conditional forwarder → ENI IPs |
| EC2 to resolve `*.corp.local` (on-prem) | **Outbound** endpoint | FORWARD resolver rule + VPC association |
| Both (typical enterprise hybrid) | **Both** | rules can be RAM-shared cross-account |
| EC2 to resolve public internet names | Nothing extra | Amazon resolver does this natively |

⚠️ Don't create a rule that forwards a domain the Amazon resolver should answer (e.g. forwarding `.`
or a PHZ-owned domain out to on-prem) — you'll black-hole resolution. Forward **only** the specific
on-prem domains.

---

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| On-prem can't resolve `db.internal` | Inbound SG blocks port 53, or no forwarder | Allow UDP/TCP 53 from on-prem CIDR; point on-prem forwarder at ENI IPs |
| On-prem resolves but AWS box can't reach the IP | Routing/SG to the target, not DNS | Fix VPN/DX route + target SG (DNS ≠ reachability) |
| EC2 can't resolve `erp.corp.local` | Rule not associated with the VPC | `associate-resolver-rule` to that VPC |
| Forwarding rule has no effect | Rule domain too broad / overlaps PHZ | Narrow `--domain-name` to the exact on-prem zone |
| Intermittent failures | Only one ENI/AZ reachable | Ensure both endpoint IPs (2+ AZs) are routable |
| One endpoint IP receives almost all queries | Forwarder ordering/load distribution | Configure all IPs on every on-prem resolver and inspect `InboundQueryVolume` per IP |
| SERVFAIL latency spikes for one suffix | Forwarding loop or failed target resolver | Trace namespace ownership, remove the loop, and test each target IP over UDP/TCP 53 |
| Outbound queries fail only from shared VPCs | RAM share accepted but rule not associated | Associate the shared rule with each consumer VPC and verify routes from endpoint subnets |
| All AWS DNS breaks after adding a rule | Forwarded `.` (root) outbound | Delete the catch-all rule; forward specific zones only |
| Cross-account rule missing | Rule not RAM-shared/associated | Share via RAM, then associate in the app account |

---

## Key Exam Points

- ✅ **Inbound endpoint** = on-prem → AWS: lets on-prem DNS **resolve AWS private hosted zones** by
  forwarding to the resolver ENIs.
- ✅ **Outbound endpoint + FORWARD resolver rule** = AWS → on-prem: lets EC2 **resolve on-prem domains**
  by forwarding specified domains to on-prem DNS.
- ✅ Endpoints are **ENIs in 2+ subnets/AZs**; their **security group must allow port 53** (UDP & TCP)
  to/from the right side.
- ✅ Resolver rules are **associated with VPCs** and can be **shared across accounts via AWS RAM**.
- ✅ Both directions traverse the existing **VPN or Direct Connect** — Route 53 Resolver does not create
  connectivity, only DNS forwarding; routing/SGs still apply.
- ⚠️ Never forward the root (`.`) or a PHZ domain outbound — it black-holes resolution.
- ✅ Production hybrid DNS also needs redundant on-prem resolvers and links,
  centralized query logs/firewall policy, per-IP capacity alarms, failure
  drills, and named operational owners.

---

**Next**: [10_s3_static_site_vs_cloudfront.md — Static Site: S3 Website vs CloudFront](10_s3_static_site_vs_cloudfront.md)
