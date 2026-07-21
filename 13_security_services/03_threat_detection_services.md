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
| Cost | No additional charge; automatic for all AWS customers | **$3,000/month per payer account**, a one-year subscription commitment, plus usage-based charges |
| Protection layer | Common **L3/L4** volumetric & SYN/UDP floods | Enhanced L3/L4 **and** L7 (with WAF) |
| Resources | CloudFront, Route 53 edge | + ELB, EC2 EIPs, Global Accelerator, Route 53 |
| **Shield Response Team (SRT)** | ❌ | ✅ Access requires **Business Support or Enterprise Support** |
| **Cost protection** | ❌ | ✅ Credits for eligible DDoS-related scaling charges, subject to conditions |
| Real-time attack visibility | Basic | ✅ Detailed dashboards & metrics |
| WAF allowance | Pay separately | Includes the documented WAF request/WCU allowance for protected resources; excess and optional managed features remain billable |

> **Rule**: Shield Standard is on by default and free. Choose **Shield Advanced** when the
> scenario mentions a need for the **Shield Response Team**, **cost protection against DDoS-driven
> scaling bills**, or guaranteed enhanced protection for a business-critical app.

Do not reduce Advanced pricing to “about $3,000 and WAF is free.” The current subscription has a
**one-year commitment** (and renews annually), its monthly fee is accompanied by usage-based data
transfer charges for protected services, and WAF usage beyond the included allowance or optional
Bot Control/Fraud Control/Marketplace features is billed separately. Shield Advanced is
available without a premium support plan, but **SRT access specifically requires Business or
Enterprise Support**. Verify commercial details against the
[official Shield pricing page](https://aws.amazon.com/shield/pricing/) when designing or quoting
the service.

---

## 4. GuardDuty — Intelligent Threat Detection

Continuous threat detection whose foundational log analysis is **agentless**, using
ML/anomaly detection over **VPC Flow Logs,
DNS query logs, and CloudTrail event logs** (plus optional S3, EKS, RDS, Lambda data sources).

- Detects things like: an EC2 instance talking to a known crypto-mining domain, credential
  exfiltration, port-scanning, anomalous API calls, or access from a Tor exit node.
- ✅ **No agents to install, no log pipelines to build** — you just enable it; it reads the logs
  natively for foundational sources. Optional Runtime Monitoring uses a GuardDuty security agent
  or supported integration, so “GuardDuty never uses an agent” is too broad.
- Produces **findings** that can trigger EventBridge → Lambda/SNS for automated response.

💡 GuardDuty answers *"is something malicious happening **right now** in my account/network?"*

---

## 5. Inspector — Vulnerability Scanning

Automated **vulnerability management**: scans for software CVEs and unintended network exposure.

- Targets: **EC2 instances, ECR container images, and Lambda functions**.
- Continuously scans against CVE databases; produces prioritized findings with risk scores.
- EC2 coverage can use agent-based scanning through Systems Manager or agentless scanning where
  supported. ECR and Lambda use their service-specific scanning integrations.

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
| **Shield Response Team** + cost protection for critical app | **Shield Advanced** |
| Managed network firewall for traffic entering/leaving VPC subnets | **AWS Network Firewall** |
| Scale third-party firewall / IDS / IPS appliances inline | **Gateway Load Balancer** |
| Detect **malicious/anomalous activity** from VPC/DNS/CloudTrail logs without deploying a log pipeline/agent for those sources | **GuardDuty** |
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

## 12. Organization-Wide Administration

An enterprise deployment normally has four account roles:

```text
Organizations management account   delegates administration; stays out of daily operations
Security tooling account            GuardDuty / Security Hub / Inspector / Macie / Detective
Network account                     Firewall Manager and centrally governed network controls
Log archive account                 immutable CloudTrail, network, WAF, and finding retention
```

Use a delegated administrator instead of operating daily security tooling from the Organizations
management account. Delegation does not make these services global: most enablement, policies,
findings, and behavior graphs remain Regional.

| Service | Delegated-administration and enrollment detail |
|---------|------------------------------------------------|
| **GuardDuty** | Use one delegated administrator consistently, then enable/configure every governed Region. Organization settings can enroll `ALL`, `NEW`, or `NONE` accounts per Region and per protection plan. `ALL` covers existing and future accounts; enabling GuardDuty does not automatically enable every optional plan. |
| **Security Hub** | Choose a delegated administrator and a **home Region** with linked Regions. Central-configuration policies can enable the service, standards, and controls for accounts/OUs across those Regions; the home Region is also the finding-aggregation Region. |
| **Inspector** | Designate the same administrator, activate it in each required Region, and use Organizations policies to auto-enable the intended EC2, ECR, and Lambda scan types for existing and new accounts. Coverage gaps are a finding in themselves. |
| **Macie** | Membership and sensitive-data discovery are Regional. The administrator can automatically enroll **new** organization accounts in each Region, but existing accounts need deliberate enrollment and discovery jobs still need scope, schedule, sampling, and cost control. |
| **Firewall Manager** | Assign a default or appropriately scoped administrator. Policies can be scoped by accounts/OUs, Regions, resource tags, and protection type. Most policies are Regional (CloudFront is global), so deploy a policy per required Region and satisfy the Organizations/AWS Config prerequisites. |
| **Detective** | Designate an administrator and enable a behavior graph per Region. Auto-enrollment applies to new organization accounts when configured; existing accounts require explicit enablement. A graph in one Region does not investigate activity in another. |

### Region and account rollout

Start with an explicit Region strategy: enabled workload Regions, Regions denied by SCP, and
Regions still monitored because global-service activity or unauthorized use could appear there.
AWS recommends GuardDuty in all supported Regions. For every allowed Region, deploy the service,
delegated-admin association, organization settings, EventBridge routes, Config recorder/rules,
log destinations, and notification targets as one versioned stack. Test new-account enrollment;
do not assume an Organizations invitation means every protection plan is active.

Apply organization policies to OUs rather than hand-configuring accounts. Roll out in audit/count
mode where the service supports it, canary on non-production OUs, then enforce. Maintain exception
expiry and ownership. A central team should continuously reconcile the account inventory against
service membership and coverage, including suspended accounts and newly enabled Regions.

### Aggregate findings, then preserve source evidence

Security Hub normalizes integrated findings into a common format and aggregates linked-Region
findings into its home Region. That makes it the prioritization and routing layer; it does not
replace the source service's detail, CloudTrail, WAF/network logs, or long-term evidence store.
Export findings and underlying logs to the log-archive/SIEM path for the organization's required
retention because service consoles have finite finding-retention windows.

Route actionable finding changes through EventBridge. A normal response path is:

```text
GuardDuty / Inspector / Macie / Security Hub finding
  → EventBridge rule (severity, product, resource, workflow state)
  → Step Functions or Systems Manager Automation runbook
  → validate account/resource + acquire idempotency lock
  → contain automatically OR request approval
  → collect evidence, create ticket, notify owner
  → update finding workflow and verify recovery
```

Systems Manager Automation supplies reusable runbooks, cross-account execution, concurrency and
error thresholds, and approval steps. Keep destructive actions behind approval unless the signal
has high confidence and the containment is safe and reversible. Make every runbook idempotent,
record before/after state, restrict its execution role, cap concurrency, test rollback, and avoid
letting an attacker-controlled finding field become an unchecked command parameter.

---

## 13. Combine Preventive, Detective, and Responsive Controls

No service in this chapter is a complete control by itself:

| Layer | Purpose | Representative controls | Limitation |
|-------|---------|-------------------------|------------|
| **Preventive** | Make unsafe changes impossible or reduce exposure | SCPs, IAM/key/resource policies, WAF, Network Firewall, Firewall Manager, service control settings | An SCP sets maximum permissions but grants none, does not apply to the management account, and cannot detect every data-plane attack |
| **Detective** | Identify drift, exposure, vulnerabilities, and malicious behavior | AWS Config, CloudTrail, IAM Access Analyzer, GuardDuty, Inspector, Macie, Security Hub | A finding does not contain or repair the problem; CloudTrail records API activity but is not a configuration-state engine |
| **Responsive** | Contain, recover, preserve evidence, and notify | EventBridge, Systems Manager Automation, Step Functions, Lambda, incident process | Blind automation can destroy evidence, lock out responders, or amplify a false positive |

**AWS Config** evaluates current and historical resource configuration and can invoke an SSM
remediation. **CloudTrail** records who made API changes; use an organization trail and include
the data events required by the threat model. **IAM Access Analyzer** finds external/internal or
unused access and validates policies, but it does not block access. **Network Firewall** performs
stateful/stateless inspection only for traffic routed through its endpoints. **SCPs** bound what
member-account identities can do; they do not grant access or repair an already unsafe resource.

### Scenario A: sensitive S3 data is accidentally exposed

- **Prevent**: an SCP protects account-level S3 Block Public Access and approved Regions; bucket
  policy/IAM conditions restrict principals and TLS; customer-managed KMS policies separate data
  use from key administration. Apply classification tags, lifecycle retention, and Object Lock
  where records require immutability.
- **Detect**: Config checks public-access/encryption/logging settings, IAM Access Analyzer reports
  external bucket access, Macie classifies sensitive objects, CloudTrail records policy changes
  and selected S3 data events, and Security Hub correlates the findings.
- **Respond**: EventBridge starts an Automation runbook that captures the current policy and
  relevant CloudTrail evidence, blocks public access or quarantines the policy, preserves objects,
  notifies the data owner, and opens an incident. Human approval should precede deletion or broad
  access revocation that could interrupt a critical producer.

The classification determines retention and response priority. “Public test fixture” and
“regulated customer records” should not share the same Macie schedule, log retention, or
containment SLA.

### Scenario B: a credential is used from an unexpected network

- **Prevent**: SCPs deny high-risk services/Regions outside the approved model; IAM uses short
  sessions, MFA, and least privilege; resource and KMS policies constrain organization/source;
  Network Firewall and egress policies restrict known workload paths.
- **Detect**: GuardDuty identifies anomalous API/DNS/network behavior, CloudTrail supplies the API
  timeline, IAM Access Analyzer shows unintended trust, and Detective builds the Regional
  relationship graph. Security Hub routes the combined high-severity case.
- **Respond**: a Step Functions/Automation workflow snapshots evidence, disables or updates the
  affected access key/role path, revokes sessions where supported, quarantines compromised
  compute, rotates dependent secrets, and invokes an approved break-glass role. Verify business
  recovery before closing the finding.

### Scenario C: a vulnerable internet-facing workload is exploited

- **Prevent**: deploy a hardened image and patch policy; use WAF managed/rate rules at Layer 7,
  Shield for DDoS, and Firewall Manager to enforce WAF/Network Firewall/security-group policy
  across accounts. Route only intended VPC paths through Network Firewall endpoints.
- **Detect**: Inspector reports vulnerable packages/images/functions, Config detects drift from
  the approved network and patch baseline, GuardDuty detects exploitation behavior, and WAF and
  Network Firewall logs support investigation.
- **Respond**: Automation removes the target from service or applies a quarantine security group,
  takes required forensic snapshots, patches or replaces it from a clean image, and verifies the
  load balancer and application before restoring traffic. Prefer immutable replacement when the
  host's integrity is uncertain.

### Scenario D: a KMS key is disabled or scheduled for deletion

- **Prevent**: key policy separates usage from administration; an SCP denies destructive KMS
  actions except approved deployment and break-glass roles; deletion requires change approval.
- **Detect**: an organization CloudTrail/EventBridge rule catches `DisableKey`, `PutKeyPolicy`,
  `CreateGrant`, and `ScheduleKeyDeletion`; Config detects disabled/nonconforming keys; Access
  Analyzer validates policy changes; Security Hub/SIEM opens the case.
- **Respond**: the runbook preserves the event and policy, cancels pending deletion or re-enables
  the key when authorized, revokes malicious grants, restores the reviewed policy, and tests
  decrypting a known recovery artifact. Never delete suspected ciphertext during containment.

These scenarios deliberately overlap controls. WAF cannot see an IAM API call; GuardDuty does not
patch a CVE; Config cannot explain a full actor timeline; Security Hub does not enforce an SCP;
and an SCP cannot stop a vulnerability exploit through an action the workload legitimately needs.

---

## Key Exam Points

- **WAF** is **Layer 7**, attaches to **CloudFront / ALB / API Gateway REST APIs** (not NLB/EC2),
  blocks SQLi/XSS, supports managed rule groups and rate-based rules.
- **Shield Standard** is automatic with no additional charge. **Shield Advanced** is
  $3,000/month per payer account plus usage charges with a one-year commitment; SRT access also
  requires Business or Enterprise Support.
- **Network Firewall** = AWS-managed VPC traffic inspection; **GWLB** = scale third-party network
  appliances.
- **GuardDuty's foundational detection is agentless** over **VPC Flow Logs, DNS logs, and
  CloudTrail**; optional Runtime Monitoring can use a security agent.
- **Inspector** scans **EC2/ECR/Lambda** for **CVEs / vulnerabilities**.
- **Macie** finds **PII/sensitive data in S3**.
- **Detective** = root-cause **investigation** (behavior graph); **Security Hub** = **aggregation
  + compliance standards** (CIS, PCI, FSBP).
- **Firewall Manager** = central policy enforcement across an **Organization**.
- Delegated administration is not global enablement: configure membership, protection plans,
  policies, findings, and remediation in every governed Region.
- Security Hub aggregates and routes findings; EventBridge plus Systems Manager Automation or
  Step Functions performs controlled, auditable remediation.

---

## Common Mistakes

- ❌ Picking **WAF** for a volumetric L3/L4 DDoS — that's **Shield**. WAF is L7 only.
- ❌ Confusing **GuardDuty** (detects active threats from logs) with **Inspector** (scans for
  software vulnerabilities). "Natively analyzes traffic/DNS/API logs" = GuardDuty; "CVEs /
  patch state on EC2" = Inspector. Optional GuardDuty Runtime Monitoring can use an agent.
- ❌ Choosing **Macie** for anything outside **S3** — its scope is S3 sensitive-data discovery.
- ❌ Choosing **Security Hub** to *investigate* an incident — that's **Detective**; Security Hub
  *aggregates* findings and runs compliance checks.
- ❌ Trying to attach **WAF** to an **NLB** or **EC2** instance directly — it attaches to
  CloudFront/ALB/API Gateway.
- ❌ Picking **GWLB** when the requirement is an AWS-managed firewall policy — use Network Firewall.
- ❌ Forgetting that **Shield Advanced** is what unlocks **cost protection** against DDoS-driven
  auto-scaling bills.
- ❌ Assuming the Shield Advanced monthly fee removes its commitment, usage charges, support
  requirement for SRT access, or every possible WAF charge.
- ❌ Designating an organization administrator and assuming existing accounts, optional plans,
  and every Region enrolled themselves.
- ❌ Automatically deleting or isolating resources on any finding without checking confidence,
  preserving evidence, limiting concurrency, and providing a recovery path.

---

**Next**: [Hybrid Networking — Direct Connect & VPN](../14_hybrid_migration_dr/01_hybrid_networking.md)
