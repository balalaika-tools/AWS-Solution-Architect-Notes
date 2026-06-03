# AWS Directory Services

> **Who this is for**: Engineers choosing how to bring Active Directory into AWS — or proxy into
> an existing on-prem AD. Read [04_organizations_sts_federation.md](04_organizations_sts_federation.md)
> first (federation context).

---

## 1. Why Directory Services?

Many enterprise workloads depend on **Microsoft Active Directory (AD)** for authentication:
Windows EC2 instances need domain-join, corporate users want AD credentials for AWS apps
(WorkSpaces, RDS, IAM Identity Center), and federated SSO pipelines need an identity source.
AWS offers three managed options that differ on where the directory lives and whether an
on-prem AD is required.

---

## 2. The Three Options

| | **AWS Managed Microsoft AD** | **AD Connector** | **Simple AD** |
|---|------------------------------|-----------------|---------------|
| What it is | Full Microsoft AD hosted in AWS | Proxy — no directory stored in AWS | Samba 4-compatible lightweight AD in AWS |
| On-prem AD required? | No (but can trust one) | **Yes — proxies all auth to on-prem** | No |
| Stores users in AWS? | **Yes** | **No** | Yes |
| On-prem trust relationship | Yes (forest/domain trust) | n/a — it *is* on-prem | **No** |
| Scale | Thousands of users | Bounded by on-prem AD | ≤ 500 or ≤ 5 000 users (two sizes) |
| GPOs / schema extensions | Yes | Via on-prem only | Limited (no full GPO) |
| Cost | Highest | Medium | Lowest |

---

## 3. Decision Tree

```
Do you have an existing on-prem Active Directory?
├── YES
│   ├── Keep all users/auth in on-prem — no AWS-side directory? → AD Connector
│   └── Want a full AWS-resident AD that can trust on-prem?     → AWS Managed Microsoft AD
└── NO
    ├── Need GPOs, trusts, or full Microsoft AD features?        → AWS Managed Microsoft AD
    └── Small workload, basic LDAP + Kerberos, no trusts?        → Simple AD
```

---

## 4. Integration Points

| AWS Service | Works with |
|-------------|-----------|
| **IAM Identity Center** | Any of the three (as an identity source for SSO across accounts) |
| **WorkSpaces / AppStream** | Managed Microsoft AD or AD Connector (domain-join required) |
| **RDS (SQL Server, Oracle)** | Managed Microsoft AD (Windows Authentication) |
| **EC2 domain-join** | Managed Microsoft AD or AD Connector |
| **SAML federation** | Managed Microsoft AD with AD FS, or AD Connector proxying to on-prem AD FS |

---

## 5. AD Connector — Important Caveats

AD Connector is purely a **proxy**. Every authentication call — EC2 domain join, WorkSpaces
login, IAM Identity Center sign-in — travels back to on-prem AD in real time.

⚠️ If your Direct Connect link or VPN goes down, **all AD-backed authentication in AWS fails**.
Plan for redundancy: two AD Connectors in different Availability Zones, each pointing at
different on-prem DCs.

---

## 6. Common Mistakes

- ❌ Using AD Connector without an on-prem AD — it has no directory of its own; it will not
  function.
- ❌ Choosing Simple AD when you need trust relationships with on-prem AD — Simple AD does not
  support AD trusts.
- ⚠️ Expecting Managed Microsoft AD to automatically sync with on-prem — a **trust** must be
  configured; it is not a sync; both directories remain independent.
- ⚠️ Underestimating AD Connector latency — every auth round-trip hits on-prem; high latency or
  network outages surface as login failures.

---

## Key Exam Points

- **AD Connector** = proxy to on-prem; nothing stored in AWS; on-prem AD is authoritative.
- **Simple AD** = Samba-based, standalone in AWS, no on-prem trust, small scale (≤ 5 000 users).
- **Managed Microsoft AD** = real Microsoft AD in AWS; optional trust with on-prem; supports GPO,
  schema extensions, all AD-integrated apps.
- "Company has on-prem AD and wants AWS apps to authenticate against it without storing
  credentials in AWS" → **AD Connector**.
- "Company needs full Microsoft AD features (GPO, trusts, domain-join) in AWS" → **Managed
  Microsoft AD**.
- "Small company, no on-prem AD, just needs basic directory" → **Simple AD**.

---

**Next**: [Networking Primer](../03_networking/01_networking_primer.md)
