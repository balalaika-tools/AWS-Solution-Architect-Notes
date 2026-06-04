# Threat Detection & Protection Services

> **Who this is for**: Engineers preparing for SAA-C03 who need to tell apart AWS's alphabet
> soup of security services — WAF, Shield, GuardDuty, Inspector, Macie, Detective, Security Hub —
> and pick the right one for a scenario. The exam tests this almost entirely as a *"which service
> for which problem"* discriminator. Before this, read
> **[01_encryption_and_kms.md](01_encryption_and_kms.md)** and
> **[02_secrets_manager_and_acm.md](02_secrets_manager_and_acm.md)**; familiarity with
> [Networking](../03_networking/README.md) (VPC Flow Logs) and
> [Monitoring](../12_monitoring/README.md) (CloudTrail) helps.

---

## 1. Two Categories: Protective vs Detective

Mentally split these services before memorizing them:

- **Protective (block attacks in real time)** — WAF, Shield, AWS Network Firewall, Gateway Load
  Balancer appliance fleets, Firewall Manager.
- **Detective (find threats / vulnerabilities / sensitive data)** — GuardDuty, Inspector, Macie,
  Detective, Security Hub.

```
        INTERNET
           │   attacks (DDoS, SQLi, XSS)
           ▼
   ┌──────────────────┐   protective layer
   │ Shield  +  WAF   │  ← blocks at the edge / L7
   └────────┬─────────┘
            ▼
       your workload (EC2, S3, RDS, Lambda …)
            │ logs (VPC Flow, DNS, CloudTrail)
            ▼
   ┌────────────────────────────────────────────┐  detective layer
   │ GuardDuty · Inspector · Macie · Detective   │  ← analyze, alert, investigate
   └───────────────────────┬────────────────────┘
                           ▼
                   ┌────────────────┐
                   │  Security Hub  │  ← aggregates all findings + compliance
                   └────────────────┘
```

---

## 2. WAF — Web Application Firewall (Layer 7)

A **Layer 7** firewall that inspects HTTP/HTTPS requests and blocks application-layer attacks —
**SQL injection, cross-site scripting (XSS)**, bad bots, and request floods.

- Configured as a **Web ACL** containing **rules**; each rule allows/blocks/counts requests based
  on conditions (IP set, geo-match, string/regex match, size, SQLi/XSS detection).
- **AWS Managed Rule Groups** — pre-built rule sets (e.g., OWASP Top 10, known-bad inputs) you
  enable without writing rules yourself.
- **Rate-based rules** — block an IP that exceeds N requests in 5 minutes (basic application-layer
  flood protection).
- Attaches to **CloudFront, Application Load Balancer (ALB), API Gateway REST APIs, AppSync, and
  Cognito** — **not** to NLB, EC2 directly, or API Gateway HTTP APIs.

⚠️ WAF is **Layer 7 only**. For Layer 3/4 volumetric DDoS, that's Shield's job, not WAF's.

---

## 3. Shield — DDoS Protection (Standard vs Advanced)

Protects against **DDoS** (Distributed Denial of Service) attacks.

| | **Shield Standard** | **Shield Advanced** |
|---|---|---|
| Cost | **Free**, automatic for all AWS customers | **~$3,000 / month** (per org) + data fees |
| Protection layer | Common **L3/L4** volumetric & SYN/UDP floods | Enhanced L3/L4 **and** L7 (with WAF) |
| Resources | CloudFront, Route 53 edge | + ELB, EC2 EIPs, Global Accelerator, Route 53 |
| **DDoS Response Team (DRT/SRT)** | ❌ | ✅ 24/7 access during attacks |
| **Cost protection** | ❌ | ✅ Refunds scaling charges caused by a DDoS |
| Real-time attack visibility | Basic | ✅ Detailed dashboards & metrics |
| WAF included | Pay separately | ✅ WAF fees included on protected resources |

