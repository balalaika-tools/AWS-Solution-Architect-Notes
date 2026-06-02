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
  "containerDefinitions": [{
    "name": "web",
    "image": "111122223333.dkr.ecr.us-east-1.amazonaws.com/web-app:1.4.2",
    "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
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
The **execution role** lets Fargate pull from ECR and push logs; an optional
**task role** grants the app's own AWS permissions (S3, DynamoDB, etc.).

---

## 3. Target Group + ALB Listener

```bash
# Target group must be type "ip" for Fargate (not "instance")
aws elbv2 create-target-group \
  --name web-app-tg \
  --protocol HTTP --port 8080 \
  --target-type ip \                 # ← IP targets, required for awsvpc/Fargate
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
  --health-check-grace-period-seconds 60
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

---

## 6. Comparison: Fargate vs EC2 Launch Type (behind ALB)

| Aspect                       | Fargate (awsvpc)            | EC2 launch type (bridge)        |
|------------------------------|-----------------------------|---------------------------------|
| Networking per task          | Own ENI + IP                | Shares host ENI                 |
| Target group type            | **`ip`**                    | `instance` (+ dynamic ports)    |
| Host/dynamic port mapping    | N/A (container port direct) | Dynamic host ports required     |
| Manage EC2 hosts/patching    | ❌ none                     | ⚠️ you manage the cluster EC2   |
| Scaling unit                 | Task count                  | Task count **and** ASG capacity |
| Cost model                   | Per vCPU/GB-second          | Per EC2 instance-hour           |

---

## 7. Troubleshooting

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

---

**Next**: [14_lambda_in_vpc.md — Lambda Inside vs Outside a VPC](14_lambda_in_vpc.md)
