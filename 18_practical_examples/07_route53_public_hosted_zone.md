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
| Health-check integration | Manual | Native for supported alias targets (`EvaluateTargetHealth`; not CloudFront) |

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

This example deliberately uses both an explicit application health check and ALB target-health
evaluation. Route 53 considers that alias healthy only when **both** pass. For a simple ALB alias,
`EvaluateTargetHealth=true` is usually sufficient; add an explicit check only when its independent
end-to-end signal is worth the extra failure mode.

✅ When the primary is healthy, Route 53 returns it. After the health threshold is crossed, new DNS
answers use the secondary. User-visible failover also includes resolver/client caching, reused
connections, retries, and secondary readiness—not just health-check detection plus authoritative
DNS processing.

---

## 5. TTL Considerations

| Record style | TTL behavior |
|---|---|
| Standard A/CNAME | You set TTL explicitly (e.g. 300s). Lower = faster failover, more queries/cost. |
| **Alias to AWS resource** | **No TTL field** — Route 53 uses the target service's own TTL (typically ~60s for ELB). |
| Non-alias record during cutover | Drop TTL to **60s at least one old TTL before** a migration, then raise it after. Alias TTL is controlled by its target service. |

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

## 7. Production Multi-Account, Multi-Region DNS

A production DNS failover is the final switch in a larger recovery system. Both Regional stacks
must already have capacity, certificates, WAF/security policy, observability, secrets/keys, and a
known data state before Route 53 sends traffic to them.

### 7.1 Delegate a subdomain to a workload account

Keep `example.com` in a central DNS account and delegate `prod.example.com` to a production account.
The workload team can change only its child zone; the DNS team retains the parent trust boundary.

```bash
# Production account: create child zone and obtain its four authoritative name servers.
aws --profile prod route53 create-hosted-zone \
  --name prod.example.com --caller-reference prod-2026-07-21

aws --profile prod route53 get-hosted-zone \
  --id ZCHILD123456789 \
  --query 'DelegationSet.NameServers'

# DNS account: create an NS record named prod.example.com in the example.com parent zone.
aws --profile dns route53 change-resource-record-sets \
  --hosted-zone-id ZPARENT12345678 \
  --change-batch '{
    "Changes":[{"Action":"UPSERT","ResourceRecordSet":{
      "Name":"prod.example.com","Type":"NS","TTL":300,
      "ResourceRecords":[
        {"Value":"ns-111.awsdns-11.com"},{"Value":"ns-222.awsdns-22.net"},
        {"Value":"ns-333.awsdns-33.org"},{"Value":"ns-444.awsdns-44.co.uk"}
      ]}}]}'
```

Verify the child directly against each assigned name server, then through public recursive
resolvers. Put all records below `prod.example.com` in the child zone; duplicates in parent and
child create cache-dependent answers. During teardown remove the parent delegation before deleting
the child zone, after its NS TTL expires, to avoid a dangling delegation.

### 7.2 Enable DNSSEC without breaking the chain of trust

DNSSEC signs answers so validating resolvers can detect tampering; it does not encrypt queries or
replace TLS. Roll it out in this order:

1. Enable query/availability monitoring. Lower the zone's maximum TTL and SOA negative-cache values
   in advance so a failed rollout can be reversed promptly.
2. Create/use an asymmetric KMS customer managed key in `us-east-1` with key spec
   `ECC_NIST_P256` and signing usage, plus the required Route 53 key policy. Create the key-signing
   key (KSK), enable hosted-zone signing, and wait for `INSYNC`.
3. Give the generated DS information to the **parent-zone owner** (or registrar) and add the DS
   record only after signing is active. For the delegated child above, the central DNS account
   publishes the child's DS record in `example.com`.
4. Validate signed positive and negative answers from multiple public validating resolvers. Alarm
   on `DNSSECInternalFailure` and `DNSSECKeySigningKeysNeedingAction`.

An incorrect DS/KSK relationship makes validating resolvers return `SERVFAIL`. To disable DNSSEC,
remove the DS from the parent first, wait out its TTL, and only then disable signing. Disabling the
signer while the parent still asserts a DS breaks the domain.

### 7.3 Understand health-check failure modes

| Failure mode | Result and prevention |
|--------------|-----------------------|
| Check uses the same failover name it controls | Circular/unpredictable resolution; check a stable Regional name such as `use1-origin.prod.example.com` |
| Endpoint is private | Public Route 53 health checkers cannot reach RFC1918/private endpoints; use ALB target health, a CloudWatch-alarm health check, or a public health facade |
| SG/firewall blocks health checkers | False failover; allow the AWS-managed Route 53 health-check prefix list on the health port |
| `/health` tests an optional/shared dependency | Both Regions can flap together; make the routing signal represent whether that Regional stack can safely serve traffic |
| Secondary has no check | Route 53 assumes it is usable when primary fails; continuously test standby readiness separately |
| Primary and secondary checks are both unhealthy | Route 53's fail-open behavior returns the **primary** record; alert and retain an operator-controlled recovery path |
| Alias has explicit check plus `EvaluateTargetHealth=true` | Both signals must be healthy; use this only intentionally |

Route 53 checks periodically, not when a user query arrives. Record health-check status and alarms,
but also run multi-location synthetic transactions against both Regional origins and the public
failover name.

### 7.4 Use ARC routing controls for guarded operator failover

Automatic endpoint health is useful for clear failures. For a partial outage, corrupt data, or a
dependency that monitors cannot interpret safely, **Amazon Application Recovery Controller (ARC)**
routing controls provide explicit on/off switches. Their special Route 53 health checks control
failover records; they do not monitor the application themselves.

