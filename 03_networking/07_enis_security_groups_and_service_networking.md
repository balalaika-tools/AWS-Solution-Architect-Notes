# ENIs, Security Groups & Service Networking

> **Who this is for**: Engineers who understand VPCs, subnets, route tables, and
> security groups, but want one clear map of which AWS resources actually get
> network interfaces, which resources need security groups, and which VPC
> endpoint type to choose on the exam.

---

## 1. The Core Mental Model

An **Elastic Network Interface (ENI)** is a virtual network card in a VPC subnet.
Because a subnet is in one Availability Zone, an ENI is also **AZ-scoped**.

An ENI can have:

- one primary private IP, plus optional secondary private IPs,
- optional IPv6 addresses,
- a MAC address,
- an optional public IPv4 or Elastic IP association,
- one or more **security groups**.

> **Rule**: A security group attaches to an **ENI**, not to a route table, subnet,
> or "the internet." When people say "the EC2 instance has this security group,"
> they really mean "the instance's ENI has this security group."

Important ENI facts:

- Every EC2 instance has a **primary ENI** (`eth0`). You cannot detach it.
- Secondary ENIs can be attached/detached and moved to another instance in the
  **same AZ**.
- Some AWS services create **requester-managed ENIs** for you. You can see them,
  but you cannot manage them like normal ENIs.
- ENIs consume IP addresses from the subnet. IP exhaustion is a real scaling
  problem for Lambda-in-VPC, ECS/Fargate, EKS, interface endpoints, NAT
  gateways, load balancers, EFS mount targets, and databases.
- EC2 ENIs have a **source/destination check**. Disable it only for appliances
  that forward traffic, such as NAT instances or firewall/router instances.

---

## 2. Which Resources Have ENIs and Security Groups

| Resource | ENI behavior | Security group behavior | Exam trap |
|----------|--------------|-------------------------|-----------|
| **EC2 instance** | Requires a primary ENI in one subnet/AZ; can have secondary ENIs depending on instance type | Requires at least one SG; default SG is used if you do not choose one | SG is per ENI. A secondary ENI can have different SGs than `eth0`. |
| **NAT Gateway** | Creates a requester-managed ENI in the subnet | **No SG support**; use instance SGs and subnet NACLs | NAT GW has an ENI but still no SG. NAT instance is different. |
| **NAT Instance** | Normal EC2 ENI | SG applies like any EC2 instance | Must disable source/destination check. |
| **Application Load Balancer** | Creates managed ENIs/nodes in enabled subnets | Has SGs; target SG should allow from the ALB SG | Do not open private instances to `0.0.0.0/0`; allow from ALB SG. |
| **Network Load Balancer** | Has per-AZ addresses/ENIs; can use static IP/EIP per AZ | Can have SGs if associated at creation; if created without SGs, you cannot add them later | Modern NLBs can use SGs. Older notes often say they cannot. |
| **Gateway Load Balancer** | Uses managed load balancer networking; traffic is routed through GWLBe | You cannot associate an SG with a GWLB | SGs belong on appliance targets, not the GWLB itself. |
| **Gateway Load Balancer endpoint** | Creates an endpoint ENI in one subnet/AZ and is used as a route target | No normal interface-endpoint SG selection | It is a VPC endpoint, but it is not the same as an interface endpoint. |
| **Interface VPC Endpoint** | Creates a requester-managed endpoint ENI in each selected subnet | Has an SG; usually allow inbound 443 from clients in the VPC | If the endpoint SG blocks 443, private DNS can resolve correctly and traffic still fails. |
| **Gateway VPC Endpoint** | **No ENI**; route-table target for AWS prefix list | **No SG** on the endpoint; SGs on clients may allow egress to prefix list | Only S3 and DynamoDB. Free. Not reachable from on-prem by itself. |
| **Lambda attached to your VPC** | Lambda creates Hyperplane ENIs for subnet + SG combinations | You choose SGs in the VPC config; downstream SGs should allow from the Lambda SG | VPC Lambda ENIs do not get public IPs, even in a public subnet. Use NAT or endpoints. |
| **ECS task in `awsvpc` mode** | Each task gets its own task ENI; Fargate requires this | Task ENI gets the SGs from the service/task network config | Fargate tasks require subnets and SGs. Watch subnet IP/ENI limits. |
| **ECS task in `bridge`/`host` mode** | Uses the EC2 container instance networking | SG is on the EC2 host ENI, not per task | Per-task SGs require `awsvpc`. |
| **EKS on EC2** | VPC CNI allocates pod IPs from node ENIs and may attach secondary ENIs | Pods usually share node SGs; SG for Pods can use branch ENIs | Pod density is limited by IPs/ENIs as well as CPU/memory. |
| **RDS DB instance** | AWS assigns a network interface from the DB subnet group | DB has VPC SGs; allow DB port from app SG | Connect to endpoint DNS, not the ENI IP. |
| **Aurora cluster** | DB instances use IPs from the DB subnet group | Cluster/instances use VPC SGs | Use writer/reader endpoints; do not hard-code instance IPs. |
| **ElastiCache node/replication group** | Nodes get IPs from a cache subnet group | Use VPC SGs; allow Valkey/Redis OSS 6379 or Memcached 11211 from app SG | It is private VPC networking; do not expose cache ports broadly. |
| **EFS mount target** | Each mount target is an ENI in a subnet/AZ | Mount target SG must allow NFS 2049 from client SGs | One mount target per AZ where clients run. |
| **Route 53 Resolver endpoint** | Inbound/outbound endpoints create ENIs in subnets | SG must allow UDP/TCP 53 from the resolver clients/targets | Hybrid DNS fails silently when port 53 is blocked. |
| **Private API Gateway REST API** | Clients reach it through an `execute-api` interface VPC endpoint | Endpoint ENI SG controls who can connect; API resource policy can also restrict source VPC/VPC endpoint | Private API != private subnet. Private API endpoints are REST API only; HTTP APIs can use private integrations to reach VPC backends, but the API endpoint itself is Regional/public. |

