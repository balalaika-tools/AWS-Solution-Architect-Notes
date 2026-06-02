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
penalty. The ENI is what gives the function a private IP inside your VPC.

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

## 5. When to Put Lambda in a VPC

✅ **Put it in a VPC** when it must reach **private-only** resources: RDS/Aurora
with no public access, ElastiCache, an internal ALB/NLB, or on-prem over a VPN/DX.

❌ **Keep it out of a VPC** when it only calls **public AWS APIs** (S3, DynamoDB,
SQS via public endpoints) or the internet — VPC attachment just adds config,
NAT cost, and an SG to manage with no benefit.

> **Rule of thumb**: VPC-attach a Lambda **only because it needs a private
> resource**, never "for security by default." Outside-VPC Lambdas are not
> internet-exposed inbound — there's no inbound listener.

---

## 6. Troubleshooting

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

**Reaching Secrets Manager hangs from VPC-Lambda**

- Secrets Manager is a *public* AWS service; from a private subnet with no NAT
  you need an **interface VPC endpoint** for `secretsmanager` (or a NAT).
- If the endpoint exists, check private DNS and the endpoint SG. A closed
  endpoint SG on 443 looks like a Lambda timeout, not an IAM error.

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

---

**Next**: [15_sqs_with_asg.md — Decoupled Workers: SQS-Driven Auto Scaling](15_sqs_with_asg.md)
