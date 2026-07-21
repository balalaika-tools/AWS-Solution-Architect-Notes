# Practical: Route 53 Private Hosted Zone & Split-Horizon DNS

> **Who this is for**: Engineers giving internal resources friendly names (`db.internal`,
> `cache.internal`) that resolve **only inside their VPCs**, and serving the **same domain**
> differently on the public internet vs internally (split-horizon). We cover association, the
> mandatory VPC DNS attributes, when records do and don't resolve, and split-horizon setup.
>
> Prerequisites: **[Route 53](../08_dns_edge/02_route_53.md)** and, if peering is involved,
> **[06_vpc_peering_dns_resolution.md](06_vpc_peering_dns_resolution.md)** (PHZ across peering).

---

## 1. What a Private Hosted Zone Is

A **Private Hosted Zone (PHZ)** is a Route 53 zone that is authoritative **only within the VPCs you
associate it with**. Queries from the public internet never see it. Resolution happens through the
**Amazon-provided DNS resolver** (`.2` / `169.254.169.253`) inside each associated VPC.

```
   ┌─────────────────────── PHZ "internal" ───────────────────────┐
   │  db.internal     A  →  10.0.5.20  (RDS private IP / endpoint) │
   │  cache.internal  CNAME → my-redis.xxxx.cache.amazonaws.com    │
   └───────────────────────────┬───────────────────────────────────┘
                 associated with VPCs:
        ┌────────────────┐          ┌────────────────┐
        │ vpc-app (prod) │          │ vpc-data (prod)│
        │ EC2 resolves   │          │ EC2 resolves   │
        │ db.internal ✅ │          │ db.internal ✅ │
        └────────────────┘          └────────────────┘
   An instance in a NON-associated VPC → NXDOMAIN ❌
```

> **Key insight**: A PHZ has **no NS delegation** on the public internet. Its scope is exactly the
> set of VPCs it's associated with — nothing more, nothing less.

---

## 2. Prerequisite: VPC DNS Attributes

A PHZ only resolves in a VPC where **both** attributes are on. `enableDnsHostnames` is **off by default
in custom VPCs**, which is the most common "PHZ not resolving" cause.

```bash
aws ec2 modify-vpc-attribute --vpc-id vpc-app --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id vpc-app --enable-dns-hostnames
```

| Attribute | Required for PHZ? | Default in custom VPC |
|-----------|-------------------|------------------------|
| `enableDnsSupport`  | ✅ yes (resolver must answer) | `true` |
| `enableDnsHostnames`| ✅ yes | **`false`** |

---

## 3. Create the PHZ and Associate VPC(s)

```bash
# Create a private zone, associating the first VPC at creation
aws route53 create-hosted-zone \
  --name internal \
  --caller-reference "$(date +%s)" \
  --hosted-zone-config PrivateZone=true \
  --vpc VPCRegion=us-east-1,VPCId=vpc-app
# -> HostedZoneId: Z0PRIVATE123456

# Associate additional VPCs (same account)
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z0PRIVATE123456 \
  --vpc VPCRegion=us-east-1,VPCId=vpc-data
```

⚠️ **Cross-account association** is a two-step handshake: the **PHZ owner** authorizes, then the
**VPC owner** associates.

```bash
# In PHZ owner account:
aws route53 create-vpc-association-authorization \
  --hosted-zone-id Z0PRIVATE123456 --vpc VPCRegion=us-east-1,VPCId=vpc-other-acct
# In the VPC owner account:
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z0PRIVATE123456 --vpc VPCRegion=us-east-1,VPCId=vpc-other-acct

# Back in the PHZ owner account, remove the one-time authorization after the
# association succeeds. This does not remove the VPC association.
aws route53 delete-vpc-association-authorization \
  --hosted-zone-id Z0PRIVATE123456 --vpc VPCRegion=us-east-1,VPCId=vpc-other-acct
```

Create one authorization per VPC. Treat the zone association as infrastructure
as code in **both** accounts: the DNS account owns the authorization and zone,
while the workload account owns the association. This avoids an untracked
console handshake and makes reassociation after an accidental deletion
repeatable.

### Internal records (point at private IPs / endpoints)

```json
{
  "Changes": [
    { "Action": "UPSERT", "ResourceRecordSet": {
        "Name": "db.internal", "Type": "A", "TTL": 60,
        "ResourceRecords": [{ "Value": "10.0.5.20" }] }},   // RDS instance private IP
    { "Action": "UPSERT", "ResourceRecordSet": {
        "Name": "cache.internal", "Type": "CNAME", "TTL": 60,
        "ResourceRecords": [{ "Value": "my-redis.abcd.0001.use1.cache.amazonaws.com" }] }}
  ]
}
```

