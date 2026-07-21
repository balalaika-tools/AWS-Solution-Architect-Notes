# Public ALB Routing to EC2 in Private Subnets

> **Who this is for**: Engineers who have the [reference VPC](01_vpc_public_private_subnets.md)
> with [private-subnet egress via NAT](02_ec2_private_subnet_nat.md) and now need to accept
> *inbound* internet traffic safely. We put an internet-facing Application Load Balancer in
> the public subnets and forward to app instances that stay private. Concept reference:
> [HA & Scaling — load balancers](../07_ha_scaling/README.md).

---

## 1. The Pattern

NAT (file 02) gave us outbound. For **inbound** internet traffic we never expose the instances directly. Instead:

- An **internet-facing ALB** sits in the **public** subnets (one per AZ). It's the only thing with a public-facing presence.
- The ALB forwards to a **target group** of EC2 instances in the **private** subnets.
- Clients hit the ALB's DNS name; the ALB terminates TLS and proxies to instances over the private network.

The instances have no public IP and accept traffic **only from the ALB**, enforced by security-group chaining (section 4).

> **Key insight**: The ALB is the public front door; the instances are private rooms behind it. Inbound exposure is concentrated on one managed, patchable, scalable layer instead of every instance. This is *the* reference web-tier pattern on AWS.

---

## 2. ASCII Topology

```
                          Internet (clients)
                                │  HTTPS :443
                                ▼
                         ┌──────────────┐
                         │  ALB (DNS)   │  internet-facing
                         └──────┬───────┘
              ┌─────────────────┴─────────────────┐
              │ ALB has a node in EACH public subnet│
┌──────────── VPC 10.0.0.0/16 ───────────────────────────────────┐
│  ── AZ a ──                       ── AZ b ──                    │
│  ┌────────────────────┐           ┌────────────────────┐      │
│  │ public-az-a         │           │ public-az-b         │      │
│  │  [ALB node]         │           │  [ALB node]         │      │
│  └─────────┬───────────┘           └─────────┬───────────┘      │
│            │ :8080 (target port, private)     │                 │
│  ┌─────────▼───────────┐           ┌─────────▼───────────┐      │
│  │ private-az-a         │           │ private-az-b         │     │
│  │  EC2  10.0.10.25     │           │  EC2  10.0.11.30     │     │
│  │  (no public IP)      │           │  (no public IP)      │     │
│  └─────────────────────┘           └─────────────────────┘      │
│         Target Group: both instances, health-checked            │
└──────────────────────────────────────────────────────────────────┘
```

⚠️ An ALB **requires subnets in at least 2 AZs** — you must give it ≥2 public subnets in different AZs at creation, even if your app currently runs in one. This is enforced for availability.

---

## 3. The Pieces: Listener, Target Group, Health Check

The ALB has three configurable layers:

1. **Listener** — what the ALB listens on. Here: HTTPS `:443`, with an ACM certificate for TLS termination. (Add an HTTP `:80` listener that *redirects* to `:443`.)
2. **Target group** — the pool of instances and the port the ALB forwards to (e.g. `:8080`). Type `instance` or `ip`.
3. **Health check** — the ALB only sends traffic to **healthy** targets. E.g. `GET /healthz` every 30s, healthy after 3 passes, unhealthy after 2 fails.

```hcl
resource "aws_lb" "web" {
  name               = "web-alb"
  internal           = false                       # internet-facing
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = [aws_subnet.public_a.id,     # >= 2 AZs required
                        aws_subnet.public_b.id]
}

resource "aws_lb_target_group" "app" {
  name        = "app-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "instance"
  health_check {
    path                = "/healthz"
    interval            = 30
    healthy_threshold   = 3
    unhealthy_threshold = 2
    matcher             = "200"
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.web.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = aws_acm_certificate.web.arn   # TLS terminates here
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

resource "aws_lb_target_group_attachment" "a" {
  target_group_arn = aws_lb_target_group.app.arn
  target_id        = aws_instance.app_a.id
  port             = 8080
}
resource "aws_lb_target_group_attachment" "b" {
  target_group_arn = aws_lb_target_group.app.arn
  target_id        = aws_instance.app_b.id
  port             = 8080
}
```

