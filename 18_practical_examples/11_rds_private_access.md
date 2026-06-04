# RDS in Private Subnets, Accessed from the App Tier

> **Who this is for**: Engineers prepping for SAA-C03 who need to place an RDS
> database where only the application tier can reach it — never the public
> internet. Assumes you know VPC subnets and security groups, and have seen
> [RDS](../06_databases/02_rds.md).

The exam's favorite "secure database" answer: RDS with **Publicly Accessible =
No**, in a **DB subnet group spanning 2+ AZs**, with a security group that
allows the DB port **only from the app-tier security group** — not from a CIDR.

---

## 1. The Tiered Layout

```
                          VPC 10.0.0.0/16
 ┌──────────────────────────────────────────────────────────────────┐
 │                                                                    │
 │   PUBLIC subnets (have route to IGW)                               │
 │   ┌────────────────┐        ┌────────────────┐                     │
 │   │ ALB (AZ-a)     │        │ ALB (AZ-b)     │   ← internet-facing  │
 │   └───────┬────────┘        └───────┬────────┘                     │
 │           │  HTTP/HTTPS             │                              │
 │           ▼                         ▼                              │
 │   APP/PRIVATE subnets (no IGW route)                               │
 │   ┌────────────────┐        ┌────────────────┐                     │
 │   │ EC2 app (AZ-a) │        │ EC2 app (AZ-b) │  SG: app-sg          │
 │   └───────┬────────┘        └───────┬────────┘                     │
 │           │  3306 (MySQL) / 5432 (Postgres)                         │
 │           ▼                         ▼                              │
 │   DATA/PRIVATE subnets (no IGW route)   ── DB subnet group ──       │
 │   ┌────────────────┐        ┌────────────────┐                     │
 │   │ RDS primary    │◀──sync─▶│ RDS standby   │  SG: rds-sg          │
 │   │ (AZ-a)         │        │ (AZ-b)         │  Public Access = No  │
 │   └────────────────┘        └────────────────┘                     │
 │                                                                    │
 └──────────────────────────────────────────────────────────────────┘

  rds-sg inbound rule:  3306  FROM  app-sg   (source = security group, not CIDR)
```

> **Key insight**: Network reachability is gated by **two** things — the
> subnet's route table (no IGW route = no public path) **and** the security
> group. RDS `Publicly Accessible = No` means RDS gets **no public DNS / public
> IP**, so even a misconfigured route can't expose it.

---

## 2. The DB Subnet Group (must span ≥ 2 AZs)

A **DB subnet group** is the set of subnets RDS may launch into. Even a
single-AZ instance requires a subnet group with **at least two subnets in two
different AZs**, because Multi-AZ failover (and AZ rebalancing) needs somewhere
to place the standby.

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name app-db-subnets \
  --db-subnet-group-description "Private data-tier subnets, 2 AZs" \
  --subnet-ids subnet-0aaa1111 subnet-0bbb2222   # one in us-east-1a, one in us-east-1b
```

⚠️ Both subnets must be **private** (no route to an Internet Gateway). Putting a
data-tier subnet in a route table with an IGW route is a common audit failure.

---

## 3. Security Groups — Reference the App SG, Not a CIDR

```bash
# App-tier SG (already attached to the EC2 app instances)
APP_SG=sg-0app111

# Data-tier SG for RDS — allow Postgres ONLY from the app SG
aws ec2 create-security-group \
  --group-name rds-sg --description "RDS data tier" --vpc-id vpc-0123

RDS_SG=sg-0rds222

aws ec2 authorize-security-group-ingress \
  --group-id $RDS_SG \
  --protocol tcp --port 5432 \
  --source-group $APP_SG          # ← reference the SG, not 10.0.0.0/16
```

In Terraform:

```hcl
resource "aws_security_group_rule" "rds_from_app" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.rds.id
  source_security_group_id = aws_security_group.app.id   # SG reference
  description              = "Postgres from app tier only"
}
```

✅ Referencing the **source security group** means the rule auto-applies to any
instance that joins `app-sg` — no IP bookkeeping, and it survives Auto Scaling
replacing instances. A hard-coded CIDR (`10.0.0.0/16`) would allow the *entire*
VPC, including subnets that shouldn't talk to the DB.

---

## 4. Create the Private RDS Instance

```bash
aws rds create-db-instance \
  --db-instance-identifier app-prod-db \
  --engine postgres \
  --db-instance-class db.t3.medium \
  --allocated-storage 50 \
  --db-subnet-group-name app-db-subnets \
  --vpc-security-group-ids $RDS_SG \
  --no-publicly-accessible \              # ← the critical flag
  --multi-az \                            # standby in the second AZ
  --master-username appadmin \
  --manage-master-user-password           # store/rotate creds in Secrets Manager