💡 For RDS, prefer a **CNAME to the RDS endpoint DNS name** over a hard-coded A record — the endpoint
follows the instance/Multi-AZ failover automatically, an A record to a private IP does not.

---

## 4. When PHZ Records Do / Don't Resolve

| Scenario | Resolves? |
|----------|-----------|
| EC2 in an **associated** VPC, both DNS attrs on | ✅ resolves to record value |
| EC2 in a VPC **not associated** with the PHZ | ❌ NXDOMAIN |
| `enableDnsSupport = false` | ❌ resolver disabled |
| `enableDnsHostnames = false` | ❌ PHZ won't resolve |
| Query from **on-prem** (over VPN/DX) | ❌ unless via **Route 53 Resolver inbound endpoint** ([09](09_route53_resolver_hybrid_dns.md)) |
| **Peered** VPC, PHZ associated with that peer VPC | ✅ |
| **Peered** VPC, PHZ **not** associated (relying on peering DNS toggle) | ❌ — the peering DNS option does **not** cover PHZ names |
| Same name exists in **public zone too** (split-horizon) | ✅ PHZ answer wins **inside** associated VPCs |

> **Rule**: PHZ resolution is governed by **VPC association**, never by peering's
> `AllowDnsResolutionFromRemoteVpc`. To resolve a PHZ across peering, associate the PHZ with the
> peer VPC (see [06 §6](06_vpc_peering_dns_resolution.md#6-route-53-private-hosted-zone-across-peering)).

---

## 5. Split-Horizon DNS (same domain, two answers)

You can have a **public** hosted zone and a **private** hosted zone with the **same name**
(`example.com`). Inside associated VPCs, the **private** zone wins; everyone else sees the public one.

```
                          example.com
        ┌──────────────────────────┬──────────────────────────┐
   PUBLIC hosted zone          PRIVATE hosted zone (assoc. vpc-app)
   api.example.com → ALB           api.example.com → 10.0.5.30 (internal NLB)
        │                                  │
   Internet client                   EC2 in vpc-app
   gets PUBLIC ALB IP  ✅             gets PRIVATE 10.0.5.30  ✅
```

```bash
# Public zone (delegated, internet-facing) — see file 07
aws route53 create-hosted-zone --name example.com --caller-reference pub-$(date +%s)
# Private zone, SAME name, associated with the internal VPC
aws route53 create-hosted-zone --name example.com --caller-reference priv-$(date +%s) \
  --hosted-zone-config PrivateZone=true \
  --vpc VPCRegion=us-east-1,VPCId=vpc-app
```

✅ Use case: internal services hit a private NLB/private ALB by the public hostname (avoiding NAT and
internet hops), while external users hit the public endpoint — **same URL, no app config change**.

⚠️ Split-horizon precedence is by **most-specific match within the VPC scope**. If the private zone is
`example.com` and a client queries `extra.sub.example.com` that only exists in the public zone, the
private zone (being authoritative for `example.com`) can return NXDOMAIN inside the VPC. Keep the two
zones' record sets aligned for any name that must work in both.

The same rule applies to overlapping private zones. If a VPC is associated with
both `example.com` and `dev.example.com`, a query for
`api.dev.example.com` goes to `dev.example.com`. If that zone lacks the record,
Resolver returns NXDOMAIN; it does **not** retry the less-specific zone. Also
remember that a matching **Resolver forwarding rule takes precedence** over a
PHZ with the same domain. Maintain a namespace ownership table so two teams do
not independently claim or forward the same suffix.

---

## 6. Verification

```bash
# From an EC2 in an associated VPC:
dig +short db.internal                 # -> 10.0.5.20
dig db.internal @10.0.0.2 +short       # confirm the Amazon resolver (.2) answers
nslookup api.example.com               # split-horizon: expect the PRIVATE value here
```

---

## 7. Production Multi-Account Operations

A common organization design puts shared private zones in a dedicated DNS or
network account. Workload accounts can change their own VPC associations, but
only the DNS platform team changes production records and zone-level policy.
Use separate zones or tightly scoped IAM roles where application teams need
delegated ownership; do not hand every account write access to one enterprise
zone.

Resolver **rules**, unlike PHZ associations, can be shared to workload accounts
through AWS RAM. The network account owns the outbound endpoint and forwarding
rules; each workload account accepts the share and associates the rule with its
VPC. Keep this control plane distinct:

| Resource | Typical owner | Consumer action |
|----------|---------------|-----------------|
| PHZ and records | DNS account | Associate each authorized VPC |
| Outbound endpoint and rule | Network/DNS account | Accept RAM share and associate rule |
| VPC DNS attributes and resolver-rule association | Workload account | Keep enabled and managed as code |
| Query-log destination and retention | Security/log-archive account | Grant delivery permissions |

Enable **Route 53 Resolver query logging** for every production VPC and deliver
it centrally to CloudWatch Logs, S3, or Firehose. Logs include the query name,
type, source VPC/resource, response, and DNS Firewall action, so they are useful
for incident response as well as NXDOMAIN diagnosis. Protect the destination,
set a retention/lifecycle policy, and budget for ingestion and storage; logging
high-volume DNS indefinitely can cost more than the hosted zones themselves.

### Recovery tests

- Export or manage all records and VPC associations through version-controlled
  infrastructure as code. Route 53's highly available data plane is not a
  substitute for recovering an accidentally deleted zone or record.
- Quarterly, remove a **test** VPC association, confirm queries fail after
  cached TTLs expire, then repeat the authorize → associate workflow. Never do
  this first in production.
- Alarm on unexpected NXDOMAIN/SERVFAIL growth in query logs and on
  `DisassociateVPCFromHostedZone`/record-change API activity in CloudTrail.
- If a Resolver endpoint or forwarding rule fails, restore that component from
  code and verify both authoritative answers and network reachability. Do not
  add public DNS as a fallback for a private namespace: that can leak queries or
  send clients to the wrong endpoint.
- Keep record TTLs aligned to the recovery objective. Low TTLs speed a planned
  target change but increase query volume; clients can also cache longer than
  expected, so validate the real application resolver behavior.

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `db.internal` → NXDOMAIN on an instance | VPC not associated with the PHZ | `associate-vpc-with-hosted-zone` |
| PHZ resolves nowhere | `enableDnsSupport` / `enableDnsHostnames` off | `modify-vpc-attribute` both flags on |
| Cross-account associate fails | Authorization step skipped | Owner runs `create-vpc-association-authorization` first |
| Cross-account association disappeared | VPC owner disassociated it, or IaC drift | Re-authorize in the PHZ account, reassociate in the VPC account, then delete the authorization |
| Split-horizon: internal box still gets public IP | Private zone not associated with that VPC, or wrong VPC | Associate the private zone with the querying VPC |
| A name under an overlapping child zone returns NXDOMAIN | More-specific PHZ matched but lacks the record | Create the record in the child zone or remove/fix the overlapping association; Resolver does not fall back |
| PHZ record exists but on-prem answer comes from elsewhere | Resolver rule overlaps the PHZ suffix | Fix namespace ownership and narrow/remove the forwarding rule; a matching rule wins |
| On-prem can't resolve `*.internal` | No inbound resolver endpoint | Set up **Route 53 Resolver inbound endpoint** ([09](09_route53_resolver_hybrid_dns.md)) |
| Peered VPC can't resolve PHZ name | Relied on peering DNS toggle | Associate the PHZ with the peer VPC (toggle is for EC2 default names only) |
| RDS failover breaks the A record | Hard-coded private IP | Use a **CNAME to the RDS endpoint** instead |

---

## Key Exam Points

- ✅ A PHZ resolves **only inside the VPCs it is associated with** — no public delegation, no NS records
  on the internet.
- ✅ Resolution requires **`enableDnsSupport` + `enableDnsHostnames` = true** on the VPC
  (`enableDnsHostnames` is **off by default** in custom VPCs).
- ✅ Add VPCs with **`associate-vpc-with-hosted-zone`**; **cross-account** needs the two-step
  **authorize → associate** handshake. Delete the one-time authorization after success.
- ✅ **Split-horizon** = same domain name in both a public and a private zone; inside associated VPCs
  the **private** zone's answer wins.
- ✅ To resolve a PHZ across **peering**, **associate the PHZ with the peer VPC** — the peering DNS
  toggle (`AllowDnsResolutionFromRemoteVpc`) covers only Amazon **default EC2 hostnames**, not PHZ names.
- ✅ On-prem resolution of PHZ names needs a **Route 53 Resolver inbound endpoint** (next file).
- ✅ Centralize ownership, query logging, and recovery automation; use **AWS RAM**
  to share Resolver rules, not as a replacement for PHZ-to-VPC association.

---

**Next**: [09_route53_resolver_hybrid_dns.md — Hybrid DNS with Route 53 Resolver Endpoints](09_route53_resolver_hybrid_dns.md)