Create a routing control per Regional cell and an assertion safety rule that requires at least one
cell to remain `On`. Use gating rules for changes that must be authorized. The ARC cluster exposes
five Regional **data-plane** endpoints; the DR script must list/update state through those endpoints,
trying another on failure, rather than depending on the console/control plane. Store tested
break-glass credentials and routing-control ARNs outside the failed Region. ARC controls can also be
shared cross-account when centralized recovery ownership is required.

### 7.5 Build both Regional endpoints before DNS

`use1-origin.prod.example.com` and `usw2-origin.prod.example.com` should each front an independent
ALB/application fleet with N-1 AZ capacity. Replicate configuration, certificates, WAF rules,
container/AMI artifacts, secrets, and KMS access through explicit pipelines. Define data behavior
separately: synchronous/global database, asynchronous replica with measured lag, or read-only
maintenance response. DNS cannot repair missing quotas, stale data, or a secret that exists only in
the failed Region.

Before enabling failover, test each Regional name directly while the other Region is unavailable.
Validate writes, background jobs, callbacks, allow lists, email/payment side effects, logging, and
alarms. Fence writers or use the database's managed promotion workflow so DNS failover cannot create
split brain.

### 7.6 Measure TTL and application RTO

Measure these timestamps in a drill:

```
failure begins
→ health/decision threshold crossed
→ Route 53/ARC state changes
→ public resolvers return the new answer after cached TTL
→ clients refresh DNS and abandon reused connections
→ first successful transaction in the recovery Region
```

Query the authoritative Route 53 name servers and several public resolvers repeatedly; record the
answer and remaining TTL. Instrument client DNS caching, connection pool/keep-alive lifetime, retry
backoff, and success rate. Alias records don't expose a user-set TTL, so use the target service's TTL
in the experiment. The measured RTO is the last step above; a 60-second DNS TTL is not a 60-second
application RTO.

### 7.7 Tested failover and failback

Failover runbook:

1. Confirm the recovery Region is healthy, scaled, and within its quota; inspect replication lag
   and state the accepted RPO.
2. Fence or stop primary writes, promote the recovery data store if required, and disable duplicate
   schedulers/consumers.
3. Change ARC routing controls or allow the tested Route 53 health policy to move new DNS answers.
4. Run synthetic read/write journeys from several networks; monitor error/latency, data integrity,
   queues, and old-Region connections until the measured RTO is met.

Failback is another migration, not "mark primary healthy": repair the old Region, resynchronize it
from the current writer, test it through its Regional name, confirm capacity and data checksums, then
move traffic in a controlled window. Failover records switch as a unit; if gradual return is needed,
use a separately tested weighted stage or application routing. Keep the recovery Region ready and a
reversal point until the observation window closes. Record actual RTO/RPO and feed every miss back
into TTL, client, capacity, and runbook design.

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Domain doesn't resolve anywhere | Registrar NS don't match Route 53 NS | Copy all 4 R53 NS into registrar; verify with `dig NS example.com @8.8.8.8` |
| Resolves to old IP long after change | TTL too high / resolver cache | Lower TTL ahead of changes; wait out the old TTL |
| Can't create record at apex pointing to ALB | Tried a **CNAME** at apex | Use an **Alias A** record instead ([§3](#3-create-an-aalias-record-to-an-alb-or-cloudfront)) |
| CloudFront alias gives SSL error / 403 | Domain not in distribution CNAMEs / wrong-region ACM cert | Add the alt domain name; ACM cert must be in **us-east-1** |
| Failover never flips | `EvaluateTargetHealth=false` or no HealthCheckId on primary | Attach health check to PRIMARY; set `EvaluateTargetHealth=true` |
| `dig NS example.com` shows two NS sets | Delegation mismatch (parent vs zone disagree) | Make registrar NS exactly equal the zone's `DelegationSet` |
| Health check flapping | Threshold too tight / `/health` returns non-2xx | Raise `FailureThreshold`; ensure health path returns 200 |
| Validating resolvers return `SERVFAIL` after DNSSEC change | Parent DS doesn't match active child KSK, or signer/key has failed | Check KSK/DS, DNSSEC CloudWatch alarms, and rollback in the safe parent-first order |
| DNS changed but users remain on old Region | Resolver/JVM cache or long-lived connection outlives authoritative answer | Measure TTLs, client DNS cache, keep-alive, connection draining, and retry behavior |
| Failover routes to an unready secondary | Secondary had no effective health/readiness/capacity test | Test the Regional origin continuously and add capacity/data readiness to the runbook |
| ARC update fails during incident | Script uses one unavailable endpoint or the control plane | Try all five cluster data-plane endpoints with break-glass credentials; inspect safety-rule result |

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
  failover-critical non-alias records, then add client caching/connection time to measured RTO.
- ⚠️ CNAME is illegal at the zone apex; Alias is the AWS-native workaround.
- ✅ Cross-account subdomain delegation is an NS record in the parent zone pointing to the child
  account's four authoritative name servers; remove delegation before deleting the child zone.
- ✅ DNSSEC requires active zone signing before the parent DS record. Remove DS and wait its TTL
  before disabling signing, or validating resolvers return `SERVFAIL`.
- ✅ ARC routing controls provide guarded operator-directed DNS failover through five Regional
  data-plane endpoints; safety rules can require at least one Regional cell to stay on.
- ✅ A tested failback resynchronizes/fences data, verifies the old Region directly, measures client
  convergence, and retains a reversal point. DNS health alone is not recovery readiness.

---

**Next**: [08_route53_private_hosted_zone.md — Private Hosted Zones & Split-Horizon DNS](08_route53_private_hosted_zone.md)
