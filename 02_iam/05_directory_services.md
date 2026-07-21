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
    └── Existing small workload, basic LDAP + Kerberos?          → Simple AD (where available)
        New architecture?                                       → AWS Managed Microsoft AD
```

---

## 4. Integration Points

| AWS Service | Works with |
|-------------|-----------|
| **IAM Identity Center** | AWS Managed Microsoft AD or AD Connector; **not Simple AD** |
| **WorkSpaces** | AWS Managed Microsoft AD, AD Connector, or Simple AD |
| **AppStream 2.0** | AWS Managed Microsoft AD or AD Connector (domain-join required) |
| **RDS (SQL Server, Oracle)** | Managed Microsoft AD (Windows Authentication) |
| **EC2 domain-join** | AWS Managed Microsoft AD, AD Connector, or Simple AD |
| **SAML federation** | IAM Identity Center or IAM trusts the SAML IdP (such as AD FS); AD Connector is not a SAML proxy |

---

## 5. AD Connector — Important Caveats

AD Connector is purely a **proxy**. Directory requests for EC2 domain join, WorkSpaces login,
and IAM Identity Center sign-in travel back to on-premises AD in real time.

⚠️ If the Direct Connect or VPN path goes down, new authentication and directory operations
through that connector can fail. An AD Connector already places managed endpoints in two
Availability Zones, but that protects only the connector tier. Use redundant Direct
Connect/VPN paths, reachable DNS servers and domain controllers, correct AD Sites/subnets, and
open DNS/Kerberos/LDAP ports from both directory subnets. Two healthy connector endpoints do
not compensate for one failed WAN path or an unavailable on-premises domain.

---

## 6. Hybrid Identity and Migration Scenarios

The deciding question is not simply "Do we already have AD?" Decide **where identities remain
authoritative**, **whether AWS workloads must survive loss of on-premises connectivity**, and
**whether applications need an actual AD domain in AWS**.

### 6.1 Keep on-premises AD authoritative

Choose **AD Connector** when AWS services only need to validate existing on-premises users and
you explicitly accept a real-time dependency on the corporate network, DNS, and domain
controllers. It is the smallest change because it stores no directory data in AWS and creates
no second forest to administer.

This is a poor fit for an isolated or intermittently connected Region. A connector outage in
one Availability Zone is handled by its other endpoint, but a WAN, firewall, DNS, service
account, or on-premises AD failure interrupts new authentication and directory operations.
Existing application sessions may continue until their own tokens or tickets expire; do not
treat that as a recovery strategy.

### 6.2 Run an AWS-resident resource forest and trust on-premises AD

Choose **AWS Managed Microsoft AD** when migrated applications require real Microsoft AD
features in AWS: domain join, GPOs, schema extensions, LDAP/Kerberos, SQL Server integration, or
an AWS-local directory service. Configure a one-way or two-way **trust** to the self-managed
forest according to which side's users need which side's resources.

A trust is **not synchronization or migration**. User objects and passwords remain in their
original directory; DNS conditional forwarding, network routes, firewall ports, and domain
controllers on both sides must remain reachable for cross-forest authentication. A trust lets
on-premises users reach AWS-domain resources, but loss of the on-premises forest still affects
those users. AWS Managed AD users and AWS-local applications remove that dependency only after
the relevant identities and workloads are migrated.

Use selective authentication and narrow group assignments instead of trusting every user to
every server. Diagnose trust failures by checking both trust directions, the shared trust
password, DNS conditional forwarders, routes, security groups/NACLs, and AD ports before
recreating the directory.

### 6.3 Migrate off self-managed domain controllers

An **AD Connector is not a migration destination**; there is no AWS-side directory into which
users can move. For a phased migration, deploy AWS Managed Microsoft AD, establish a trust,
migrate users/groups/service accounts and application dependencies with appropriate Microsoft
migration tooling, then cut applications over before retiring the old forest.

Inventory hard dependencies first: service-account names, SPNs, LDAP distinguished names,
GPOs, certificate enrollment, DNS zones, and hard-coded domain controller addresses. A trust
can make the transition coexist, but it does not rewrite any of these dependencies.

### 6.4 Centralize workforce access without moving application AD

**IAM Identity Center** needs one identity source for the organization:

- its built-in Identity Center directory;
- an external SAML IdP, commonly provisioned with SCIM; or
- Active Directory through AWS Managed Microsoft AD or AD Connector.

If the company already has a modern workforce IdP and only needs AWS account/application SSO,
connect that IdP directly. Do not deploy Directory Service solely to preserve AD terminology.
Use the AD identity source when AD groups are authoritative or AWS-integrated applications
already need the directory.

For an Active Directory identity source, the AWS Managed AD or AD Connector directory must be
owned by the Organizations management account, or by the IAM Identity Center delegated
administrator account when delegation is configured. Configure IAM Identity Center in a Region
where the directory exists. Changing identity-source type can remove synchronized users,
groups, assignments, and provisioned permission-set roles, so preserve break-glass access and
plan the cutover instead of treating it as a toggle.

### 6.5 Multiple accounts and Regions

Directory placement is an architecture decision:

- **Across accounts**: share AWS Managed Microsoft AD with workload accounts through Directory
  Sharing rather than deploying a forest per account. Sharing is Regional and requires network
  connectivity from consumer VPCs. Organizations-based directory sharing requires the
  directory owner to be the management account; otherwise use the supported handshake method.
- **Across Regions**: Enterprise Edition AWS Managed Microsoft AD supports native multi-Region
  replication. Applications use local domain controllers while directory data replicates.
  Standard Edition, AD Connector, and Simple AD do not gain this behavior automatically.
- **AD Connector in another Region**: deploy a Regional connector with redundant network paths
  to reachable self-managed domain controllers. A connector in Region A is not a failover
  endpoint for Region B.
- **Trusts**: a trust configured for a multi-Region Managed AD directory applies across its
  replicas, but authentication to identities in the trusted self-managed forest still needs
  working DNS and network reachability.

Separate the failure domains in a design review: directory endpoints, directory data, network
paths, DNS, the authoritative identity store, the IAM Identity Center home Region, and each
consumer account/VPC. A "multi-AZ directory" alone does not cover the remaining dependencies.

---

## 7. Simple AD Availability for New Customers

Simple AD remains useful for small existing standalone deployments, but AWS has announced that
it will close to **new customers on July 30, 2026**; existing customers retain functionality.
For a new architecture, choose AWS Managed Microsoft AD when you need an AWS-resident directory,
or AD Connector when a reachable self-managed AD remains authoritative.

---

## 8. Common Mistakes

- ❌ Using AD Connector without an on-prem AD — it has no directory of its own; it will not
  function.
- ❌ Choosing Simple AD when you need trust relationships with on-prem AD — Simple AD does not
  support AD trusts.
- ⚠️ Expecting Managed Microsoft AD to automatically sync with on-prem — a **trust** must be
  configured; it is not a sync; both directories remain independent.
- ⚠️ Underestimating AD Connector latency — every auth round-trip hits on-prem; high latency or
  network outages surface as login failures.
- ⚠️ Assuming the two-AZ connector deployment removes the WAN, DNS, or on-premises domain
  controller dependency.
- ⚠️ Treating a trust as replication, or treating IAM Identity Center account assignments as a
  multi-Region directory design.
- ⚠️ Connecting Simple AD to IAM Identity Center—it is not a supported identity source.

---

## Key Exam Points

- **AD Connector** = proxy to on-prem; nothing stored in AWS; on-prem AD is authoritative.
- **Simple AD** = Samba-based, standalone in AWS, no on-prem trust, small scale (≤ 5 000 users).
- **Managed Microsoft AD** = real Microsoft AD in AWS; optional trust with on-prem; supports GPO,
  schema extensions, all AD-integrated apps.
- **Trust** = cross-directory authentication, not object/password synchronization. **Directory
  Sharing** extends one Managed AD directory across accounts; Enterprise Edition multi-Region
  replication extends it across supported Regions.
- IAM Identity Center can use its own directory, an external IdP, AWS Managed Microsoft AD, or
  AD Connector. Choose the identity source separately from the directory required by workloads.
- "Company has on-prem AD and wants AWS apps to authenticate against it without storing
  credentials in AWS" → **AD Connector**.
- "Company needs full Microsoft AD features (GPO, trusts, domain-join) in AWS" → **Managed
  Microsoft AD**.
- "Small existing standalone deployment needs only basic directory features" → **Simple AD**
  may fit. For a new architecture, select AWS Managed Microsoft AD because Simple AD closes to
  new customers on July 30, 2026.

---

**Next**: [Networking Primer](../03_networking/01_networking_primer.md)
