# ECS Fargate Service Behind an ALB

> **Who this is for**: Engineers prepping for SAA-C03 who need to run a
> containerized web service on **Fargate** (no EC2 to manage), front it with an
> **ALB**, keep tasks in **private subnets**, and auto-scale on load. Assumes
> you've seen [ECS/ECR/Fargate/EKS](../09_containers/02_ecs_ecr_fargate_eks.md)
> and ALBs in [ELB/ALB/NLB/GWLB](../07_ha_scaling/02_elb_alb_nlb_gwlb.md).

> **Key insight**: Fargate tasks use **awsvpc** networking — every task gets its
> **own ENI and private IP**. So unlike EC2 bridge mode, there's **no dynamic
> host port mapping**; the target group targets the task's IP on the **container
> port** directly.

---

## 1. Architecture

```
        Internet
           │ HTTPS:443
      AWS WAF web ACL
           │
           ▼
   ┌───────────────────────┐  PUBLIC subnets (2 AZs)
   │  ALB (internet-facing)│  SG: alb-sg  (in 443 from 0.0.0.0/0)
   └───────────┬───────────┘
               │ forward → target group (IP targets)
               │ health check GET /health
               ▼
   ┌───────────────────────┐  PRIVATE subnets (2 AZs)  ── awsvpc ──
   │ Fargate task (AZ-a)   │  ENI + private IP, container :8080
   │ Fargate task (AZ-b)   │  SG: task-sg  (in 8080 FROM alb-sg only)
   │ Fargate task (AZ-a)   │  ← Service Auto Scaling adds/removes tasks
   └───────────┬───────────┘
               │ outbound (pull image, call APIs)
               ▼
        NAT GW  /  VPC endpoints (ECR, S3, CloudWatch Logs)
```

> **SG chaining**: `alb-sg` allows 443 from the internet; `task-sg` allows the
> container port **from `alb-sg`** (a security-group reference), nothing else.
> The ALB is the only thing that can reach the tasks.

For private tasks with no NAT Gateway, create interface endpoints for
`ecr.api`, `ecr.dkr`, and `logs`, plus an S3 gateway endpoint for ECR image
layers. If tasks read secrets, add `secretsmanager` or `ssm`; if they use
customer-managed keys, add `kms`. The endpoint SG must allow HTTPS (`443`) from
`task-sg`, and private DNS should be enabled where supported.

Create interface endpoint ENIs in every AZ used by the service and allow their
SGs from `task-sg`. Restrict endpoint and ECR repository policies to the required
accounts/repositories; an endpoint is a private route, not an authorization
grant. Compare the per-AZ hourly and data-processing cost of several interface
endpoints with NAT gateways for the service's actual traffic—“no NAT” is a
security/connectivity choice, not automatically the cheapest design.

---

## 2. Task Definition (Fargate)

```json
{
  "family": "web-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::111122223333:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::111122223333:role/webAppTaskRole",
  "containerDefinitions": [{
    "name": "web",
    "image": "111122223333.dkr.ecr.us-east-1.amazonaws.com/web-app:1.4.2",
    "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
    "secrets": [{
      "name": "DATABASE_URL",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:111122223333:secret:web/prod/db-AbCdEf"
    }],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/web-app",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "web"
      }
    }
  }]
}
```

💡 With `awsvpc` + Fargate you only declare a **containerPort** — there's no
`hostPort` to map. Each task's ENI exposes 8080 directly to the target group.
Keep the roles separate:

- The **execution role** is used by the ECS/Fargate agent to pull ECR images,
  send `awslogs`, and fetch secrets referenced by the task definition. Add
  `secretsmanager:GetSecretValue` and `kms:Decrypt` only for the referenced
  secrets/keys; the standard managed execution policy does not grant arbitrary
  secret access.
- The **task role** supplies credentials to application code. Grant its exact
  S3/DynamoDB/SQS/etc. actions and resources; it does not need ECR pull or log
  driver permissions. Cross-account resources also need the destination
  resource/KMS policy or an explicitly trusted role.

Injected secret values are resolved when the task starts. Rotation does not
update an already running container, so force a new deployment after rotation
or have the application retrieve the current secret at runtime through its task
role.

---

## 3. Target Group + ALB Listener

```bash
# Target group must be type "ip" for Fargate (not "instance")
aws elbv2 create-target-group \
  --name web-app-tg \
  --protocol HTTP --port 8080 \
  --target-type ip \
  --vpc-id vpc-0123 \
  --health-check-path /health \
  --health-check-protocol HTTP \
  --healthy-threshold-count 2 \
  --health-check-interval-seconds 15

# HTTPS listener forwarding to the target group
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/web-alb/abc \
  --protocol HTTPS --port 443 \
  --certificates CertificateArn=arn:aws:acm:us-east-1:111122223333:certificate/xyz \
  --default-actions Type=forward,TargetGroupArn=arn:aws:...:targetgroup/web-app-tg/def
```