💡 In production the targets are usually an **Auto Scaling Group** attached to the target group, not fixed instances — see [16_cloudwatch_alarm_scaling.md](16_cloudwatch_alarm_scaling.md).

---

## 4. Security-Group Chaining (the heart of this pattern)

This is what keeps the instances private even though the ALB is public. **Two** security groups that reference each other:

**ALB security group** — open to the internet on 443 only:

| Direction | Protocol | Port | Source / Dest | Why |
|-----------|----------|------|---------------|-----|
| Inbound | TCP | 443 | `0.0.0.0/0` | Public clients reach the ALB over HTTPS |
| Inbound | TCP | 80  | `0.0.0.0/0` | HTTP, redirected to 443 |
| Outbound | TCP | 8080 | **instance SG** | ALB forwards to app port |

**Instance security group** — accepts traffic **only from the ALB's SG**:

| Direction | Protocol | Port | Source / Dest | Why |
|-----------|----------|------|---------------|-----|
| Inbound | TCP | 8080 | **ALB SG (`sg-alb`)** | Only the ALB may reach the app port |
| Outbound | All | All | `0.0.0.0/0` | Egress (via NAT, file 02) |

> **Rule**: The instance SG's inbound source is the **ALB's security-group ID**, not a CIDR. This is *SG referencing* — it follows the ALB wherever its nodes are, scales automatically, and is far safer than guessing the ALB's IP range.

```hcl
resource "aws_security_group" "alb" {
  name   = "alb-sg"
  vpc_id = aws_vpc.main.id
  ingress { from_port = 443 to_port = 443 protocol = "tcp" cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 80  to_port = 80  protocol = "tcp" cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0   to_port = 0   protocol = "-1"  cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_security_group" "instance" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]   # <-- reference, not CIDR
  }
  egress { from_port = 0 to_port = 0 protocol = "-1" cidr_blocks = ["0.0.0.0/0"] }
}
```

❌ **Anti-pattern**: setting the instance SG inbound to `0.0.0.0/0:8080`. That defeats the entire design — the app is now reachable by anyone who learns a private IP path, the SG no longer documents intent, and you've widened the attack surface for nothing.

---

## 5. Request Flow

```
1. Client resolves web-alb-1234.us-east-1.elb.amazonaws.com → ALB public IPs
2. Client → ALB :443  (allowed: ALB SG inbound 443 from 0.0.0.0/0)
3. ALB terminates TLS, picks a healthy target
4. ALB → EC2 :8080    (allowed: instance SG inbound 8080 from ALB SG)
5. EC2 responds → ALB → client   (SGs are stateful; return needs no extra rule)
```

Because security groups are **stateful**, you do *not* add outbound rules for return traffic — the response to an allowed inbound request is automatically permitted (contrast with NACLs in [file 04](04_sg_vs_nacl_examples.md)).

---

## 6. Production Build, Deployment & Recovery

The fixed two-instance target group is useful for learning. Production replaces it with immutable
instances in an Auto Scaling group, protects the HTTPS entry point, records every layer of the
request, and proves that a deployment or AZ can fail without taking the service down.

### 6.1 Harden the front door

1. Issue or import an **ACM certificate in the ALB's Region** and validate every hostname clients
   use. Attach it to the `:443` listener with the current organization-approved TLS policy.
2. Make `:80` redirect to HTTPS; don't forward plaintext traffic to the application.
3. Associate an **AWS WAF web ACL**. Start managed rule groups in count mode, inspect false
   positives, then block; add a rate-based rule and explicit application exceptions. Deliver WAF
   logs to the central security destination.
