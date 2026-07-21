# AMIs, User Data and Instance Metadata

> **Who this is for**: SAA-C03 candidates who understand what an EC2 instance is and now need to know how it *boots*: where the OS image comes from (AMI), how the instance configures itself on first boot (User Data), and how it discovers facts about itself (Instance Metadata Service). The IMDSv1 vs IMDSv2 distinction is a recurring exam/security topic. Assumes you've read [01_ec2_fundamentals.md](01_ec2_fundamentals.md).

---

## 1. AMIs — Amazon Machine Images

An **AMI** is the template an instance is launched from. It bundles everything needed to boot:

- A **root volume image** (the OS + any pre-installed software/config), stored as **EBS snapshots** in S3.
- **Launch permissions** (who can use this AMI: private to your account, shared with specific accounts, or public).
- A **block device mapping** describing the volumes attached at launch.

```
        AMI
        ├── root volume snapshot (OS + pre-baked software)
        ├── block device mapping (which EBS volumes to attach)
        └── launch permissions (private / shared / public)
                │
                ▼  launch
           EC2 Instance  (a running copy, customized further by user data)
```

### Key AMI facts

- **Region-scoped**: an AMI exists in **one region**. To use it elsewhere you **copy** it to the target region (which copies its underlying snapshots). Copying *across accounts* is done by sharing the AMI (and snapshots) and then copying.
- **Sources**: AWS-provided (Amazon Linux, Ubuntu, Windows…), AWS Marketplace (vendor images, sometimes with per-use licensing), Community AMIs, or **your own** custom AMIs.
- ⚠️ AMI IDs differ per region. A launch template referencing `ami-0abc…` from `us-east-1` will fail in `eu-west-1`.

> **Key insight**: An AMI is a *frozen starting point*. User Data customizes the instance *after* launch; an AMI bakes customization *in* before launch. Use both together.

---

## 2. The Golden Image Pattern

A **golden image** (a.k.a. "baked AMI") is a custom AMI with your runtime, dependencies, agents, and configuration **already installed**. Instead of installing software on every boot (slow, flaky, dependent on external repos), you pre-bake it once.

```
   Bake once:                          Launch many (fast):
   base AMI → launch → install app  →  golden AMI → ASG launches N instances
   → configure → create AMI            (boot in seconds, no install step)
```

| Approach | Boot time | Reliability | When |
|----------|-----------|-------------|------|
| **Bake** (golden AMI) | Fast — software pre-installed | High — no runtime download | Production, Auto Scaling, fast scale-out |
| **Boot-time config** (User Data only) | Slow — installs each boot | Lower — depends on external repos at boot | Frequently changing config, small scripts |
| **Hybrid** (golden AMI + small user-data) | Fast | High | Most real fleets: bake the heavy stuff, inject per-instance config at boot |

- ✅ Golden images make Auto Scaling fast and deterministic — every instance is identical and boots quickly.
- 💡 **EC2 Image Builder** automates building, testing, and distributing golden AMIs on a schedule (e.g. monthly patched images).

---

## 3. Creating an AMI from a Running Instance

You can snapshot a configured instance into a reusable AMI:

1. Launch a base instance, install/configure everything.
2. **Create Image** from the instance (Console, CLI, or API).
3. AWS takes **EBS snapshots** of the attached volumes and registers an AMI that references them.
4. Launch new instances from that AMI — each new root volume is created from the snapshot.

```bash
# Create an AMI from a running/stopped instance
aws ec2 create-image \
  --instance-id i-0123456789abcdef0 \
  --name "webapp-golden-2026-06" \
  --description "App v2.3 + CloudWatch agent, patched" \
  --no-reboot   # ⚠️ skips the reboot; risks a non-clean filesystem snapshot
```

- ⚠️ By default `create-image` **reboots** the instance to guarantee filesystem consistency. `--no-reboot` avoids downtime but may capture an inconsistent state — only safe if the app is quiesced.
- The **AMI ↔ snapshot relationship**: deleting (deregistering) an AMI does **not** automatically delete its backing snapshots — you pay for orphaned snapshots until you delete them too.

---

## 4. User Data — Bootstrapping at First Boot

**User Data** is a script (or cloud-init directive) you pass at launch that runs **once, at first boot**, as **root**, to configure the instance — install packages, fetch config, register with a service, etc.

Key facts:
- Runs **once on the first boot** by default (not on every reboot).
- Executed by **cloud-init** on most Linux AMIs; runs as **root** (no `sudo` needed).
- Sent **base64-encoded** to the API; the Console/CLI handle the encoding for you.
- Limited to **16 KB** (raw, before encoding) — for larger setups, have user data *download* a bigger script.
- Logs to `/var/log/cloud-init-output.log` — your first stop when bootstrapping "didn't work."

