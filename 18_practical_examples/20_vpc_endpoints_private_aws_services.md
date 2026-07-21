# Private AWS Service Access with VPC Endpoints

> **Scenario**: Workloads in private subnets need S3, ECR, CloudWatch Logs,
> Secrets Manager, and SQS without public IPs and without paying NAT data charges
> for AWS-service traffic.

---

## 1. Target Architecture

```
                     VPC 10.0.0.0/16
 ┌────────────────────────────────────────────────────────────┐
 │ Private subnet A/B                                         │
 │  ECS task / EC2 / Lambda ENI                               │
 │        │                                                   │
 │        ├── S3/DynamoDB ── route table ──► Gateway endpoint │
 │        │                                                   │
 │        └── ECR/Logs/Secrets/SQS ──443──► Interface endpoint│
 │                                      endpoint ENIs + SG    │
 └────────────────────────────────────────────────────────────┘
```

Use **gateway endpoints** for S3/DynamoDB and **interface endpoints** for most
other AWS services.

---

## 2. Security Groups

Create one SG for the workload and one SG for interface endpoints:

```
sg-app
  outbound: 443 to sg-vpce
  outbound: 443 to S3 prefix list if outbound is restrictive

sg-vpce
  inbound: 443 from sg-app
  outbound: no special rule needed for stateful response traffic
```

Security groups are stateful, so return traffic is automatic. If you use custom
NACLs, allow the matching return traffic because NACLs are stateless.

---

## 3. S3 Gateway Endpoint

Gateway endpoints do not create ENIs and do not have security groups. They add
routes to the route tables you choose.

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0123 \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-private-a rtb-private-b
```

Optional endpoint policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::example-private-bucket/*"
    }
  ]
}
```

Bucket policy to require the endpoint path:

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": [
    "arn:aws:s3:::example-private-bucket",
    "arn:aws:s3:::example-private-bucket/*"
  ],
  "Condition": {
    "StringNotEquals": {
      "aws:sourceVpce": "vpce-0123456789abcdef0"
    }
  }
}
```

---

## 4. Interface Endpoints

Create endpoint ENIs in private subnets and attach `sg-vpce`. Enable private DNS
when supported so normal SDK hostnames resolve privately.

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0123 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.secretsmanager \
  --subnet-ids subnet-private-a subnet-private-b \
  --security-group-ids sg-vpce \
  --private-dns-enabled
```

Useful private-subnet endpoint set:

| Workload need | Endpoint |
|---------------|----------|
| Pull ECR images | `ecr.api`, `ecr.dkr`, plus S3 gateway endpoint |
| Write logs | `logs` |
| Read secrets | `secretsmanager` or `ssm` |
| Use KMS-encrypted data/secrets | `kms` |
| Poll/send queues | `sqs` |
| Publish notifications | `sns` |
| Call STS regional endpoint | `sts` |

For ECS/Fargate image pulls, missing the **S3 gateway endpoint** is a common
mistake because ECR image layers are stored in S3.

---

## 5. NAT vs Endpoints

| Requirement | Better answer |
|-------------|---------------|
| Heavy S3/DynamoDB traffic from private subnets | Gateway endpoint |
| Private access to AWS APIs such as Secrets/KMS/SQS/ECR | Interface endpoint |
| Arbitrary third-party internet APIs | NAT Gateway |
| No internet path at all, only AWS services | Endpoints, no NAT required |

Endpoints reduce NAT cost and keep traffic on the AWS network, but they do not
grant permissions by themselves. IAM, resource policies, endpoint policies, SGs,
NACLs, and routes must all agree.

---

## 6. Multi-Account Endpoint Architecture

At organization scale, choose deliberately between **distributed endpoints** in each workload
VPC and **centralized interface endpoints** in a network-services VPC. Gateway and interface
endpoints behave differently here.

```text
Workload account A VPC ─┐
Workload account B VPC ─┼─ Transit Gateway ─ inspection route ─┐
Workload account C VPC ─┘                                      │
                                                               ▼
                                                Network-services account/VPC
                                                ├─ interface endpoints in AZ A/B/C
                                                ├─ endpoint security group/policies
                                                ├─ central private hosted zones
                                                └─ Resolver endpoints for hybrid DNS
```

An interface endpoint's private IPs are reachable over routed VPC peering or Transit Gateway, so
spokes can use a central set. **Gateway endpoints cannot be shared or reached through a Transit
Gateway**: their prefix-list route is installed in route tables in the endpoint's own VPC. Keep
S3/DynamoDB gateway endpoints distributed in each spoke, or deliberately use paid S3/DynamoDB
interface endpoints where the supported access/cost model justifies it.