4. Enable ALB access logs to a protected S3 bucket in the same Region. The bucket can be in a
   logging account but needs the precise ELB log-delivery bucket policy. ALB access-log buckets
   currently support SSE-S3 encryption, not SSE-KMS.
5. Turn on deletion protection and tag the ALB, target group, WAF ACL, certificate, log bucket,
   launch template, and Auto Scaling group for ownership and cost.

```hcl
resource "aws_lb" "web" {
  # ...the attributes from §3...
  enable_deletion_protection = true

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "orders"
    enabled = true
  }
}

resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.web.arn
  port              = 80
  protocol          = "HTTP"
  default_action {
    type = "redirect"
    redirect { protocol = "HTTPS"; port = "443"; status_code = "HTTP_301" }
  }
}

resource "aws_wafv2_web_acl_association" "web" {
  resource_arn = aws_lb.web.arn
  web_acl_arn  = aws_wafv2_web_acl.web.arn
}
```

### 6.2 Replace fixed instances with an Auto Scaling group

Use a versioned launch template and spread the ASG across three private app subnets. Attach the
target group through `target_group_arns`, set `health_check_type = "ELB"`, and choose a health-check
grace period from measured boot + application warm-up time. Keep minimum capacity high enough for
the surviving AZs to handle the peak, not merely enough to keep one target green.

Scale on a service signal such as ALB request count per target or a measured CPU/latency target.
Alarm on `UnHealthyHostCount`, target 5xx, ALB 5xx, p95/p99 target response time, rejected
connections, ASG launch failures, and capacity/quota shortfall. The health endpoint should test
whether the process can serve traffic, but it should not fail every target because one optional
downstream dependency is slow.

### 6.3 Drain connections before replacement

The target group's **deregistration delay** stops new requests to a draining target and gives
in-flight requests time to complete (default 300 seconds). Set it from the longest legitimate
request or streaming behavior. On termination, the app should stop accepting new work, finish or
checkpoint active work, flush telemetry, and exit before the ASG lifecycle timeout. A short delay
drops requests; an unnecessarily long delay slows scale-in and rollback.

Test keep-alive, WebSocket, file upload, and long-poll behavior explicitly. ALB draining doesn't
make a non-idempotent request safe to repeat after a client disconnect.

### 6.4 Deploy with an evidence-based rollback

Publish a new AMI and launch-template version; never patch the live fleet in place. Use an ASG
instance refresh with a minimum healthy percentage, bake/checkpoint periods, and CloudWatch alarm
rollback. A safe rollout is:

1. Launch one new instance, wait for ALB health, then run a synthetic HTTPS transaction through
   the public hostname.
2. Compare its target latency, 4xx/5xx, app errors, CPU/memory, and dependency failures with the old
   version during a bake period.
3. Continue in batches only while alarms remain healthy. Preserve the previous launch-template
   version and AMI.
4. On an alarm, use instance-refresh automatic rollback or start manual rollback while the refresh
   is still in progress. After a completed refresh, rollback is a new refresh to the prior version.

Database/schema changes must remain backward compatible with both versions during the rollout;
rolling the EC2 fleet back cannot undo an incompatible data migration.

### 6.5 Cross-zone and zonal failure behavior

ALB cross-zone load balancing is always enabled at the load-balancer level. Target groups inherit
that setting by default, although a target group can explicitly disable cross-zone routing. With it
enabled, an ALB node can use healthy targets in all enabled AZs; this smooths uneven target counts
but can send traffic across AZs. With it disabled, every enabled AZ needs enough healthy local
targets—an empty enabled subnet can cause 503s.

For an AWS or application impairment in one AZ, ALB DNS and health processing can remove unusable
zonal capacity. For deliberate evacuation, enable Route 53 ARC zonal shift on the ALB and test it:
the shifted zonal ALB IPs are removed from DNS and, with cross-zone enabled, traffic to targets in
that AZ is blocked. Before shifting, confirm the other AZs have N-1 capacity, their subnets have IP
headroom, and downstream databases/caches aren't still tied to the impaired zone. Existing client
connections can outlive the DNS change.

