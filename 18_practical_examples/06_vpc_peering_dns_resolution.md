# Practical: Private DNS Resolution Across VPC Peering

> **Who this is for**: Engineers who have a peering connection up and "ping by IP works,
> but the hostname resolves to the wrong (public) IP — or doesn't resolve at all." This is
> the classic exam trap and a real-world footgun. We trace **exactly** when a private DNS
> name in a peer VPC resolves to a **private** IP versus a **public** IP versus **fails**,
> and the precise settings, routes, and SGs required.
>
> Prerequisites: **[Route 53](../08_dns_edge/02_route_53.md)** (hosted zones, the Amazon
> DNS resolver) and **[VPC Peering & Endpoints](../03_networking/05_vpc_peering_and_endpoints.md)**
> (peering basics, route tables, non-transitivity).

---

## 1. The Scenario

Two VPCs in the same account/Region, peered together. We want an instance in VPC A to reach
an instance in VPC B **by its DNS hostname** and have that hostname resolve to B's **private**
IP, so traffic stays on the AWS backbone.

```
   VPC A  10.0.0.0/16                          VPC B  172.16.0.0/16
   ┌────────────────────────┐                  ┌────────────────────────┐
   │  EC2-A  10.0.1.5        │                  │  EC2-B  172.16.2.9      │
   │                        ─┼──── pcx-abc123 ──┼─                        │
   │  queries:               │   (peering conn) │  hostname:              │
   │  ip-172-16-2-9          │                  │  ip-172-16-2-9          │
   │  .ec2.internal          │                  │  .ec2.internal          │
   └────────────────────────┘                  └────────────────────────┘
              ▲                                            │
              │   What does this resolve to?               │
              │   172.16.2.9 (private)  ✅  ← what we want  │
              │   <public IP>           ⚠️  ← the default   │
              │   NXDOMAIN / SERVFAIL   ❌                  │
              └────────────────────────────────────────────┘
```

There are **two independent DNS questions** people conflate:

1. **The default EC2 hostname** (`ip-172-16-2-9.ec2.internal` / `*.compute.internal`) — managed
   by the **Amazon-provided DNS resolver** (the `.2` address, `VPC_CIDR_base + 2`, also reachable at
   `169.254.169.253`). This section is about *this*.
