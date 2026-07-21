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
 ┌────────────────────────────────────────────────────────────────────┐
 │                                                                    │
 │   PUBLIC subnets (have route to IGW)                               │
 │   ┌────────────────┐        ┌────────────────┐                     │
 │   │ ALB (AZ-a)     │        │ ALB (AZ-b)     │   ← internet-facing │
 │   └───────┬────────┘        └───────┬────────┘                     │
 │           │  HTTP/HTTPS             │                              │
 │           ▼                         ▼                              │
 │   APP/PRIVATE subnets (no IGW route)                               │
 │   ┌────────────────┐        ┌────────────────┐                     │
 │   │ EC2 app (AZ-a) │        │ EC2 app (AZ-b) │  SG: app-sg         │
 │   └───────┬────────┘        └───────┬────────┘                     │
 │           │  3306 (MySQL) / 5432 (Postgres)                        │
 │           ▼                         ▼                              │
 │   DATA/PRIVATE subnets (no IGW route)   ── DB subnet group ──      │
 │   ┌────────────────┐        ┌────────────────┐                     │
 │   │ RDS primary    │◀─sync─▶│ RDS standby    │  SG: rds-sg         │
 │   │ (AZ-a)         │        │ (AZ-b)         │  Public Access = No │
 │   └────────────────┘        └────────────────┘                     │
 │                                                                    │
 └────────────────────────────────────────────────────────────────────┘

  rds-sg inbound rule:  3306  FROM  app-sg   (source = security group, not CIDR)
```

> **Key insight**: Network reachability is gated by **two** things — the
> subnet's route table (no IGW route = no public path) **and** the security
> group. RDS `Publicly Accessible = No` means its DNS endpoint resolves to a
> **private IP**, with no public IP path. The DNS name still exists; private
> reachability and the security group determine whether a client can connect.

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
# Use encrypted storage, retained backups, deletion protection, and a separate
# maintenance window in addition to private networking.
aws rds create-db-instance \
  --db-instance-identifier app-prod-db \
  --engine postgres \
  --db-instance-class db.t3.medium \
  --allocated-storage 50 \
  --db-subnet-group-name app-db-subnets \
  --vpc-security-group-ids $RDS_SG \
  --no-publicly-accessible \
  --multi-az \
  --storage-encrypted \
  --kms-key-id alias/rds-prod \
  --backup-retention-period 35 \
  --preferred-backup-window 03:00-03:30 \
  --preferred-maintenance-window sun:04:00-sun:04:30 \
  --copy-tags-to-snapshot \
  --deletion-protection \
  --master-username appadmin \
  --manage-master-user-password
```

The app connects via the **endpoint**, never an IP:

```
app-prod-db.cxy9abc123de.us-east-1.rds.amazonaws.com:5432
```

💡 The endpoint is a DNS name. With Multi-AZ, on failover AWS repoints this same
name to the promoted standby — your app's connection string never changes.

---

## 5. Credentials via Secrets Manager

`--manage-master-user-password` makes RDS create and manage the master
credential in Secrets Manager. RDS rotates it every seven days by default; you
can change that schedule or rotate it immediately through RDS. RDS keeps the
database password and secret version synchronized:

```bash
aws secretsmanager get-secret-value \
  --secret-id rds!db-1234abcd-... \
  --query SecretString --output text
```

✅ No plaintext DB password in environment variables, AMIs, or user-data. The
EC2 instance role just needs `secretsmanager:GetSecretValue` on that secret ARN
and `kms:Decrypt` when the secret uses a customer-managed KMS key. Configure and
test the rotation schedule; applications must fetch a new value for new
connections instead of caching one password forever.

> **Rule**: Production DB credentials live in **Secrets Manager** (rotation) or
> **SSM Parameter Store SecureString** (cheaper, manual rotation) — never in
> code, AMIs, or unencrypted config.

---

## 6. Encrypt Storage and Validate TLS

`--storage-encrypted` protects the DB storage, logs, automated backups,
snapshots, and replicas with KMS. Treat the KMS key as part of the database
recovery path: restrict key administrators separately from DB administrators,
monitor disable/deletion requests, and make sure a cross-account or
cross-Region backup copy can use its destination key.

Private IPs do not remove the need for encryption in transit. Download the
current RDS CA bundle into the application's trust store and enable both
encryption **and server identity validation**. PostgreSQL example:

```bash
psql "host=app-prod-db.cxy9abc123de.us-east-1.rds.amazonaws.com \
      port=5432 dbname=app user=app_runtime \
      sslmode=verify-full sslrootcert=/etc/pki/rds/global-bundle.pem"
```

Use `VERIFY_IDENTITY` for MySQL clients and the equivalent strict mode for the
other engines. `require`/encrypted-only modes that do not validate the CA and
hostname are weaker. Inventory client trust stores and test new and old CA
bundles before changing the RDS CA; some engine versions rotate the server
certificate without restart, while others schedule maintenance.

Use a least-privileged application database user, not the RDS master user.
Network controls, IAM control-plane permissions, the secret, and database
grants are separate layers.

---

## 7. Add RDS Proxy for Connection and Failover Pressure

Place **RDS Proxy** between bursty applications (especially Lambda) and the DB
when connection creation or failover recovery is a problem:

```
app-sg ──TLS──▶ proxy endpoint / proxy-sg ──TLS──▶ RDS / rds-sg
```

- Let the app reach `proxy-sg`; let `rds-sg` accept the DB port from
  `proxy-sg`. Do not leave the old broad app-to-DB rule unless a direct path is
  intentional.
- Give the proxy's IAM role access to the exact Secrets Manager secret and KMS
  key. Clients can use database credentials or IAM database authentication
  where supported; require TLS at the proxy.