> **Rule**: Shield Standard is on by default and free. Choose **Shield Advanced** when the
> scenario mentions a need for the **DDoS Response Team**, **cost protection against DDoS-driven
> scaling bills**, or guaranteed enhanced protection for a business-critical app.

---

## 4. GuardDuty — Intelligent Threat Detection

Continuous, **agentless** threat detection that uses ML/anomaly detection over **VPC Flow Logs,
DNS query logs, and CloudTrail event logs** (plus optional S3, EKS, RDS, Lambda data sources).

- Detects things like: an EC2 instance talking to a known crypto-mining domain, credential
  exfiltration, port-scanning, anomalous API calls, or access from a Tor exit node.
- ✅ **No agents to install, no log pipelines to build** — you just enable it; it reads the logs
  natively. This "no agents / analyzes flow + DNS + CloudTrail" phrasing is the dead giveaway.
- Produces **findings** that can trigger EventBridge → Lambda/SNS for automated response.

💡 GuardDuty answers *"is something malicious happening **right now** in my account/network?"*

---

## 5. Inspector — Vulnerability Scanning

Automated **vulnerability management**: scans for software CVEs and unintended network exposure.

- Targets: **EC2 instances, ECR container images, and Lambda functions**.
- Continuously scans against CVE databases; produces prioritized findings with risk scores.
- Uses the **SSM Agent** on EC2 (so EC2 must be SSM-managed).