2. **A custom name from a Route 53 Private Hosted Zone** (e.g. `db.internal`). That's a *different*
   mechanism — covered in [§6](#6-route-53-private-hosted-zone-across-peering) and in
   [08_route53_private_hosted_zone.md](08_route53_private_hosted_zone.md).

> **Key insight**: A peering connection moves *packets*. It does **not** automatically make the
> two VPCs' DNS resolvers answer each other's private hostnames with private IPs. DNS resolution
> across peering is a **separate opt-in** on top of routing.

---

## 2. The Default Behavior (and Why It Bites)

Out of the box, when EC2-A queries the **public DNS name** of an instance in the peer VPC, the
Amazon resolver returns the **public IP** (if the target has one) — or the query effectively gives
you nothing usable for private routing. It does **not** return the peer's private IP.

```
EC2-A  ──"resolve ec2-203-0-113-9.compute-1.amazonaws.com"──►  Amazon DNS (.2 in VPC A)
                                                  returns ◄──  203.0.113.9   (PUBLIC IP)  ⚠️
```

Why this is bad across peering:

- The public IP routes **out to the internet** (via IGW/NAT), not across `pcx-abc123`.
- You pay for NAT/egress and traffic leaves the backbone — defeating the point of peering.
- If the target instance has **no public IP**, you get nothing routable at all.

⚠️ This is the trap: peering "works" (you can `ping 172.16.2.9` by IP), `traceroute` by IP is clean,
but DNS hands back a public address. People blame routing or SGs when the real fix is a **DNS toggle**.

---

## 3. The Fix — Enable DNS Resolution on the Peering Connection (BOTH Sides)

To make the Amazon resolver return **private** IPs for the peer VPC's hostnames, you must turn on
the peering connection's DNS resolution option **on each side that needs it**. This depends on the
**VPC-level DNS attributes** being on as well.

### 3a. VPC attributes (prerequisite, on both VPCs)

Both `enableDnsSupport` and `enableDnsHostnames` must be **true** on each VPC.

| Attribute | What it does | Default (default VPC) | Default (custom VPC) |
|-----------|--------------|-----------------------|----------------------|
| `enableDnsSupport` | Amazon DNS resolver (`.2`) answers queries at all | `true` | `true` |
| `enableDnsHostnames`| Instances get a resolvable public/private DNS hostname | `true` | **`false`** |

```bash
# Check, then enable on BOTH VPCs
aws ec2 describe-vpc-attribute --vpc-id vpc-aaa111 --attribute enableDnsHostnames
aws ec2 describe-vpc-attribute --vpc-id vpc-aaa111 --attribute enableDnsSupport

aws ec2 modify-vpc-attribute --vpc-id vpc-aaa111 --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id vpc-aaa111 --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id vpc-bbb222 --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id vpc-bbb222 --enable-dns-hostnames
```

⚠️ `enableDnsHostnames` defaults to **false** in a custom (non-default) VPC. This alone is a frequent
cause of "no hostname / won't resolve."

### 3b. The peering connection DNS option (the actual cross-peer toggle)

Each side of the peering connection has an accepter/requester DNS option:
**`AllowDnsResolutionFromRemoteVpc`**. Enable it on the side that needs to *resolve the other side's*
private hostnames. For bidirectional resolution, enable it on **both**.

```bash
# Let the REQUESTER side (VPC A) resolve VPC B's private DNS to private IPs
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-abc123 \
  --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true

# And the ACCEPTER side (VPC B) so B can resolve A's, too
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-abc123 \
  --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true
```

Console path: **VPC → Peering connections → select `pcx-abc123` → Actions → Edit DNS settings →**
check **"Accepter/Requester DNS resolution"**.

✅ After this, EC2-A querying EC2-B's hostname gets back `172.16.2.9` (private) instead of the public IP:

```
EC2-A  ──"resolve ip-172-16-2-9.ec2.internal"──►  Amazon DNS (.2 in VPC A)
                                       returns ◄──  172.16.2.9   (PRIVATE IP)  ✅
```

> **Rule**: VPC attributes (`enableDnsSupport` + `enableDnsHostnames`) are *necessary but not
> sufficient*. The peering-connection `AllowDnsResolutionFromRemoteVpc` option is the toggle that
> actually flips public-IP answers to private-IP answers across the peer.

---

## 4. Routing and Security Groups (still required — DNS ≠ reachability)

Getting the private IP back from DNS is useless if packets can't flow. DNS resolution and network
reachability are **orthogonal** — you need both.

### 4a. Route tables — routes to the peer CIDR, BOTH directions

Peering does **not** auto-create routes. Each VPC's route table (the one associated with the relevant
subnet) needs a route to the *other* VPC's CIDR via the peering connection.

```
VPC A route table (rtb-a)              VPC B route table (rtb-b)
┌──────────────┬───────────────┐       ┌──────────────┬───────────────┐
│ 10.0.0.0/16  │ local         │       │ 172.16.0.0/16│ local         │
│ 172.16.0.0/16│ pcx-abc123    │  ◄──► │ 10.0.0.0/16  │ pcx-abc123    │
└──────────────┴───────────────┘       └──────────────┴───────────────┘
```

```bash
aws ec2 create-route --route-table-id rtb-a \
  --destination-cidr-block 172.16.0.0/16 --vpc-peering-connection-id pcx-abc123
aws ec2 create-route --route-table-id rtb-b \
  --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id pcx-abc123
```

⚠️ Missing the **return route** on the far side is the #1 "it half-works" cause: SYN reaches B, B's
reply has no route back to A's CIDR, connection hangs.

### 4b. Security groups — allow the traffic (you can reference the peer's SG)

The target's SG must allow inbound from the source. Within the **same Region**, you can reference the
**peer VPC's security group ID** directly instead of hard-coding CIDRs.

```bash
# Allow EC2-B (sg-b) to accept 443 from EC2-A's SG across the peer (same-Region only)
aws ec2 authorize-security-group-ingress \
  --group-id sg-b \
  --ip-permissions 'IpProtocol=tcp,FromPort=443,ToPort=443,UserIdGroupPairs=[{GroupId=sg-a,VpcId=vpc-aaa111}]'
```

| Connectivity | SG-by-CIDR | SG-by-peer-SG-reference |
|--------------|-----------|--------------------------|
| Same Region peering | ✅ | ✅ |
| **Inter-Region** peering | ✅ | ❌ (must use CIDR) |

