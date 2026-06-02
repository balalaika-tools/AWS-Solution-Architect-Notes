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

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| ALB won't create | Only one subnet/AZ provided | Provide ≥2 public subnets in different AZs |
| Targets show **unhealthy** | Instance SG doesn't allow the health-check port from the ALB SG, or `/healthz` returns non-200 | Allow the target port from the ALB SG; verify the health-check path and matcher |
| 502 / 504 from ALB | App not listening on the target port, or instance SG blocks the ALB | Confirm app binds `:8080`; confirm SG reference is correct |
| Clients can't connect at all | ALB SG missing inbound 443, or ALB is `internal` not `internet-facing` | Add 443 ingress; set scheme to internet-facing (needs public subnets) |
| Works intermittently | Only one healthy target across AZs; cross-zone settings | Ensure healthy targets in each AZ; ALB cross-zone LB is on by default |
| Instance reachable from internet directly | Instance has a public IP or SG open to `0.0.0.0/0` | Remove public IP; restrict instance SG source to the ALB SG |

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

---

**Next**: [04_sg_vs_nacl_examples.md — Security Groups vs NACLs, Worked Examples](04_sg_vs_nacl_examples.md)