> **Useful shortcut**: if the resource is something that receives or sends
> packets from inside your VPC, look for its ENI and SG story. Then remember the
> exceptions: route-table gateways and gateway endpoints generally do **not** use
> SGs, even when a managed ENI exists behind the scenes.

---

## 3. VPC Endpoint Decision Table

| Requirement | Choose | ENI? | SG? | Notes |
|-------------|--------|------|-----|-------|
| Private access to **S3 or DynamoDB** from resources in the same VPC | **Gateway endpoint** | No | No | Free; adds prefix-list routes to selected route tables; supports endpoint policies. |
| Private access to most AWS APIs/services (Secrets Manager, KMS, SQS, SNS, ECR, CloudWatch Logs, STS, EC2 API, etc.) | **Interface endpoint** | Yes | Yes | Powered by PrivateLink; use private DNS when supported; paid hourly + data. |
| Private access to an API Gateway private REST API | **Interface endpoint for `execute-api`** | Yes | Yes | Pair with API resource policy for tighter access. |
| Expose one private service from another VPC/account/SaaS provider | **PrivateLink endpoint service + interface endpoint** | Yes on consumer side | Yes on consumer endpoint | Provider normally fronts service with NLB. No full VPC routing required. |
| Route traffic through third-party inspection appliances | **GWLB + Gateway Load Balancer endpoint** | Yes for GWLBe | Not like an interface endpoint | Route tables steer traffic to the GWLBe. |
| Access S3/DynamoDB from on-prem over VPN/DX | Usually **interface endpoint** | Yes | Yes | Gateway endpoints are not directly reachable from on-prem networks. |

Endpoint policies are only one authorization layer. A request must also pass IAM
identity policies, resource policies such as S3 bucket policies, security groups
where applicable, NACLs, and route tables.

Common endpoint policy pattern:

- Put a gateway endpoint on private subnet route tables for S3.
- Add an S3 bucket policy condition such as `aws:sourceVpce` so the bucket can
  only be accessed through that endpoint.
- Keep private workloads off the NAT path for S3/DynamoDB to avoid NAT data
  processing charges.

---

## 4. Security Group Design Patterns

### Web to app to database

```
internet
  -> ALB SG: allow 443 from 0.0.0.0/0
  -> app SG: allow app port from ALB SG
  -> DB SG:  allow 5432/3306 from app SG
```

Use SG references rather than CIDRs when both sides are in VPC security groups.
The rule follows Auto Scaling, replacement instances, and task rescheduling.

### Private workload to AWS service

```
private EC2/Lambda/ECS
  -> gateway endpoint for S3/DynamoDB
  -> interface endpoint SG allows 443 from workload SG for other AWS services
```

If a private subnet workload only needs AWS services, VPC endpoints can avoid a
NAT Gateway entirely. If it needs arbitrary public internet or third-party APIs,
it still needs NAT or another egress design.

### Hybrid DNS

```
on-prem DNS -> Route 53 Resolver inbound endpoint ENIs
VPC resolver -> outbound endpoint ENIs -> on-prem DNS
```

Resolver endpoint SGs need UDP and TCP 53. DNS uses TCP for large responses and
zone-transfer-like behavior, so allowing only UDP is fragile.

---

## 5. Key Exam Points

- **ENI = network card in one subnet/AZ.** It consumes subnet IP space.
- **SG = stateful firewall on an ENI.** NACL = stateless subnet boundary.
- **EC2, `awsvpc` ECS tasks, VPC Lambda, RDS/Aurora, ElastiCache, EFS mount
  targets, interface endpoints, Resolver endpoints, and many load balancers**
  all have ENI/security group implications.
- **Gateway endpoints** for S3/DynamoDB are route-table constructs: no ENI, no
  SG, free.
- **Interface endpoints** are PrivateLink ENIs: SG + private DNS + endpoint
  policy, paid.
- **NAT Gateway** has a requester-managed ENI but **no SG**. NAT Instance is EC2
  and **does** have an SG.
- **NLB can have SGs now**, but only if SGs were associated when the NLB was
  created. **GWLB cannot have SGs.**
- Watch **IP exhaustion** in small private subnets. `/28` might be fine for a
  toy subnet but is easy to exhaust with endpoints, Lambda, Fargate, databases,
  load balancers, and mount targets.

---

## Common Real-World Misconfigurations

- Creating interface endpoints but leaving the endpoint SG closed on 443.
- Routing private subnet S3/DynamoDB traffic through NAT instead of a free
  gateway endpoint.
- Putting Lambda ENIs in a public subnet and expecting internet access; Lambda
  does not receive a public IP from that subnet.
- Opening RDS or ElastiCache to CIDRs instead of allowing only the app SG.
- Creating tiny private subnets and then running out of IPs because every
  endpoint, task, function, database, and mount target needs addresses.
- Assuming every ENI has an editable SG. Requester-managed ENIs may be visible
  but controlled by the owning AWS service.

---

**Next**: [01_hybrid_networking.md - VPN and Direct Connect](../14_hybrid_migration_dr/01_hybrid_networking.md)