⚠️ Choosing **`--target-type instance`** for a Fargate service is a classic
mistake — Fargate registers task **IPs**, so the target group must be **`ip`**.

---

## 4. ECS Service (wires tasks to the ALB)

```bash
aws ecs create-service \
  --cluster prod \
  --service-name web-app \
  --task-definition web-app:7 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration '{
      "awsvpcConfiguration": {
        "subnets": ["subnet-priv-a","subnet-priv-b"],
        "securityGroups": ["sg-task"],
        "assignPublicIp": "DISABLED"
      }}' \
  --load-balancers '[{
      "targetGroupArn":"arn:aws:...:targetgroup/web-app-tg/def",
      "containerName":"web",
      "containerPort":8080
  }]' \
  --health-check-grace-period-seconds 60 \
  --deployment-configuration '{
      "minimumHealthyPercent":100,
      "maximumPercent":200,
      "deploymentCircuitBreaker":{"enable":true,"rollback":true}
  }'
```

The service registers/deregisters task IPs with the target group automatically as
tasks start, stop, or get replaced during a deployment.

---

## 5. Service Auto Scaling

Register the service as a scalable target and attach a **target-tracking** policy:

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/prod/web-app \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 --max-capacity 20

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/prod/web-app \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-target-50 \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
      "TargetValue": 50.0,
      "PredefinedMetricSpecification": {
        "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
      },
      "ScaleInCooldown": 60,
      "ScaleOutCooldown": 60
  }'
```

✅ You can also target **`ALBRequestCountPerTarget`** to scale on requests per
task — often a better signal than CPU for a request-driven web service.

Set minimum capacity high enough to survive one task/AZ loss and maximum
capacity high enough for the expected peak, then load-test the target value and
cooldowns. CPU may stay low while memory, connections, queue depth, or request
latency saturate, so use the metric tied to the bottleneck. Alarm when the
service remains at maximum capacity or has pending tasks; auto scaling cannot
fix a subnet with no IPs or unavailable Fargate capacity.

---

## 6. TLS, WAF, and Safe Deployments

Terminate viewer TLS on the ALB with an ACM certificate in the **same Region**
as the ALB, redirect port 80 to 443, and create a Route 53 alias to the ALB.
Attach a Regional AWS WAF web ACL and roll managed rules from count to block
after reviewing false positives. Restrict `alb-sg` to the intended client ranges
where the service is not public.

The `/health` endpoint should prove the task can serve traffic but remain fast
and avoid making one optional downstream dependency take every target out of
service. Set the health-check grace period from measured startup time, and make
the container handle `SIGTERM` by failing readiness, draining work, and exiting
within `stopTimeout`/the ALB deregistration delay.

The service example enables the rolling deployment **circuit breaker with
rollback**. Also attach CloudWatch deployment alarms for application failures
that target health alone cannot see (for example, 5xx rate or latency). A
deployment needs a previously completed revision to roll back to; keep image
tags immutable or deploy by digest so “rollback” really runs the prior image.
Validate `minimumHealthyPercent`/`maximumPercent` against quota and subnet IP
headroom so the scheduler can start a replacement before stopping the old task.

For higher-risk releases, use ECS blue/green deployment with separate target
groups, test traffic, a bake period, and explicit rollback criteria. It costs
more during overlap but separates code validation from the production traffic
shift.

---

## 7. Fargate Capacity Providers and Spot

Use a capacity-provider strategy instead of `--launch-type` when mixing
on-demand and Spot capacity. This example keeps the first two tasks on Fargate
and distributes additional tasks approximately 1:3 across Fargate and Fargate
Spot:

```bash
aws ecs update-service \
  --cluster prod \
  --service web-app \
  --capacity-provider-strategy \
    capacityProvider=FARGATE,base=2,weight=1 \
    capacityProvider=FARGATE_SPOT,weight=3 \
  --force-new-deployment