### 6.6 Observable success criteria

| Test | Evidence required |
|------|-------------------|
| TLS and redirect | HTTP returns redirect; HTTPS hostname validates; weak protocol policy is rejected as designed |
| WAF | Test rule appears in WAF logs/metrics; known-good transaction passes; sampled bad request is counted or blocked |
| Logs | ELB test object and real access-log objects arrive in S3; request ID correlates with app log/trace |
| Scaling | Load test reaches the target metric, launches healthy capacity in multiple AZs, and scales in only after draining |
| Deployment | One-instance canary and later batches meet latency/error SLO; injected alarm triggers rollback to the old launch template |
| Zonal failure | Zonal shift or controlled target removal preserves the measured success rate and latency on N-1 capacity |

Success is not merely `curl` returning 200 once. Record detection time, dropped requests, time to
healthy replacement capacity, rollback duration, and the exact SLO impact in the runbook.

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| ALB won't create | Only one subnet/AZ provided | Provide ≥2 public subnets in different AZs |
| Targets show **unhealthy** | Instance SG doesn't allow the health-check port from the ALB SG, or `/healthz` returns non-200 | Allow the target port from the ALB SG; verify the health-check path and matcher |
| 502 / 504 from ALB | App not listening on the target port, or instance SG blocks the ALB | Confirm app binds `:8080`; confirm SG reference is correct |
| Clients can't connect at all | ALB SG missing inbound 443, or ALB is `internal` not `internet-facing` | Add 443 ingress; set scheme to internet-facing (needs public subnets) |
| Works intermittently | Only one healthy target across AZs; cross-zone settings | Ensure healthy targets in each AZ; ALB cross-zone LB is on by default |
| Instance reachable from internet directly | Instance has a public IP or SG open to `0.0.0.0/0` | Remove public IP; restrict instance SG source to the ALB SG |
| HTTPS certificate error | Hostname isn't on the certificate, validation incomplete, or cert is in another Region | Use an ACM certificate in the ALB Region with every client hostname |
| WAF blocks valid requests | Managed rule enabled directly in block mode or exception too broad/narrow | Inspect WAF logs, tune a scoped exception, validate in count mode, then block |
| Access logs never arrive | S3 Region, encryption, or bucket policy is wrong | Use a same-Region SSE-S3 bucket and verify the ELB log-delivery test object |
| Refresh stalls or rolls back | New target never becomes healthy or a rollback alarm fired | Inspect target reason codes, app boot logs, ASG activity, and the named CloudWatch alarm before retrying |

💡 "Targets unhealthy" is the #1 ALB issue. Check, in order: target SG allows the **health-check port from the ALB SG**, the app actually listens on that port, and the path returns the expected status code.

---

## Key Exam Points

- The reference web tier: **internet-facing ALB in public subnets → instances in private subnets**, with instances having no public IP.
- An ALB **requires subnets in at least 2 AZs**.
- **SG chaining**: the instance SG's inbound *source is the ALB's security-group ID*, not a CIDR. Traffic comes only from the ALB.
- Security groups are **stateful** — return traffic is automatic; no outbound rule needed for replies.
- The ALB has three layers: **listener** (what it accepts), **target group** (where it forwards + health checks), and **health checks** (only healthy targets get traffic).
- ALB **terminates TLS** using an **ACM certificate**; redirect `:80 → :443`.
- Unhealthy targets almost always trace to an **SG blocking the health-check port** or a failing health-check path.
- Production adds WAF, S3 access logs, an ASG with ELB health checks, measured deregistration delay,
  alarm-protected instance refresh, and an N-1 zonal-failure test.
- ALB cross-zone routing is on at the load-balancer level; target groups can override it. DNS
  changes and zonal shifts don't terminate existing client connections.

---

**Next**: [04_sg_vs_nacl_examples.md — Security Groups vs NACLs, Worked Examples](04_sg_vs_nacl_examples.md)
