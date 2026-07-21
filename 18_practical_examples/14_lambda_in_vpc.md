# Lambda Inside vs Outside a VPC

> **Who this is for**: Engineers prepping for SAA-C03 who need a Lambda function
> to reach a **private RDS** database, and discover that attaching it to a VPC
> suddenly breaks its internet access. Assumes you've read
> [Lambda](../10_serverless/01_lambda.md).

> **Key insight**: By default, Lambda runs **outside your VPC** — it has internet
> access but **cannot** reach resources in your private subnets. Attach it to a
> VPC and it can reach private RDS, **but it loses the default internet route**
> unless you add a **NAT gateway** (for the internet) or **VPC endpoints** (for
> AWS services).

---

## 1. The Two Modes

```
DEFAULT — Lambda OUTSIDE your VPC
─────────────────────────────────────────────
  ┌──────────┐   internet (AWS-managed)   ┌─────────────┐
  │  Lambda  │ ─────────────────────────▶ │ Public APIs │  ✅ reaches internet
  └──────────┘                            │ S3, DDB,... │  ✅ reaches AWS svc
        │                                 └─────────────┘
        │  ✗ cannot reach
        ▼
  ┌─────────────────────────────┐
  │ Private RDS in YOUR VPC      │  ❌ no route into your private subnets
  └─────────────────────────────┘


VPC-ATTACHED — Lambda gets an ENI in YOUR subnets
─────────────────────────────────────────────
  ┌──────────┐  Hyperplane ENI in
  │  Lambda  │  your private subnet        ┌─────────────────────────────┐
  └────┬─────┘ ──────────────────────────▶ │ Private RDS (same VPC)      │ ✅
       │                                   └─────────────────────────────┘
       │  internet?  ── needs NAT GW ──────────▶ Internet   (else ✗ timeout)
       │  S3 / DynamoDB? ── gateway VPC endpoint ─▶ (no NAT data charges)
       │  Secrets/KMS/SQS? ── interface VPC endpoint ─▶
       ▼
```

⚠️ A VPC-attached Lambda in a **private subnet with no NAT gateway** has **no
internet access**. Calls to public endpoints hang until the function times out.

---

## 2. The ENI / Hyperplane Detail

When you attach a function to a VPC, Lambda creates **Hyperplane ENIs** in the
subnets/SGs you specify. These are **shared and pre-provisioned** across all
invocations of functions sharing the same subnet+SG combination — which is why
modern VPC Lambda **no longer** suffers the old multi-second cold-start ENI
creation on every scale-out. The first use of a new subnet+security-group
combination can still leave the function `Pending` while Lambda creates the ENI.

One Hyperplane ENI supports up to **65,000 connections/ports**; Lambda creates
more ENIs as network traffic and concurrency require. Concurrency therefore
does **not** consume one subnet IP per invocation, but ENIs still need free IPv4
addresses. Interface VPC endpoints, RDS, load balancers, ECS tasks, and other
resources consume addresses from the same subnet too. Avoid unnecessary unique
subnet+SG combinations, monitor `AvailableIpAddressCount`, the ENI quota, and
connection pressure, and leave deployment/failure headroom rather than sizing a
subnet to today's steady state.

> **Rule**: Give the function **at least two private subnets in two AZs** for
> resilience, and a security group whose **egress** allows the destinations it
> needs (e.g. 5432 to RDS). The RDS SG must allow inbound **from the Lambda SG**.

---

## 3. Configure a VPC-Attached Lambda (to reach private RDS)

```bash
aws lambda update-function-configuration \
  --function-name order-processor \
  --vpc-config '{
      "SubnetIds": ["subnet-priv-a","subnet-priv-b"],
      "SecurityGroupIds": ["sg-lambda"]
  }'
```

The execution role needs the EC2 networking permissions to manage ENIs (the
managed policy **`AWSLambdaVPCAccessExecutionRole`** grants them):

```hcl
resource "aws_iam_role_policy_attachment" "vpc_access" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}
```

RDS SG inbound rule referencing the Lambda SG (same pattern as the app tier):

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds --protocol tcp --port 5432 \
  --source-group sg-lambda