### Central private DNS ownership

With private DNS enabled on a normal interface endpoint, AWS creates a service-managed private
hosted zone for that endpoint VPC. Spoke VPCs do not automatically inherit it over Transit
Gateway. A central pattern therefore uses:

1. interface endpoints in the network-services VPC, commonly with their managed private DNS
   disabled for the shared naming path;
2. a centrally owned Route 53 private hosted zone for the exact Regional service name, with alias
   or managed records that resolve to the endpoint's Regional/zonal DNS targets;
3. private hosted-zone association with each spoke VPC (using cross-account association/RAM or a
   governed Route 53 Profile pattern where applicable); and
4. Route 53 Resolver inbound endpoints and conditional forwarding for on-premises clients.

Spoke instances should keep using the normal SDK name such as
`secretsmanager.us-east-1.amazonaws.com`; no application-specific endpoint URL is then required.
Test resolution from every account/VPC and from on premises. Avoid creating both the endpoint's
managed private DNS zone and a custom zone for the same name in one VPC, which creates conflicting
namespace ownership. DNS administration belongs with the endpoint lifecycle: deleting/replacing
an endpoint without updating its records can direct the entire organization to dead IPs.

Check the service-specific DNS model before applying this pattern. **DynamoDB PrivateLink does
not support general private/hybrid DNS overrides** for its normal Regional name; applications use
the endpoint-specific URL, while ordinary in-VPC traffic can keep using a local no-charge gateway
endpoint. S3 has its own interface-endpoint/private-DNS options, including an inbound-Resolver-only
mode that lets on-premises traffic use the paid interface endpoint while VPC traffic keeps using
the gateway endpoint. Generic DNS overrides can break when a service changes its endpoints.

Centralization is **Regional**. Build a network-services endpoint/DNS stack in each Region and do
not route ordinary AWS API calls across Regions merely to save endpoint hourly charges; that adds
latency, inter-Region dependency, data transfer, and a much larger failure domain.

### Route and security path

Spoke subnet route tables send the central endpoint VPC/subnet prefixes through the Transit
Gateway. TGW and endpoint-VPC route tables must provide the return path. The endpoint SG allows TCP 443 from
approved spoke CIDRs (or supported cross-VPC SG references), not the internet. Workload SG/NACLs
must allow the outbound/return path.

If policy requires network inspection, use dedicated TGW route tables/appliance mode and
Network Firewall or an inspection VPC so forward and return traffic traverse the same stateful
firewall path. Confirm that DNS answers resolve to endpoint IPs reachable through that path.
PrivateLink does not automatically detour through a firewall, and a local S3 gateway endpoint
route bypasses a centralized egress/inspection VPC by design. Compensate with endpoint/resource
policy, S3 access logging/data events, and the control appropriate to the data path.

### Layer authorization instead of relying on “private”

For a call through a central endpoint, all relevant layers must allow it:

```text
caller IAM/SCP/boundary
  AND interface endpoint policy
  AND target resource policy (bucket, secret, KMS key, queue, ...)
  AND service-specific KMS authorization where encrypted
  AND SG/NACL/route/DNS path
```

An endpoint policy limits what can pass through that endpoint; it does not grant service access.
Central endpoint policies must cover many spoke roles/resources and have a document-size/blast-
radius trade-off. Prefer named organization roles/resources and deny unintended services/actions;
keep workload-specific least privilege in IAM and resource policies.

Be careful with a bucket-policy deny using `aws:sourceVpce`: it also denies console, replication,
AWS service, or recovery paths whose request context does not contain that endpoint unless the
policy has deliberate exceptions. In a centralized design every spoke shares one endpoint ID, so
that condition proves the network entry point, not which spoke was the caller. Keep principal and
organization conditions as the authorization boundary.

### Cost and blast-radius decision

| Factor | Distributed interface endpoints | Centralized interface endpoints |
|--------|---------------------------------|---------------------------------|
| Hourly endpoint ENI charge | Repeated per service/AZ/VPC | Shared per service/AZ in endpoint VPC |
| Endpoint data processing | Paid | Paid, plus TGW and possible cross-AZ/inspection processing |
| Network path | Local and simple | More routes, DNS, return-path, and inspection dependencies |
| Policy isolation | Per-workload endpoint policy/SG | Shared policy/SG has broader scope and size pressure |
| Failure domain | One VPC | Central endpoint/DNS/TGW can affect many accounts |
| Operations | Many repeated endpoints | Fewer endpoints, more platform ownership and change control |

