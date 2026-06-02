# Practical: Route 53 Public Hosted Zone — Domains, Alias Records & Failover

> **Who this is for**: Engineers standing up `www.example.com` in front of an ALB or CloudFront
> and wanting DNS-based failover. We walk the full path: register/delegate a domain, wire up NS
> records, create A/Alias records, attach a health check, and build an active-passive failover pair.
>
> Prerequisite: **[Route 53](../08_dns_edge/02_route_53.md)** (hosted zones, record types,
> routing policies).

---

## 1. What a Public Hosted Zone Is

A **public hosted zone** is the authoritative DNS container for a domain on the **public internet**.
When you create the zone `example.com`, Route 53 gives it **four NS records** (a set of name servers,
e.g. `ns-123.awsdns-45.com`) and an **SOA** record. The internet finds your records by following the
delegation chain from the root → TLD → your NS servers.

```
Browser asks "where is www.example.com?"
   │
   ▼
Root NS  ──►  .com TLD NS  ──►  Route 53 NS (ns-123.awsdns-45.com)  ──►  answer: 1.2.3.4
   (the .com registry must point .com → your Route 53 NS = "delegation")
```

> **Key insight**: Creating a hosted zone does nothing until the **parent zone delegates to it**.
> The registrar (who controls the `.com` entry for your domain) must list your Route 53 NS records.

---

## 2. Register or Delegate the Domain

Two cases:

**(a) Domain registered *in* Route 53** — delegation is automatic. Route 53 creates the public hosted
zone and sets the domain's NS to match. Nothing else to do.

**(b) Domain registered *elsewhere* (GoDaddy, Namecheck, etc.)** — you must copy Route 53's NS records
into the external registrar's "name servers" setting.

```bash
# 1. Create the public hosted zone
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference "$(date +%s)"

# 2. Read the 4 NS records Route 53 assigned
aws route53 get-hosted-zone --id Z0123456789ABCDEFGHIJ \
  --query 'DelegationSet.NameServers'
# -> [ "ns-123.awsdns-45.com", "ns-678.awsdns-90.net", "ns-901.awsdns-12.org", "ns-234.awsdns-56.co.uk" ]
```

Then, **at the external registrar**, replace the existing name servers with those four values.

⚠️ Set **all four** NS records, exactly, with **no trailing dot** in most registrar UIs. Missing or
mismatched NS = your zone is invisible to the world.

---

## 3. Create an A/Alias Record to an ALB or CloudFront

You point the apex (`example.com`) and/or `www` at AWS resources. For AWS targets (ALB, CloudFront,
S3 website, API Gateway), use an **Alias record**, not a CNAME.

### Why Alias (especially at the apex)

DNS forbids a CNAME at the **zone apex** (`example.com` itself) because the apex already has SOA/NS
records and a CNAME must be the only record at a name. Route 53 **Alias** records sidestep this — they
look like A/AAAA records to clients but resolve dynamically to the AWS resource's IPs.

| | CNAME | Route 53 Alias |
|---|---|---|
| Works at zone apex (`example.com`) | ❌ | ✅ |
| Target | Any hostname | AWS resource or another R53 record |
| Cost per query | Charged | **Free** for AWS-resource aliases |
| Health-check integration | Manual | Native (`EvaluateTargetHealth`) |

```json
// change-batch.json — apex Alias to an ALB
{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "example.com",
      "Type": "A",
      "AliasTarget": {
        "HostedZoneId": "Z35SXDOTRQ7X7K",                       // the ALB's zone ID (region-specific, fixed value)
        "DNSName": "dualstack.my-alb-123456.us-east-1.elb.amazonaws.com",
        "EvaluateTargetHealth": true
      }
    }
  }]
}
```

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z0123456789ABCDEFGHIJ \
  --change-batch file://change-batch.json
```

💡 For **CloudFront**, the `AliasTarget.HostedZoneId` is always **`Z2FDTNDATAQYW2`** and the `DNSName`
is `dxxxx.cloudfront.net`. CloudFront alias also requires the domain be in the distribution's
**Alternate Domain Names (CNAMEs)** and a matching ACM cert (in **us-east-1** for CloudFront).

---

## 4. Health Check + Active-Passive Failover

Goal: serve from a **primary** ALB; if it's unhealthy, automatically answer with a **secondary**
(e.g. a static S3/CloudFront "maintenance" site or a DR region ALB).

```
                example.com  (A / Alias, Failover policy)
                       │
        ┌──────────────┴───────────────┐
        │ PRIMARY                       │ SECONDARY
        │ ALB us-east-1                 │ DR ALB us-west-2
        │ health check ✅ → served      │ served only when primary ❌
        └───────────────────────────────┘