💡 NACLs (if you've tightened them from the default allow-all) must permit the traffic **and its
ephemeral return ports** in both subnets — NACLs are stateless.

---

## 5. Resolves vs Does Not Resolve — the Master Table

Assume the peering connection `pcx-abc123` is **active** and you're querying a peer-VPC resource.

| # | enableDnsSupport (both) | enableDnsHostnames (both) | Peering `AllowDnsResolutionFromRemoteVpc` | Query type | Result |
|---|---|---|---|---|---|
| 1 | ✅ | ✅ | ❌ off | Peer EC2 **public** hostname | Resolves to **PUBLIC IP** ⚠️ (routes via internet, not peer) |
| 2 | ✅ | ✅ | ✅ on (that side) | Peer EC2 **public/private** hostname | Resolves to **PRIVATE IP** ✅ |
| 3 | ❌ off | — | any | any internal name | **No resolution** (resolver disabled) ❌ |
| 4 | ✅ | ❌ off | any | Peer EC2 hostname | Instance may have **no DNS hostname** to resolve ❌/⚠️ |
| 5 | ✅ | ✅ | ✅ on **one** side only | Reverse direction's name | **Asymmetric** — only the enabled direction gets private IPs ⚠️ |
| 6 | ✅ | ✅ | ✅ on | **Route 53 PHZ** name (`db.internal`) but PHZ **not associated** with querying VPC | **NXDOMAIN** ❌ (see §6) |
| 7 | ✅ | ✅ | n/a | **Route 53 PHZ** name, PHZ **associated** with querying VPC | Resolves to PHZ record value ✅ |

> **Key insight**: The peering DNS option only governs the **Amazon default hostnames**
> (`*.compute.internal` / `*.ec2.internal`). It has **no effect** on Route 53 Private Hosted Zone
> names — those need VPC *association* instead (next section).

---

## 6. Route 53 Private Hosted Zone Across Peering

Custom names like `db.internal` live in a **Private Hosted Zone (PHZ)**. A PHZ resolves only inside
the VPCs it is **associated** with. Peering + the DNS option above do **nothing** for PHZ names —
you must explicitly associate the PHZ with the peer VPC.

```
                Route 53 PHZ  "internal"  (db.internal → 172.16.2.9)
                          │
            associated ───┼─── associated
                          │
   ┌──────────────────────┴───────────────────────┐
   │ VPC A (vpc-aaa111)            VPC B (vpc-bbb222)│
   │  EC2-A resolves db.internal ──► 172.16.2.9 ✅   │
   └────────────────────────────────────────────────┘
   If VPC A is NOT associated → EC2-A gets NXDOMAIN ❌
```

```bash
# Associate the PHZ with the peer VPC so its instances can resolve PHZ records
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z0123456789ABCDEFGHIJ \
  --vpc VPCRegion=us-east-1,VPCId=vpc-aaa111
```

⚠️ **Cross-account PHZ association is a two-step handshake**: the PHZ owner runs
`create-vpc-association-authorization`, then the *VPC owner* runs `associate-vpc-with-hosted-zone`.
A single account can do it in one call. And the target VPC still needs `enableDnsSupport` +
`enableDnsHostnames` for PHZ resolution to work.

```bash
# DNS account: authorize this exact VPC and Region.
aws --profile dns-owner route53 create-vpc-association-authorization \
  --hosted-zone-id Z0123456789ABCDEFGHIJ \
  --vpc VPCRegion=us-east-1,VPCId=vpc-aaa111

# Workload account: consume the authorization by associating its VPC.
aws --profile workload route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z0123456789ABCDEFGHIJ \
  --vpc VPCRegion=us-east-1,VPCId=vpc-aaa111

# DNS account: remove the now-consumed authorization record. The association remains.
aws --profile dns-owner route53 delete-vpc-association-authorization \
  --hosted-zone-id Z0123456789ABCDEFGHIJ \
  --vpc VPCRegion=us-east-1,VPCId=vpc-aaa111
```

Authorization grants the association operation; it does not grant permission to edit records in
the hosted zone. Keep zone ownership in the DNS account and delegate record changes through a
controlled pipeline.

See **[08_route53_private_hosted_zone.md](08_route53_private_hosted_zone.md)** for the full PHZ walkthrough.

---

## 7. Centralized & Hybrid DNS Beyond One Peer

Peering DNS options solve EC2 public-hostname answers between two VPCs. They do not create a
scalable hybrid namespace. For many accounts and on-premises DNS, put Route 53 VPC Resolver
endpoints and forwarding-rule ownership in a shared network/DNS account.

### 7.1 Namespace ownership

| Namespace | Authoritative location | Query path |
|-----------|------------------------|------------|
| `aws.example.com` | Route 53 PHZ associated with the endpoint/shared-services VPC and workload VPCs | AWS workloads use VPC Resolver directly; on-prem resolver conditionally forwards to inbound endpoint IPs |
| `corp.example.com` | On-premises DNS | AWS VPC Resolver matches a forwarding rule and sends the query through an outbound endpoint to on-prem resolvers |
| Public names | Public authoritative DNS | VPC Resolver recursively resolves them unless a more-specific private zone/rule overrides them |
| Reverse zones | Owner of the corresponding address space | Create explicit `in-addr.arpa`/`ip6.arpa` forwarding rules or PHZs rather than assuming forward rules cover reverse lookups |

Do not create the same private suffix independently in several accounts. A matching PHZ or a more
specific forwarding rule can shadow another source and return NXDOMAIN instead of "falling through"
to public DNS. Define one owner for each suffix and its subdomain delegations, then test positive and
negative answers from every network boundary.

### 7.2 Share Resolver rules, not an endpoint per VPC

Create one outbound endpoint with IPs in at least two AZs in each Region. A forwarding rule for
`corp.example.com` references that endpoint and at least two on-premises resolver IPs reached over
independent VPN/Direct Connect paths. Share the rule to accounts/OUs through AWS RAM; workload
accounts associate the shared rule with their VPCs.

The rule owner controls the rule, tags, targets, and outbound endpoint. A consumer can associate or
disassociate the shared rule but cannot edit/delete it. Sharing a rule indirectly shares its
outbound endpoint, so its security group, routes, query capacity, and change window are a central
dependency. Endpoints and rules are Regional—repeat the resilient design in every used Region.

For inbound resolution, configure on-premises conditional forwarders with **all** inbound endpoint
IPs. Resolver endpoints require at least two IPs; place them in different AZs and add another ENI
when both normal QPS headroom and maintenance availability require it. Allow TCP and UDP 53 through
endpoint SGs, NACLs, TGW/VPN/DX routes, and on-premises firewalls.

### 7.3 Prevent forwarding loops

A healthy ownership split is one-way per suffix:

```
on-prem: aws.example.com  → AWS inbound endpoint
AWS:     corp.example.com → AWS outbound endpoint → on-prem DNS
```

Never associate an outbound rule with a VPC when its target path leads back to an inbound endpoint
in that same VPC—directly or through on-premises DNS. That creates a query loop until hop/timeout
limits are reached. Review every new conditional forwarder in both systems, include the endpoint
IPs and suffix in the change record, and run a trace/query-log test before broad RAM association.

### 7.4 Central query logging

Associate VPC Resolver query logging configurations with every workload and endpoint VPC and send
them to a central CloudWatch Logs group, S3 bucket, or Firehose stream. Logs include source VPC/ENI,
query name/type, response code/data, and DNS Firewall action; they can cover VPC-originated,
inbound-endpoint, outbound-endpoint, and DNS Firewall queries.

Alert on `SERVFAIL`, unexpected public answers for private suffixes, blocked high-value domains,
query-volume anomalies, and repeated lookups that suggest a loop. Resolver caches answers by TTL,
so query logs record unique upstream lookups rather than every cache hit; application-side metrics
are still required to measure user-visible DNS latency and error rate.

### 7.5 Failure behavior and recovery

- If one endpoint ENI/AZ fails, the endpoint uses its surviving IPs; on-prem resolvers must actually
  be configured to try all inbound IPs, and capacity must fit on the survivors.
- If every outbound target or hybrid link is unavailable, forwarded names time out or return
  `SERVFAIL`. VPC Resolver does not safely invent a public fallback for a private namespace. Cached
  positive/negative answers can mask the failure until their TTL expires.
- PHZ and normal VPC Resolver names remain local when they do not depend on the failed forwarding
  rule. Test them separately so a central-link incident isn't mistaken for total VPC DNS failure.
- If a rule is unshared/deleted or an account moves out of the shared OU, VPC associations can be
  removed and remaining Resolver rules determine behavior. That can expose a public answer for a
  similarly named domain; monitor associations and block unintended public resolution at design
  time.

Recovery runbook: verify endpoint ENI status and query-volume metrics, test each target resolver IP
over TCP and UDP 53, inspect routing/TGW/VPN/DX, compare query logs on both sides, restore the rule
association if lost, then flush/test caches only where operationally safe. Recreate endpoints with
reserved IPs through IaC so on-premises forwarder configuration need not change.

---

## 8. Verification

```bash
# From EC2-A, confirm the answer is the PRIVATE IP, not a public one:
dig +short ip-172-16-2-9.ec2.internal      # expect: 172.16.2.9
nslookup db.internal                        # expect: PHZ record value (private)

# Confirm the Amazon resolver is the one answering (.2 of VPC A's CIDR = 10.0.0.2):
dig db.internal @10.0.0.2 +short

# Then confirm packets actually flow (DNS correct ≠ routed correctly):
nc -vz 172.16.2.9 443
```

---

## 9. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Hostname resolves to a **public IP** | Peering DNS option off | Enable `AllowDnsResolutionFromRemoteVpc` on the querying side ([§3b](#3b-the-peering-connection-dns-option-the-actual-cross-peer-toggle)) |
| Resolves to private IP **one way only** | DNS option enabled on one side | Enable it on **both** requester and accepter |
| **NXDOMAIN** for an EC2 hostname | `enableDnsHostnames=false` on the target VPC | `modify-vpc-attribute --enable-dns-hostnames` |
| **SERVFAIL / no resolver** | `enableDnsSupport=false` | `modify-vpc-attribute --enable-dns-support` |
| `db.internal` → **NXDOMAIN** but EC2 names work | PHZ not associated with querying VPC | `associate-vpc-with-hosted-zone` ([§6](#6-route-53-private-hosted-zone-across-peering)) |
| DNS returns correct **private IP** but connection **hangs** | Missing peer route (often the return route) | Add route to peer CIDR in **both** route tables ([§4a](#4a-route-tables--routes-to-the-peer-cidr-both-directions)) |
| Connection refused / timeout, routes fine | SG/NACL blocking | Allow the port inbound on target SG; NACL allow ephemeral return ([§4b](#4b-security-groups--allow-the-traffic-you-can-reference-the-peers-sg)) |
| SG peer-reference rejected | Inter-Region peering | Use **CIDR-based** SG rules instead of SG references |
| Cross-account PHZ association fails | Missing authorization step | Owner runs `create-vpc-association-authorization` first |
| Hybrid suffix intermittently `SERVFAIL` | Only one endpoint/target IP works, or one VPN/DX path is missing | Test every inbound/outbound and target-resolver IP over TCP+UDP 53; restore multi-AZ/link paths |
| Queries bounce between AWS and on-prem | Same suffix forwards from each side back to the other | Remove the circular rule and assign one authoritative owner/direction per suffix |
| Shared rule disappeared from a VPC | RAM share/OU membership changed or association was removed | Restore the share and VPC association; audit which remaining rule answered during the gap |

---

## Key Exam Points

- ✅ Across peering, instances by default resolve a peer's DNS name to its **PUBLIC IP** (or get
  nothing useful) — **not** the private IP.
- ✅ To resolve to **PRIVATE IPs**, enable **DNS resolution on the peering connection**
  (`AllowDnsResolutionFromRemoteVpc`) on the side(s) that need it — typically **both**.
- ✅ Prerequisites: **`enableDnsSupport` + `enableDnsHostnames` = true on both VPCs**
  (`enableDnsHostnames` is **off by default** in custom VPCs).
- ✅ DNS resolution is **separate** from reachability: you still need **routes to the peer CIDR in
  both route tables** and **SG/NACL rules** allowing the traffic.
- ✅ Same-Region peering lets SGs **reference the peer VPC's security group ID**; inter-Region peering
  forces **CIDR-based** rules and disallows SG references.
- ✅ The peering DNS toggle governs only **Amazon default hostnames**. For a **Route 53 Private
  Hosted Zone**, you must **`associate-vpc-with-hosted-zone`** with the peer VPC (cross-account =
  two-step authorize-then-associate).
- ⚠️ Peering is **non-transitive** — A↔B and B↔C does not give A↔C, and there's no transitive DNS either.
- ✅ Central hybrid DNS uses multi-AZ inbound/outbound Resolver endpoints and RAM-shared forwarding
  rules. Consumers associate shared rules; only the owner edits them and their endpoint targets.
- ✅ Query logs expose answer codes and paths but not Resolver cache hits. Avoid suffix loops and
  test failure of every endpoint IP, target resolver, and hybrid link.

---

**Next**: [07_route53_public_hosted_zone.md — Public Hosted Zone: Domains, Alias Records & Failover](07_route53_public_hosted_zone.md)