Interface endpoint pricing includes an hourly charge for every endpoint AZ ENI and tiered data
processing. The cheapest diagram depends on traffic distribution: a central endpoint can remove
hundreds of hourly endpoint copies but add TGW processing and cross-AZ charges. Create an endpoint
in every AZ that sends material traffic or deliberately accept cross-AZ cost/failure exposure.
Gateway endpoints remain attractive for S3/DynamoDB because they have no endpoint hourly/data-
processing charge, though normal service and data-transfer charges still apply.

### Quotas, ownership, and failure tests

Track interface/gateway endpoints per VPC, ENIs/IP addresses per subnet, SG rules, private hosted
zone associations, Resolver endpoints/rules, TGW routes/attachments, and endpoint-policy size.
Request adjustable increases before an account/VPC factory reaches them; leave IP and quota
headroom for endpoint replacement during deployment.

The network platform team owns endpoint/DNS/TGW health and change rollback. Workload teams own
IAM and resource-policy correctness and declare their required services/Regions. Monitor endpoint
ENI health/bytes/errors, DNS query logs, Resolver health, TGW drops, firewall capacity, rejected
flow logs, and synthetic `GetCallerIdentity`/service calls from each representative spoke.

Test:

- loss of one endpoint AZ and one Resolver endpoint;
- endpoint SG/policy and private-zone rollback;
- TGW/inspection return-path failure;
- endpoint replacement without stale DNS;
- a denied principal/resource to prove least privilege; and
- Regional recovery using only that Region's endpoint/DNS stack.

If central DNS or routing fails, do not silently fall back to public/NAT access unless that is an
explicitly governed recovery path; otherwise a private-access control has become optional.

---

## 7. Troubleshooting

**DNS resolves to a private IP but connections time out**

- Interface endpoint SG likely does not allow inbound `443` from the workload SG.
- A custom NACL might be missing return traffic.

**SDK still goes to the public endpoint**

- Private DNS might be disabled.
- The VPC might not have DNS support/hostnames enabled.
- The SDK might be configured with a custom endpoint URL.

**S3 works through NAT but not after adding a gateway endpoint**

- Check that the private subnet route tables are associated with the endpoint.
- Check endpoint policy and bucket policy conditions such as `aws:sourceVpce`.
- If outbound SG rules are restrictive, allow HTTPS to the S3 prefix list.

**Fargate task cannot pull an image in a private subnet**

- Add `ecr.api`, `ecr.dkr`, CloudWatch Logs if used, and the S3 gateway endpoint.
- Ensure endpoint SG allows `443` from the task SG.

**Spoke resolves the public service IP even though a central endpoint exists**

- The endpoint-managed private zone applies only to its VPC. Associate the centrally owned
  private hosted zone/Profile with the spoke, check exact Regional service name and VPC DNS
  settings, and inspect Resolver query logs.

**Spoke resolves a central private IP but the request times out**

- Verify spoke→TGW→inspection→endpoint and return routes, endpoint SG source, NACL ephemeral
  return traffic, and stateful-firewall symmetry. A correct DNS answer proves no network path.

**Centralization costs more than distributed endpoints**

- Break the bill into endpoint AZ-hours/data, TGW processing, cross-AZ transfer, and inspection.
  Keep high-volume S3/DynamoDB on local gateway endpoints and deploy endpoint ENIs in the traffic's
  AZs; centralize only where the avoided endpoint copies exceed the added path cost/risk.

---

## Key Exam Points

- Gateway endpoint = S3/DynamoDB, route table, no ENI, no SG, free.
- Interface endpoint = PrivateLink ENI, SG, private DNS, endpoint policy, paid.
- Endpoint policies do not replace IAM or bucket/resource policies.
- S3 bucket policies can require `aws:sourceVpce` to force endpoint access.
- Private workloads that only call AWS services might not need NAT at all.
- Gateway endpoints stay local to a VPC. Interface endpoints can be centralized over TGW/peering,
  but DNS, routing, policy, cost, quotas, and failure ownership become platform dependencies.
- Central private DNS needs explicit hosted-zone/Resolver ownership; endpoint private DNS does not
  automatically propagate to spoke VPCs.
- Compare hourly endpoint copies against TGW/cross-AZ/inspection processing and the larger central
  blast radius. Test zonal, DNS, endpoint-policy, and return-route failures.

---

**Back to**: [Practical Examples](README.md)
