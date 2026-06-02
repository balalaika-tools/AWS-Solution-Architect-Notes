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
  outbound: all (or service-appropriate)
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

## 6. Troubleshooting

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

---

## Key Exam Points

- Gateway endpoint = S3/DynamoDB, route table, no ENI, no SG, free.
- Interface endpoint = PrivateLink ENI, SG, private DNS, endpoint policy, paid.
- Endpoint policies do not replace IAM or bucket/resource policies.
- S3 bucket policies can require `aws:sourceVpce` to force endpoint access.
- Private workloads that only call AWS services might not need NAT at all.

---

**Back to**: [Practical Examples](README.md)
