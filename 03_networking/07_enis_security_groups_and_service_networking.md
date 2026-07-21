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
| **NAT Gateway** | Zonal mode creates a requester-managed ENI in its subnet; regional mode manages interfaces across AZs at VPC scope | **No SG support**; use workload SGs, route tables, and NACLs | Neither NAT mode supports SGs. A NAT instance is different. |
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

## 5. Capacity Planning — IPs and ENIs Are Hard Limits

CPU can be available while a deployment fails because a subnet has no assignable
IP or an instance/account has reached an ENI quota. Plan both dimensions:

1. **Address capacity** — usable IPs in each subnet and AZ, including headroom
   for failover, rolling deployment, and service-managed replacement.
2. **ENI capacity** — Region-level network-interface quotas and per-instance ENI
   and IP limits, which vary by instance type.

Do the calculation per AZ. A healthy `/20` in AZ-a does not help a service trying
to create an ENI in an exhausted `/24` in AZ-b.

### How major services consume capacity

| Workload | Capacity behavior | What to reserve or verify |
|----------|-------------------|---------------------------|
| **VPC Lambda** | Lambda shares Hyperplane ENIs for functions using the same subnet + SG combination and creates more as connection/concurrency demand requires | Provide multiple healthy AZ subnets, avoid needless subnet/SG combinations, and include Lambda ENIs in the Regional ENI quota. Function count alone does not predict ENI count. |
| **EKS with VPC CNI** | Nodes consume primary ENIs; Pods consume secondary IPs/prefixes, and the CNI keeps a warm pool for fast scheduling | Model peak Pods plus warm capacity. Check the node type's ENI/IP limits. Prefix delegation increases pod density but needs contiguous `/28` blocks; use subnet CIDR reservations to prevent fragmentation. |
| **Interface endpoints** | Each service endpoint creates one ENI/IP in every selected AZ subnet | Multiply `endpoint services × AZs`. Twenty endpoints across three endpoint subnets consume at least 60 subnet IPs before workload growth. |
| **Load balancers** | Managed nodes use addresses in every enabled AZ and need spare addresses for replacement/scale-out | Give ALB subnets at least a `/27` and keep at least eight IPs free per subnet; do not fill load-balancer subnets with unrelated workloads. |
| **ECS/Fargate** | Every `awsvpc` task has a task ENI and private IP | Budget for desired count, deployment surge, and failed-task replacement in each AZ. |
| **RDS/Aurora and caches** | Instances/nodes use subnet-group addresses and may need temporary capacity during failover, maintenance, or replacement | Keep eligible subnets in multiple AZs with headroom beyond the steady-state node count. |
| **Other managed services** | EFS mount targets, Resolver endpoints, NAT, and inspection/endpoints place one or more managed interfaces in selected subnets | Inventory by ENI description/owner and reserve per-AZ capacity for the service's HA topology. |

Dedicated endpoint, load-balancer, EKS-pod, and database subnets make growth more
predictable and prevent one service from consuming another service's failover
capacity. They cost address space, so allocate them from the
[organization IP plan](02_vpc_subnets_route_tables.md#9-organization-scale-ip-address-management),
not as isolated `/28`s.

### Detect exhaustion before a scale event

Use a planned peak, not today's count:

```
required headroom per AZ =
  deployment surge + AZ-failure redistribution + managed replacement + endpoint growth
```

Then monitor the control-plane signals:

- `DescribeSubnets` / the VPC console exposes each subnet's available IPv4
  address count. Alert on the **absolute headroom required by the workload**, not
  only a generic percentage.
- VPC IPAM publishes `SubnetIPUsage` to CloudWatch. Alarm early enough to add a
  secondary CIDR and new subnets before an incident.
- For EKS, use VPC CNI metrics such as assigned Pod IPs, maximum ENIs, and
  available IPs; compare actual CNI warm-pool settings with the scaling plan.
- Check Service Quotas for network interfaces per Region and the instance-type
  ENI/IP limits. Request quota increases before a launch or disaster-recovery
  test, not during it.
- Run an Auto Scaling, Lambda concurrency, EKS node/pod, or deployment-surge test
  at expected peak. A dashboard cannot reveal an incorrect AZ or subnet choice
  that the service uses only during failover.

### Diagnose an ENI or IP allocation failure

When scaling stalls, do not assume compute capacity is the cause:

1. Read the service event and CloudTrail failures for `CreateNetworkInterface`,
   `AssignPrivateIpAddresses`, `InsufficientFreeAddressesInSubnet`, quota errors,
   or EKS `InsufficientCidrBlocks`.
2. Compare available addresses **in every configured subnet/AZ**. Then list ENIs
   by subnet and inspect their descriptions and requesters to find which service
   owns the address consumption.
3. Compare Regional ENI quota use and per-node ENI/IP limits. A subnet can have
   free addresses while an EKS node or the account cannot attach another ENI.
4. For EKS prefix delegation, check for a free contiguous `/28`; scattered free
   IPs still fail prefix allocation. Use a subnet CIDR reservation or a new
   subnet rather than treating the total free count as sufficient.
5. Recover safely: remove genuinely unused endpoints/resources, add a secondary
   VPC CIDR and new subnets, or move growth to prepared subnets. Do not delete a
   requester-managed ENI directly; change or remove the service that owns it.

You cannot resize an existing subnet CIDR. Emergency subnet creation is much
harder when the VPC has no unallocated contiguous block, which is why VPC-level
growth reservations and per-service subnet budgets belong in the initial design.

---

## 6. Key Exam Points

- **ENI = network card in one subnet/AZ.** It consumes subnet IP space.
- **SG = stateful firewall on an ENI.** NACL = stateless subnet boundary.
- **EC2, `awsvpc` ECS tasks, VPC Lambda, RDS/Aurora, ElastiCache, EFS mount
  targets, interface endpoints, Resolver endpoints, and many load balancers**
  all have ENI/security group implications.
- **Gateway endpoints** for S3/DynamoDB are route-table constructs: no ENI, no
  SG, free.
- **Interface endpoints** are PrivateLink ENIs: SG + private DNS + endpoint
  policy, paid.
- A **zonal NAT Gateway** has a requester-managed ENI; regional mode manages
  networking at VPC scope. Neither has an SG. A NAT Instance is EC2 and **does**
  have an SG.
- **NLB can have SGs now**, but only if SGs were associated when the NLB was
  created. **GWLB cannot have SGs.**
- Watch **IP exhaustion** in small private subnets. `/28` might be fine for a
  toy subnet but is easy to exhaust with endpoints, Lambda, Fargate, databases,
  load balancers, and mount targets.
- Plan IP and ENI capacity per AZ for normal scale, rolling deployment, service
  replacement, and AZ-failure redistribution. Monitor IPAM/CloudWatch and service
  quotas before the event.
- EKS can exhaust node ENI/IP slots or subnet addresses; prefix delegation needs
  contiguous `/28` space. An ALB subnet should be at least `/27` with eight free
  addresses for scale/replacement.

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
- Checking only total VPC free space instead of the exact AZ subnet a service
  must use, or only subnet free IPs while the Regional/per-instance ENI quota is full.
- Counting steady-state resources but not rolling-deployment surge, managed
  replacement, or AZ-failure headroom.
- Assuming every ENI has an editable SG. Requester-managed ENIs may be visible
  but controlled by the owning AWS service.

---

**Next**: [08_dhcp_prefix_lists_sharing_analyzer.md — DHCP Option Sets, Prefix Lists, VPC Sharing & Reachability Analyzer](08_dhcp_prefix_lists_sharing_analyzer.md)