💡 Inspector answers *"do my workloads have **known vulnerabilities / patches missing**?"* — a
software-state question, not a live-attack question (that's GuardDuty).

---

## 6. Macie — Sensitive Data Discovery in S3

Uses **machine learning + pattern matching** to discover and classify **sensitive data (PII,
PHI, credentials, financial data)** stored in **S3**.

- Reports which buckets contain sensitive data, and flags buckets that are public, unencrypted,
  or shared externally.
- Scope is **S3-specific**. If the question is "find PII / credit-card numbers in our buckets,"
  the answer is **Macie**.

---

## 7. Detective — Root-Cause Investigation

Once GuardDuty (or another source) raises a finding, **Detective** helps you **investigate**: it
automatically builds a **linked behavior graph** from VPC Flow Logs, CloudTrail, and GuardDuty
findings so you can answer *"how did this happen, what else was touched?"*

💡 GuardDuty **detects**; Detective **investigates the root cause** of what was detected. The
words "investigate / root cause / behavior graph" point to Detective.

---

## 8. Security Hub — Aggregation & Compliance

A **single pane of glass** that **aggregates and normalizes findings** from GuardDuty, Inspector,
Macie, IAM Access Analyzer, and partner tools across accounts.

- Runs **automated compliance checks** against standards: **CIS AWS Foundations, PCI DSS, AWS
  Foundational Security Best Practices**.
- Deduplicates and prioritizes findings org-wide.

💡 "Aggregate findings from multiple security services" or "check against CIS/PCI compliance
standards" → **Security Hub**.

---

## 9. Firewall Manager (brief)

**AWS Firewall Manager** centrally configures and enforces **WAF rules, Shield Advanced
protections, security groups, and Network Firewall policies across all accounts in an
Organization**. Use it when the scenario is *"enforce the same firewall/WAF policy across many
accounts centrally."*

---

## 10. Network Firewall vs GWLB vs WAF

These sound similar but sit in different places.

| Need | Choose | Why |
|------|--------|-----|
| Inspect/filter HTTP requests for SQLi, XSS, bots, rate limits | **AWS WAF** | Layer 7 web ACL on CloudFront/ALB/API Gateway/AppSync |
| Managed stateful/stateless firewall inside VPC subnet routing | **AWS Network Firewall** | AWS-managed network firewall for VPC traffic paths |
| Use third-party firewall/IDS/IPS appliances and scale them | **Gateway Load Balancer** | Load-balances appliance fleet with GENEVE traffic |
| Centrally enforce WAF/Shield/Network Firewall/SG policy across accounts | **Firewall Manager** | Organization-level policy management |

> **Rule**: WAF is for HTTP application requests. Network Firewall is AWS-managed VPC network
> inspection. GWLB is for third-party appliances. Firewall Manager is central governance.

---

## 11. Which Service for Which Problem? (the exam picker)

| The scenario asks for… | Service |
|------------------------|---------|
| Block **SQL injection / XSS** / bad HTTP requests | **WAF** |
| **Rate-limit** an IP flooding a web app | **WAF** (rate-based rule) |
| Free baseline **DDoS** protection | **Shield Standard** |
| **DDoS Response Team** + cost protection for critical app | **Shield Advanced** |
| Managed network firewall for traffic entering/leaving VPC subnets | **AWS Network Firewall** |
| Scale third-party firewall / IDS / IPS appliances inline | **Gateway Load Balancer** |
| Detect **malicious/anomalous activity** from VPC/DNS/CloudTrail logs, **no agents** | **GuardDuty** |
| Find **OS/software vulnerabilities (CVEs)** on EC2/ECR/Lambda | **Inspector** |
| Discover **PII / sensitive data in S3** | **Macie** |
| **Investigate root cause** of a security finding (behavior graph) | **Detective** |
| **Aggregate findings** from many security tools / check **CIS/PCI** compliance | **Security Hub** |
| **Centrally enforce** WAF/Shield/SG policies across an **Organization** | **Firewall Manager** |

> **Key insight**: The fastest way to disambiguate on the exam — **WAF/Shield = stop attacks at
> the front door**, **GuardDuty = "is something bad happening now?"**, **Inspector = "am I
> vulnerable?"**, **Macie = "is sensitive data exposed in S3?"**, **Detective = "why/how did it
> happen?"**, **Security Hub = "show me everything in one place + compliance."**

---

## Key Exam Points

- **WAF** is **Layer 7**, attaches to **CloudFront / ALB / API Gateway REST APIs** (not NLB/EC2),
  blocks SQLi/XSS, supports managed rule groups and rate-based rules.
- **Shield Standard** is free and automatic; **Shield Advanced** adds the **DDoS Response Team**
  and **cost protection** for ~$3k/month.
- **Network Firewall** = AWS-managed VPC traffic inspection; **GWLB** = scale third-party network
  appliances.
- **GuardDuty** is **agentless** anomaly detection over **VPC Flow Logs, DNS logs, CloudTrail**.
- **Inspector** scans **EC2/ECR/Lambda** for **CVEs / vulnerabilities**.
- **Macie** finds **PII/sensitive data in S3**.
- **Detective** = root-cause **investigation** (behavior graph); **Security Hub** = **aggregation
  + compliance standards** (CIS, PCI, FSBP).
- **Firewall Manager** = central policy enforcement across an **Organization**.

---

## Common Mistakes

- ❌ Picking **WAF** for a volumetric L3/L4 DDoS — that's **Shield**. WAF is L7 only.
- ❌ Confusing **GuardDuty** (detects active threats from logs) with **Inspector** (scans for
  software vulnerabilities). "No agents, analyzes traffic/DNS/API logs" = GuardDuty; "CVEs /
  patch state on EC2" = Inspector.
- ❌ Choosing **Macie** for anything outside **S3** — its scope is S3 sensitive-data discovery.
- ❌ Choosing **Security Hub** to *investigate* an incident — that's **Detective**; Security Hub
  *aggregates* findings and runs compliance checks.
- ❌ Trying to attach **WAF** to an **NLB** or **EC2** instance directly — it attaches to
  CloudFront/ALB/API Gateway.
- ❌ Picking **GWLB** when the requirement is an AWS-managed firewall policy — use Network Firewall.
- ❌ Forgetting that **Shield Advanced** is what unlocks **cost protection** against DDoS-driven
  auto-scaling bills.

---

**Next**: [Hybrid Networking — Direct Connect & VPN](../14_hybrid_migration_dr/01_hybrid_networking.md)