```

---

## 4. Restoring Internet / AWS-Service Access

Once in a VPC, pick the cheapest route that satisfies the destination:

| Destination from VPC-Lambda      | Solution                                  |
|----------------------------------|-------------------------------------------|
| Public internet / 3rd-party API  | **NAT gateway** in a public subnet + route|
| **S3** or **DynamoDB**           | **Gateway VPC endpoint** (free, no NAT)   |
| Secrets Manager, KMS, SQS, SNS, ECR, CW Logs | **Interface VPC endpoint** (PrivateLink) |
| Private RDS in same VPC          | nothing extra (the ENI reaches it)        |

**Gateway endpoint for S3** (no NAT data-processing charges):

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0123 \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-priv-a rtb-priv-b   # adds the S3 prefix-list route
```

💡 If the Lambda only needs S3/DynamoDB (no general internet), **gateway
endpoints alone** are enough — you can skip the NAT gateway entirely and avoid
its hourly + per-GB cost.

**Interface endpoint SG rule** (example for Secrets Manager/KMS/SQS/etc):

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-vpce \
  --protocol tcp --port 443 \
  --source-group sg-lambda
```

Enable **private DNS** on the interface endpoint when the service supports it so
normal SDK hostnames resolve to the endpoint's private IPs. If private DNS is
off, configure the SDK/CLI with the endpoint-specific `vpce-...` DNS name.

Endpoint policies can further restrict what the function may do through the
endpoint, but they do not replace the Lambda execution role or resource policies
such as S3 bucket policies.

---

## 5. Choose NAT or Endpoints from Traffic and Failure Requirements

| Pattern | Cost/operations | Best fit | Availability trap |
|---------|-----------------|----------|-------------------|
| S3/DynamoDB gateway endpoints | No endpoint hourly/data-processing charge | High-volume access to those services | Associate every Lambda subnet route table and allow endpoint/bucket policy |
| Interface endpoints | Hourly per endpoint/AZ plus data processing | Private access to a small, known set of supported AWS APIs | One endpoint ENI/AZ or restrictive endpoint SG/private DNS can create zonal timeouts |
| NAT gateway | Hourly per NAT plus data processing | Many AWS/public APIs and third-party internet destinations | A single NAT/AZ is a shared failure point and can add cross-AZ data-transfer cost |

Several interface endpoints in every AZ can cost more than NAT for a low-volume
function, while high-volume S3 through NAT is needlessly expensive when a free
gateway endpoint fits. Inventory destinations and bytes, include endpoint/NAT
hourly and data charges, then choose. A resilient NAT design normally has one
NAT gateway per AZ and same-AZ routes; a centralized NAT is a deliberate
cost-versus-blast-radius tradeoff.

Endpoint policies and bucket/key/queue policies should restrict the function's
resources and organization, but they do not replace the execution-role policy.
Create endpoint ENIs in each AZ used by the function and alarm on rejected
connections/timeouts; “private” does not mean “highly available by default.”

---

## 6. Control Concurrency Before It Reaches the VPC

Estimate steady concurrency as roughly:

```
concurrency = requests per second × average duration in seconds
```

Then include burst, retry, and failure-mode headroom. The limiting resource is
often RDS connections, NAT ports, an interface endpoint, or an on-premises API,
not Lambda itself.

- **Reserved concurrency** reserves part of the Regional account pool for this
  function and sets its maximum concurrency. It does not pre-initialize
  execution environments and has no additional charge. Use the ceiling to
  protect a database or partner API, but understand that upstream events will
  queue, retry, or fail according to their source.
- **Provisioned concurrency** pre-initializes environments for a published
  version or alias to reduce initialization latency and has an additional
  charge. It does not replace reserved concurrency, control downstream load, or
  guarantee that traffic above the provisioned amount will not use on-demand
  environments.
- Use RDS Proxy or a bounded connection pool for database access. Do not open a
  new unbounded DB connection per record and assume ENI capacity is the only
  limit.

### Event-source back-pressure

For SQS, set the event source mapping's **maximum concurrency** no higher than
the function/downstream budget, and keep the sum of mappings within reserved
concurrency. Set queue visibility timeout to at least six times the function
timeout (plus the batch window), enable partial batch responses so successful
records are not retried, and configure a DLQ/redrive count. Watch queue age and
depth, not only Lambda errors.

For Kinesis/DynamoDB Streams, ordering and shard throughput constrain scaling.
Configure bounded retry attempts/record age, partial batch failure reporting or
batch bisection where appropriate, and an on-failure destination. A poison
record must not block a shard forever. Event source mappings are **at least
once**, so duplicate delivery remains possible even after tuning.

---

## 7. Cross-Account Private Resources

Cross-account access needs both a network path and authorization:

1. Connect non-overlapping VPCs with peering/Transit Gateway, expose a service
   intentionally with PrivateLink, or use another approved private connectivity
   pattern. Configure both route tables, DNS resolution, NACLs, and the resource
   SG. A private IP in another account is not reachable merely because IAM
   allows it.
2. Grant the function execution role the resource action. The destination
   resource policy or an assumable role must trust it, and a customer-managed
   KMS key policy must allow decrypt/use as well. Network access alone does not
   authorize S3, SQS, Secrets Manager, or database operations.
3. Scope trust to specific role/resource ARNs and organization/account
   conditions. Log role assumption and resource access in the owning account.

Lambda can consume an SQS queue in another account in the same Region with a
cross-account event source mapping and the appropriate queue/KMS/execution-role
policies. That is different from attaching the function to the queue owner's
VPC; SQS authorization is an API control-plane/data-plane policy problem, not a
VPC routing problem.

---

## 8. Timeouts, Retries, Idempotency, and Tracing

Set the function timeout from a measured end-to-end budget and make SDK connect,
read, and total timeouts **shorter** than it. Otherwise a dead NAT path or
single-AZ database consumes the whole invocation before your code can fall back
or record useful context.

Retry semantics depend on the invoker:

- A synchronous invocation returns the function error to the caller; Lambda
  does not automatically retry the function-code error for that caller.
- An asynchronous invocation retries a function error twice by default and can
  send exhausted events to an on-failure destination/DLQ.
- SQS, streams, Kafka, and other event source mappings have their own polling,
  checkpoint, visibility, age, and retry controls.

Because a timeout can occur after a side effect committed, use an idempotency
key and a conditional write/transaction record before charging, ordering, or
publishing. Make the idempotency record retention at least as long as the
source's retry/redrive window.

Use structured CloudWatch logs and metrics for cold starts, duration, timeout,
throttles, concurrent executions, ENI/IP symptoms, downstream latency, and
event age. Enable **X-Ray active tracing** (and the execution-role write
permissions) or the organization's OpenTelemetry path so a trace separates
Lambda initialization, DNS/connect, NAT/endpoint, and database time. Alarm on
throttles, errors, duration close to timeout, iterator/queue age, DLQ depth, and
provisioned-concurrency spillover where used.

### Zonal failure drill

Select private subnets in at least two AZs, but also remove single-AZ
dependencies from the path: use Multi-AZ RDS/proxy, endpoint ENIs per AZ, and
same-AZ NAT gateways/routes where internet egress is required. During a game
day, make one test-AZ egress/downstream path unavailable and verify new
invocations continue through another path, retries remain within the downstream
budget, queue age recovers, and no duplicate side effects occur. Measure from
the first error until backlog and concurrency return to normal—not only until a
single invocation succeeds.

---

## 9. When to Put Lambda in a VPC

✅ **Put it in a VPC** when it must reach **private-only** resources: RDS/Aurora
with no public access, ElastiCache, an internal ALB/NLB, or on-prem over a VPN/DX.

❌ **Keep it out of a VPC** when it only calls **public AWS APIs** (S3, DynamoDB,
SQS via public endpoints) or the internet — VPC attachment just adds config,
NAT cost, and an SG to manage with no benefit.

> **Rule of thumb**: VPC-attach a Lambda **only because it needs a private
> resource**, never "for security by default." Outside-VPC Lambdas are not
> internet-exposed inbound — there's no inbound listener.

---

## 10. Troubleshooting

**Lambda times out calling a public API after I attached it to a VPC**

- Classic symptom. The function is in private subnets with no internet route.
  Add a **NAT gateway** in a public subnet and a `0.0.0.0/0 → nat-...` route in
  the Lambda subnets' route tables. (Or, if it only needs S3/DynamoDB, add a
  **gateway endpoint** instead.)

**Lambda timed out — I put it in the PUBLIC subnet to get internet**

- Lambda ENIs in a public subnet still get **no public IP**, so they can't use
  the IGW. Public subnet ≠ internet for Lambda. Use **private subnets + NAT**.

**Lambda can't reach S3 even with a NAT gateway working**

- It works via NAT, but you're paying NAT data charges. Add an **S3 gateway VPC
  endpoint** and ensure its route is in the function's subnet route tables. If S3
  access broke, a restrictive **bucket policy** may require the endpoint ID.

**Lambda can't connect to RDS — connection timeout**

- RDS SG doesn't allow inbound from the **Lambda SG**, or the Lambda is in a
  different VPC. Reference `sg-lambda` on the RDS SG inbound for 5432/3306.

**Function deploy/update fails: cannot create network interface**

- Execution role lacks ENI permissions. Attach
  `AWSLambdaVPCAccessExecutionRole`. Also check you haven't hit the **ENI / IP
  limit** in the chosen subnets — give them adequate free IP space.

**Function scales, but RDS starts rejecting connections**

- Lambda concurrency grew faster than the DB connection budget. Cap reserved
  and event-source concurrency, reuse connections, and use RDS Proxy where it
  fits. More subnet IPs or ENIs will not fix an exhausted database.

**Reaching Secrets Manager hangs from VPC-Lambda**

- Secrets Manager is a *public* AWS service; from a private subnet with no NAT
  you need an **interface VPC endpoint** for `secretsmanager` (or a NAT).
- If the endpoint exists, check private DNS and the endpoint SG. A closed
  endpoint SG on 443 looks like a Lambda timeout, not an IAM error.

**SQS backlog grows while the function shows throttles**

- Reserved/account concurrency or downstream capacity is below the mapping's
  demand. Align mapping maximum concurrency with the function reservation,
  check visibility timeout and DLQ policy, and alarm on oldest-message age.

**Only invocations using one AZ's path time out**

- Look for a missing interface endpoint ENI, route to a NAT in a failed AZ, or a
  zonal downstream target. Compare subnet route tables, endpoint SGs, NACLs, and
  flow logs for every configured AZ.

---

## Key Exam Points

- **Default Lambda = outside VPC**: has internet, **cannot** reach private RDS.
- **VPC-attached Lambda** reaches private resources but **loses default internet**
  — needs **NAT gateway** (internet) or **VPC endpoints** (AWS services).
- S3 / DynamoDB ⇒ **gateway endpoint** (free); Secrets/KMS/SQS/etc ⇒ **interface
  endpoint**; private RDS ⇒ ENI is enough.
- Use **private subnets** for VPC-Lambda; a public subnet does **not** give it
  internet (no public IP).
- Attach the **`AWSLambdaVPCAccessExecutionRole`** for ENI management; give ≥ 2
  AZs of subnets.
- Only VPC-attach when the function **needs a private resource** — not by default.
- Hyperplane ENIs are shared and support many connections, but subnet IP/ENI,
  endpoint, NAT-port, and downstream connection capacity still need monitoring.
- Reserved concurrency protects capacity and sets a ceiling; provisioned
  concurrency reduces cold starts. Neither replaces event-source back-pressure.
- Retry behavior is source-specific and delivery can be duplicated: set
  deadlines, DLQs/destinations, partial-batch handling, and idempotency.
- Multi-AZ subnets are only useful when endpoints, NAT, DNS, and downstream
  services also survive a zonal failure—and the complete path is tested.

---

**Next**: [15_sqs_with_asg.md — Decoupled Workers: SQS-Driven Auto Scaling](15_sqs_with_asg.md)