### Real bash User Data example

This installs and starts a web server, then writes a page that reports the instance's own ID (pulled from metadata — see §5):

```bash
#!/bin/bash
# User data runs as root at first boot. Fail fast and log everything.
set -euxo pipefail
exec > >(tee /var/log/user-data.log | logger -t user-data -s 2>/dev/console) 2>&1

# 1. Patch and install packages (Amazon Linux 2023 uses dnf)
dnf update -y
dnf install -y nginx

# 2. Fetch an IMDSv2 token, then read this instance's ID from metadata
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)

# 3. Render a simple landing page proving which instance served the request
cat > /usr/share/nginx/html/index.html <<EOF
<h1>Served by ${INSTANCE_ID}</h1>
<p>Availability Zone: ${AZ}</p>
EOF

# 4. Enable and start the service so it survives reboots
systemctl enable --now nginx
```

- ✅ `set -euxo pipefail` + logging is the production pattern — without it, a failing line is silently skipped and you get a half-configured instance.
- 💡 To re-run user data on **every** boot, add a cloud-init directive (`#cloud-config` with `cloud_final_modules: [scripts-user]` set to `always`) — but the default and exam-expected behavior is **once**.

> **Rule**: User Data = "what to do the first time this instance boots." It's how Auto Scaling instances configure themselves without manual login.

---

## 5. Instance Metadata Service (IMDS)

From *inside* an instance, the **Instance Metadata Service** at the special link-local address **`169.254.169.254`** exposes data about the instance: its ID, instance type, AZ, security groups, attached IAM role credentials, the public key, user data, and more.

```
   inside the instance:
   ┌───────────────────────────────────────────────┐
   │  curl http://169.254.169.254/latest/meta-data/│
   │     instance-id        ami-id                 │
   │     placement/...      security-groups        │
   │     local-ipv4         public-ipv4            │
   │     iam/security-credentials/<role>  ← creds! │
   └───────────────────────────────────────────────┘
   169.254.169.254 = link-local; never routed off the host
```

- **Metadata** (facts about the instance) vs **User Data** (the bootstrap script you supplied — also readable at `/latest/user-data`).
- The endpoint is the mechanism by which the **IAM role** attached to an instance delivers **temporary credentials** to the SDK/CLI — apps shouldn't hardcode keys; they read role creds from IMDS automatically.

### IMDSv1 vs IMDSv2 — and why IMDSv2 matters

| | **IMDSv1** | **IMDSv2** |
|--|-------------|-------------|
| Request flow | Simple `GET` (request/response) | **Session-oriented**: `PUT` to get a token, then `GET` with the token header |
| Auth | None | Requires the token (`X-aws-ec2-metadata-token`) on every request |
| Defends against SSRF? | ❌ No | ✅ Yes — a tricked server can't be made to forward a token-bearing PUT easily |
| Hop limit | n/a | Default TTL **1 hop** — blocks containers/proxies from reaching it inadvertently |

**Why IMDSv2 matters (the security story)**: with IMDSv1, a **Server-Side Request Forgery (SSRF)** vulnerability in your app could be abused to make the app fetch `169.254.169.254/.../iam/security-credentials/...` and leak the instance's IAM role credentials to an attacker. IMDSv2 requires a `PUT` to obtain a token first (with a header and a default 1-hop TTL), which typical SSRF/reverse-proxy attacks can't perform — closing that exfiltration path.

```bash
# IMDSv1 — single GET (no token). Vulnerable to SSRF abuse.
curl http://169.254.169.254/latest/meta-data/instance-id

# IMDSv2 — get a token first, then use it. Default and recommended.
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

- ✅ Set instances to **IMDSv2-required** (`HttpTokens=required`) to disable the legacy v1 path entirely. New AMIs/launch templates increasingly default to this.
- 💡 Metadata and user data are **not encrypted** — never put secrets in user data; use **Secrets Manager** or **SSM Parameter Store** instead.

---

## 6. Controlled Image Delivery at Enterprise Scale

A golden image becomes useful only when its build and promotion path is
repeatable. **EC2 Image Builder** turns that path into a pipeline:

```
versioned recipe + components + base image
                 |
                 v
          build -> test -> scan
                 |
                 v
        distribute approved AMI
                 |
                 v
  new launch-template version -> canary -> fleet rollout