```

Fargate Spot is for interruption-tolerant capacity. It sends an EventBridge task
state-change event and `SIGTERM` about two minutes before interruption; set
`stopTimeout` (up to the supported 120 seconds), stop accepting new work, and
finish/checkpoint safely. Spot can be unavailable and ECS does not silently
replace missing Spot capacity with on-demand capacity. Keep the availability
floor on `FARGATE`, use Spot for the elastic portion, and alarm on placement
failures and Spot interruptions.

---

## 8. Observability and Multi-AZ Failure Handling

- Set CloudWatch Logs retention and use structured logs with request/trace IDs.
  Enable Container Insights with enhanced observability for service/task
  resource and lifecycle visibility; use ADOT/X-Ray (or your standard tracer)
  for calls through the ALB and downstream services.
- Enable ALB access logs to a protected S3 bucket. Alarm on unhealthy/healthy
  host counts, ALB 5xx and target 5xx separately, response time, rejected
  connections, ECS running-versus-desired count, CPU/memory saturation,
  deployment failure, and task stopped reasons.
- Run at least two tasks and pass private subnets from at least two AZs. Verify
  tasks are actually distributed and that each AZ has enough spare subnet IPs
  for scaling plus a rolling deployment. The ALB must also have healthy targets
  in multiple AZs.
- Avoid a hidden zonal egress dependency. Put interface endpoint ENIs in each
  service AZ; if using NAT gateways, use same-AZ routes to a NAT per AZ for a
  higher-availability design. One shared NAT makes every task's startup or API
  access depend on its AZ.
- In a game day, stop the tasks in one AZ and make one egress path unavailable.
  Measure ALB error rate, task replacement, endpoint/NAT behavior, and time to
  restore the availability floor. Then test a bad image and confirm the circuit
  breaker or alarm restores the last good revision.

---

## 9. Comparison: Fargate vs EC2 Launch Type (behind ALB)

| Aspect                       | Fargate (awsvpc)            | EC2 launch type (bridge)        |
|------------------------------|-----------------------------|---------------------------------|
| Networking per task          | Own ENI + IP                | Shares host ENI                 |
| Target group type            | **`ip`**                    | `instance` (+ dynamic ports)    |
| Host/dynamic port mapping    | N/A (container port direct) | Dynamic host ports required     |
| Manage EC2 hosts/patching    | ❌ none                     | ⚠️ you manage the cluster EC2   |
| Scaling unit                 | Task count                  | Task count **and** ASG capacity |
| Cost model                   | Per vCPU/GB-second          | Per EC2 instance-hour           |

---

## 10. Troubleshooting

**Targets stuck "unhealthy" / 502 from the ALB**

- Health check path wrong: `/health` returns non-2xx, or the app actually listens
  on a different path/port. Match `--health-check-path` and `containerPort` to the app.
- `task-sg` doesn't allow the container port **from `alb-sg`** — health checks
  originate from the ALB's ENIs. Add inbound `8080 FROM alb-sg`.
- Health check **grace period** too short — the app hadn't finished booting before
  the first probe failed and the task was killed. Raise `--health-check-grace-period-seconds`.

**Tasks fail to start: `CannotPullContainerError`**

- Tasks are in private subnets with **no NAT GW** and **no VPC endpoints**, so
  they can't reach ECR/S3 to pull the image. Add a NAT gateway **or** ECR
  (`ecr.api`, `ecr.dkr`), **S3 (gateway)**, and **CloudWatch Logs** VPC endpoints.
  If endpoints already exist, check private DNS and that the endpoint SG allows
  `443` from `task-sg`.

**Tasks can't pull image even with public subnet**

- `assignPublicIp` is `DISABLED` in a public subnet → no route out. For Fargate
  in a public subnet you must `ENABLED`; the recommended design is **private
  subnets + NAT/VPC endpoints** instead.

**Target group registration fails: target type mismatch**

- Target group is `instance` but Fargate registers IPs. Recreate the target group
  with `--target-type ip`.

**Deployment never stabilizes / tasks cycle**

- New tasks fail health checks and get replaced repeatedly (bad image, crash on
  start, wrong port). Check CloudWatch Logs `/ecs/web-app` and ECS task
  "stopped reason."
- Confirm the deployment has subnet IP and service quota headroom for old and
  new tasks at the same time. Check the circuit-breaker/CloudWatch alarm event
  and that a last completed deployment exists for rollback.

**Tasks start in one AZ but fail in another**

- That AZ is missing an endpoint ENI/NAT route, or the endpoint SG/NACL differs.
  Test ECR, S3 image layers, Logs, Secrets, KMS, and application dependencies
  from every task subnet.

**Fargate Spot interruption drops requests**

- The task did not handle `SIGTERM`, deregistration/stop timeouts are
  mismatched, or the service put its minimum availability on Spot. Keep an
  on-demand base and drain within the two-minute notice.

---

## Key Exam Points

- Fargate uses **awsvpc**: each task gets its **own ENI/IP** → target group must
  be **`ip`**; **no dynamic host ports** (that's EC2 bridge mode).
- **SG chaining**: ALB SG open to internet on 443; **task SG allows container
  port only from the ALB SG**.
- Keep tasks in **private subnets**; give them egress via **NAT GW or VPC
  endpoints** (ECR.api, ECR.dkr, S3 gateway, Logs, plus Secrets/KMS as needed)
  to pull images and call AWS services.
- Scale the **service** with Application Auto Scaling target tracking — CPU or
  **ALBRequestCountPerTarget**.
- Fargate = no EC2 to patch; EC2 launch type = you also manage the cluster's ASG.
- Use distinct **execution** and **task** roles, deploy by immutable image,
  enable circuit-breaker/alarm rollback, and protect TLS traffic with WAF.
- Multi-AZ availability includes task placement **and** zonally resilient image,
  logging, secret, and egress paths; prove it with failure drills.
- Keep an on-demand Fargate base for availability; Fargate Spot can disappear
  and provides only a short termination warning.

---

**Next**: [14_lambda_in_vpc.md — Lambda Inside vs Outside a VPC](14_lambda_in_vpc.md)