```

### 4a. Create the health check (on the primary endpoint)

```bash
aws route53 create-health-check \
  --caller-reference "primary-$(date +%s)" \
  --health-check-config '{
     "Type": "HTTPS",
     "FullyQualifiedDomainName": "primary.example.com",
     "Port": 443,
     "ResourcePath": "/health",
     "RequestInterval": 30,
     "FailureThreshold": 3
  }'
# -> HealthCheckId: hc-abc123
```

### 4b. Two failover records sharing the same name

```json
{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "example.com", "Type": "A",
        "SetIdentifier": "primary",
        "Failover": "PRIMARY",
        "HealthCheckId": "hc-abc123",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "dualstack.primary-alb.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "example.com", "Type": "A",
        "SetIdentifier": "secondary",
        "Failover": "SECONDARY",
        "AliasTarget": {
          "HostedZoneId": "Z1H1FL5HABSF5",
          "DNSName": "dr-alb.us-west-2.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }
  ]
}
```

✅ When `hc-abc123` is healthy, Route 53 returns the primary. On 3 consecutive failures it returns the
secondary. Failover speed ≈ health-check detection time **+ the record's TTL** (clients cache the old
answer until TTL expires).

---

## 5. TTL Considerations

| Record style | TTL behavior |
|---|---|
| Standard A/CNAME | You set TTL explicitly (e.g. 300s). Lower = faster failover, more queries/cost. |
| **Alias to AWS resource** | **No TTL field** — Route 53 uses the target service's own TTL (typically ~60s for ELB). |
| Apex during cutover | Drop TTL to **60s a day before** a migration so changes propagate fast, raise it after. |

⚠️ A long TTL (e.g. 86400s) means clients can keep hitting a dead primary for up to a day after
failover. For failover-critical records, keep TTL low (60–300s).

---

## 6. Public Resolution Path (end to end)

```
 Client          Recursive resolver (ISP)       .com TLD        Route 53 (authoritative)
   │                      │                         │                     │
   │── www.example.com ──►│                         │                     │
   │                      │── who serves example.com? ──────────────►     │
   │                      │◄── NS: ns-123.awsdns-45.com ──────────────    │
   │                      │── A www.example.com? ──────────────────────►  │
   │                      │            (Route 53 evaluates failover/health)│
   │                      │◄── A 1.2.3.4 (healthy primary) ─────────────   │
   │◄── 1.2.3.4 ──────────│  (cached for TTL)                              │
   ▼
 connects to ALB 1.2.3.4
```

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Domain doesn't resolve anywhere | Registrar NS don't match Route 53 NS | Copy all 4 R53 NS into registrar; verify with `dig NS example.com @8.8.8.8` |
| Resolves to old IP long after change | TTL too high / resolver cache | Lower TTL ahead of changes; wait out the old TTL |
| Can't create record at apex pointing to ALB | Tried a **CNAME** at apex | Use an **Alias A** record instead ([§3](#3-create-an-aalias-record-to-an-alb-or-cloudfront)) |
| CloudFront alias gives SSL error / 403 | Domain not in distribution CNAMEs / wrong-region ACM cert | Add the alt domain name; ACM cert must be in **us-east-1** |
| Failover never flips | `EvaluateTargetHealth=false` or no HealthCheckId on primary | Attach health check to PRIMARY; set `EvaluateTargetHealth=true` |
| `dig NS example.com` shows two NS sets | Delegation mismatch (parent vs zone disagree) | Make registrar NS exactly equal the zone's `DelegationSet` |
| Health check flapping | Threshold too tight / `/health` returns non-2xx | Raise `FailureThreshold`; ensure health path returns 200 |

---

## Key Exam Points

- ✅ A public hosted zone is authoritative only once the **registrar's NS records point to Route 53's 4
  NS records** (delegation). Registering the domain *in* Route 53 does this automatically.
- ✅ Use an **Alias** record (not CNAME) for AWS targets — and it's the **only** way to point the
  **zone apex** at an ALB/CloudFront/S3. Alias queries to AWS resources are **free**.
- ✅ CloudFront alias zone ID is always **`Z2FDTNDATAQYW2`**; the ACM cert for CloudFront must be in
  **us-east-1**.
- ✅ **Failover routing** = PRIMARY + SECONDARY records, same name, with a **health check** on primary
  and `EvaluateTargetHealth=true`.
- ✅ Failover/propagation latency = health-check detection **+ TTL**; keep TTL low (60–300s) for
  failover-critical records.
- ⚠️ CNAME is illegal at the zone apex; Alias is the AWS-native workaround.

---

**Next**: [08_route53_private_hosted_zone.md — Private Hosted Zones & Split-Horizon DNS](08_route53_private_hosted_zone.md)