```

Keep installation and hardening steps in versioned components. Run functional
and security tests before distribution, tag the output with provenance and an
expiry date, and promote by AMI ID rather than rebuilding independently in each
environment. Failed tests should stop distribution.

### Cross-account encrypted AMIs

Sharing the AMI alone is insufficient when its snapshots are encrypted:

1. Encrypt with a **customer managed KMS key**; snapshots encrypted only with
   the AWS managed EBS key cannot be shared directly across accounts.
2. Grant the destination account launch permission on the AMI and permission to
   use the KMS key. AMI sharing gives launch access to its referenced snapshots,
   so they do not need a separate snapshot-share operation; the KMS key policy
   is still required, and an IAM allow in the destination cannot replace it.
3. Have the destination copy the AMI using a KMS key it controls. Launch from
   that local copy so the workload is not permanently dependent on the source
   account's image permission and key lifecycle.
4. Test launch, boot, agent registration, and application health before making
   the image the default.

For many accounts and Regions, make Image Builder distribution settings and
Organizations-based sharing part of the pipeline. Keep the source image until
all destinations confirm a usable copy; a revoked grant or scheduled key
deletion can otherwise break a recovery path.

### Launch templates are the release pointer

A launch template is versioned, but editing it creates a new version; it does
not mutate instances or necessarily change what an Auto Scaling group launches.
Use an explicit tested version in production, update the Auto Scaling group,
and perform an instance refresh or blue/green replacement. Set the template's
default version deliberately, and do not let an unreviewed `$Latest` silently
become production.

If a canary fails, point the group back to the previous template version and
replace the failed instances. The previous AMI must still exist and its KMS key,
snapshots, IAM profile, security groups, and bootstrap dependencies must remain
usable for rollback.

### Immutable image or configuration management?

| Change | Prefer | Reason |
|--------|--------|--------|
| OS packages, runtime, security agents | New image | Deterministic boot and a testable rollback unit. |
| Small environment-specific values | User data or parameter retrieval | Avoids one image per environment; never embed secrets. |
| Required state on long-lived instances | Systems Manager State Manager | Detects and reapplies desired configuration without manual login. |
| Emergency remediation | Tested Automation/State Manager association, followed by a rebuilt image | Restores safety quickly without making the emergency mutation the permanent source of truth. |

Immutable replacement reduces configuration drift but needs image storage,
pipeline, capacity, and rollout discipline. In-place configuration is useful for
long-lived or licensed hosts, but increases drift and rollback complexity. Most
fleets use a hybrid: bake slow, security-sensitive dependencies; retrieve small
environment configuration at boot; continuously verify required state.

---

## Key Exam Points

- ✅ An **AMI** = OS image + block device mapping + launch permissions; backed by **EBS snapshots**; **region-scoped** — **copy** it to use in another region/account.
- ✅ **Golden image** = pre-baked custom AMI → fast, deterministic boots (great for Auto Scaling). EC2 Image Builder automates it.
- ✅ Creating an AMI takes snapshots; deregistering an AMI does **not** delete its snapshots (you keep paying).
- ✅ **User Data** runs **once at first boot, as root**, base64-encoded, **16 KB** limit, via cloud-init; logs in `/var/log/cloud-init-output.log`.
- ✅ **IMDS** lives at **`169.254.169.254`**, serves instance facts and **IAM role temporary credentials**.
- ✅ **IMDSv2** is token-based (PUT then GET, 1-hop TTL) and defends against **SSRF credential theft** — prefer/require it.
- ✅ Never store secrets in user data or metadata — use Secrets Manager / SSM Parameter Store.
- ✅ Image Builder should build, test, and distribute a versioned image; a launch-template version is the production release pointer.
- ✅ Cross-account encrypted AMI delivery requires AMI launch permission **and** a customer managed KMS key policy; copy into a destination-owned key.
- ✅ Use immutable replacement for deterministic fleet changes and State Manager where controlled in-place state is required.

---

## Common Mistakes

- ❌ Reusing an AMI ID across regions — AMI IDs are region-specific; copy first.
- ❌ Expecting user data to run on every reboot — by default it runs **once** at first boot.
- ❌ Putting passwords/keys in user data — it's readable via metadata and stored unencrypted.
- ❌ Forgetting that deregistering an AMI leaves orphaned snapshots you still pay for.
- ❌ Leaving **IMDSv1 enabled** on internet-facing apps — an SSRF flaw can exfiltrate IAM role credentials.
- ❌ Using `--no-reboot` on a busy instance and getting an inconsistent filesystem in the AMI.
- ❌ Sharing an encrypted AMI without granting use of its snapshots and KMS key, or leaving the destination dependent on a source-owned key.
- ❌ Updating a launch template but forgetting to update and replace the running Auto Scaling fleet.

---

## Relevant Limits

- **User Data size**: 16 KB (raw, before base64 encoding).
- **IMDS address**: `169.254.169.254` (link-local, not routable off the host).
- **IMDSv2 token TTL**: up to 6 hours (`21600` seconds); default response hop limit **1**.
- An AMI can reference multiple EBS snapshots (one per mapped volume).

---

**Next**: [Part 4: Placement Groups](04_placement_groups.md)