```

The app connects via the **endpoint**, never an IP:

```
app-prod-db.cxy9abc123de.us-east-1.rds.amazonaws.com:5432
```

💡 The endpoint is a DNS name. With Multi-AZ, on failover AWS repoints this same
name to the promoted standby — your app's connection string never changes.

---

## 5. Credentials via Secrets Manager

`--manage-master-user-password` makes RDS create and **rotate** the master
credential in Secrets Manager automatically. The app reads it at runtime and
gets the rotation for free:

```bash
aws secretsmanager get-secret-value \
  --secret-id rds!db-1234abcd-... \
  --query SecretString --output text
```

✅ No plaintext DB password in environment variables, AMIs, or user-data. The
EC2 instance role just needs `secretsmanager:GetSecretValue` on that secret ARN.

> **Rule**: Production DB credentials live in **Secrets Manager** (rotation) or
> **SSM Parameter Store SecureString** (cheaper, manual rotation) — never in
> code, AMIs, or unencrypted config.

---

## 6. Comparison: Reachability Settings

| Setting / control            | Public access | Private (this pattern)        |
|------------------------------|---------------|-------------------------------|
| `Publicly Accessible`        | Yes           | **No**                        |
| Subnet route table           | IGW route     | **No IGW route**              |
| Gets public IP / public DNS  | Yes           | **No**                        |
| Inbound SG source            | 0.0.0.0/0 ❌  | **app-sg reference** ✅       |
| DB subnet group AZ count     | any           | **≥ 2 AZs**                   |
| Who can connect              | anyone on net | only app-tier instances       |

---

## 7. Troubleshooting

**App can't connect — connection times out**

- `rds-sg` inbound doesn't allow the app SG, or allows the wrong port (3306 vs
  5432). Confirm the rule references `app-sg` and the engine's port.
- Outbound on the **app** SG is restricted. Egress to the DB port must be
  allowed (default egress is all-allow, but locked-down SGs may block it).
- App instances are in a subnet/AZ that has no DB subnet in the same... actually
  cross-AZ traffic is fine within a VPC; check the SG first.

**App can't connect — name does not resolve**

- You set `Publicly Accessible = No` (correct) but are trying to resolve from
  *outside* the VPC. The private endpoint only resolves to a private IP from
  inside the VPC (enable DNS hostnames/resolution on the VPC).

**Created RDS, but it refused — "needs subnets in 2 AZs"**

- The DB subnet group had only one AZ. Add a second private subnet in a
  different AZ before creating the instance.

**Security review flags the DB as internet-exposed**

- `Publicly Accessible = Yes` was left on (the console default in some flows is
  Yes). Modify the instance to `--no-publicly-accessible`. Also check the SG
  isn't using `0.0.0.0/0`.

**Multi-AZ "failover" didn't scale my reads**

- Classic Multi-AZ DB instance standby is **not readable**. That's read replicas — see the next file.
  The newer Multi-AZ DB cluster variant has readable standbys, but the classic exam pattern is
  "Multi-AZ instance = HA, not read scaling."

---

## Key Exam Points

- Secure RDS = **Publicly Accessible: No** + **private subnets (no IGW route)** +
  **SG inbound from the app SG**.
- A **DB subnet group needs ≥ 2 AZs**, even for single-AZ instances.
- Reference the **source security group**, not a CIDR — survives ASG churn and
  scopes access tightly.
- Apps connect via the **endpoint DNS name**, which Multi-AZ repoints on failover.
- DB credentials ⇒ **Secrets Manager** (auto-rotation) or SSM SecureString.

---

**Next**: [12_rds_multiaz_vs_read_replica.md — Multi-AZ vs Read Replicas, Side by Side](12_rds_multiaz_vs_read_replica.md)