- Point the application at the **proxy endpoint**, not the DB endpoint. The
  proxy pools/multiplexes connections and can reconnect to a standby while
  preserving many application connections.
- Transactions or session state can pin a client to one database connection,
  reducing multiplexing. Watch pinning and connection-borrow metrics; RDS Proxy
  is not a substitute for bounded pools and correct retry logic.

---

## 8. Operate and Prove Recovery

| Concern | Production control |
|---------|--------------------|
| AZ/instance failure | Multi-AZ deployment; test a forced failover |
| Accidental change/corruption | Automated backups with point-in-time recovery; manual/AWS Backup copies for longer retention |
| Regional/account loss | Copy snapshots or automated backups to the recovery Region/account with an accessible destination KMS key |
| Performance | CloudWatch alarms for CPU, memory, free storage, connections, I/O latency/queue depth; Enhanced Monitoring and Database Insights for diagnosis |
| Control-plane events | RDS event subscriptions/EventBridge for failover, maintenance, low storage, backup, and certificate events |
| Planned change | Non-overlapping preferred backup and maintenance windows; stage engine/parameter/CA changes first |

Backups only count after a restore test. Restore point-in-time to a new isolated
instance, validate data and application startup, record the achieved RTO/RPO,
then remove it through the normal change process. Keep final snapshots and
deletion protection in the deletion runbook rather than relying on an operator
to remember them.

Force a Multi-AZ test with `reboot-db-instance --force-failover` in an approved
window. Measure from the first failed query until successful reads **and
writes**, not until the console says available. Existing TCP connections drop;
the application must close them, re-resolve the endpoint, and reconnect with
bounded exponential backoff plus jitter. Set language/runtime DNS caching to a
finite value (AWS recommends no more than 60 seconds for JVM clients) and make
write retries idempotent because the client might not know whether a commit
succeeded. Test through RDS Proxy separately if it is in the path.

---

## 9. Cross-Account Boundaries

A centralized database account does not make a DB instance itself a shared IAM
resource. Separate the two planes:

- **Administration**: operators and automation assume a tightly scoped IAM role
  in the database account. RDS DB resources do not use a general resource-based
  policy for cross-account control-plane administration.
- **Data connection**: route private traffic with Transit Gateway, VPC peering,
  or VPN/DX; then allow
  only the application source in the DB/proxy security group. The RDS API
  interface endpoint is for RDS API calls, not SQL traffic to a DB instance.
- **Authentication**: use a database identity with least privilege. If another
  account reads a secret, its secret resource policy, identity policy, and KMS
  key policy must all allow that access. Prefer a local assumed role so the
  database account can audit and revoke it centrally.
- **Recovery**: snapshot sharing/copying and KMS permissions are explicit. Test
  the destination account's ability to restore, not merely see the snapshot.

Network reachability never grants RDS API permission or database permission,
and an IAM role that can modify RDS does not automatically have a SQL login.

---

## 10. Comparison: Reachability Settings

| Setting / control            | Public access | Private (this pattern)        |
|------------------------------|---------------|-------------------------------|
| `Publicly Accessible`        | Yes           | **No**                        |
| Subnet route table           | IGW route     | **No IGW route**              |
| Endpoint DNS resolves to     | Public IP outside VPC, private IP inside | **Private IP** |
| Inbound SG source            | 0.0.0.0/0 ❌  | **app-sg reference** ✅       |
| DB subnet group AZ count     | **≥ 2 AZs**   | **≥ 2 AZs**                   |
| Who can connect              | Public-path clients allowed by SG | Private-path clients allowed by SG |

---

## 11. Troubleshooting

**App can't connect — connection times out**

- `rds-sg` inbound doesn't allow the app SG, or allows the wrong port (3306 vs
  5432). Confirm the rule references `app-sg` and the engine's port.
- Outbound on the **app** SG is restricted. Egress to the DB port must be
  allowed (default egress is all-allow, but locked-down SGs may block it).
- App instances are in a subnet/AZ that has no DB subnet in the same... actually
  cross-AZ traffic is fine within a VPC; check the SG first.

**App can't connect — name does not resolve**

- The endpoint is still a DNS name when `Publicly Accessible = No`, but it
  resolves to a private IP. Check VPC DNS support and the client's resolver
  path. A client outside the connected private network cannot route to the
  resulting address.

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

**TLS works with `require`, but fails with `verify-full`**

- The client lacks the current RDS CA bundle or is validating a private alias
  rather than the RDS certificate's endpoint name. Update the trust store and
  connect using a hostname covered by the server certificate.

**Rotated credentials cause intermittent authentication failures**

- A process or connection pool cached the old secret. Fetch the current secret
  for new connections, drain stale pools, and test rotation in staging. RDS
  Proxy can centralize secret use but still needs correct secret/KMS policy.

---

## Key Exam Points

- Secure RDS = **Publicly Accessible: No** + **private subnets (no IGW route)** +
  **SG inbound from the app SG**.
- A **DB subnet group needs ≥ 2 AZs**, even for single-AZ instances.
- Reference the **source security group**, not a CIDR — survives ASG churn and
  scopes access tightly.
- Apps connect via the **endpoint DNS name**, which Multi-AZ repoints on failover.
- Encrypt storage/backups with **KMS** and use TLS with CA/hostname validation.
- DB credentials ⇒ **Secrets Manager** with a tested rotation and client-refresh path.
- Production also needs **Multi-AZ, restore/failover tests, alarms, maintenance
  windows, application reconnect logic**, and explicit cross-account roles.

---

**Next**: [12_rds_multiaz_vs_read_replica.md — Multi-AZ vs Read Replicas, Side by Side](12_rds_multiaz_vs_read_replica.md)
