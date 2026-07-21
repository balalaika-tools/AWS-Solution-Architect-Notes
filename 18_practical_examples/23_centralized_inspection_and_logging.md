# Professional Build: Centralized Inspection and Logging

> **Scenario**: Dozens of workload accounts need segmented east-west connectivity, controlled
> internet egress, hybrid access, centralized threat controls and evidence that a packet was
> inspected. Product VPCs must not become dependent on one untested appliance or one AZ.

Centralization reduces duplicated policy and operations, but it increases **blast radius and data
processing cost**. Separate the traffic architecture from the log architecture, give each an owner,
and design an explicit degraded mode.

---

## 1. Reference Architecture

```
                    Network account
on-prem ─ DX/VPN ─┐
                  ├─ Transit Gateway ─ inspection VPC ─ same TGW ─ egress VPC/IGW
workload VPCs ────┘        │               │
  prod/dev route tables    │        AWS Network Firewall
  remain segmented         │        or GWLB appliance fleet
                           ▼
                    central DNS endpoints

workload/security telemetry ──▶ Log Archive account (immutable evidence)
                              └▶ Security Tooling account (search/detect/respond)
```

Use separate TGW route tables for production, nonproduction, shared services, hybrid and inspection
attachments. Association decides which table an attachment **uses**; propagation decides where its
routes **appear**. Do not propagate every attachment into one flat table.

### Traffic paths

- **Egress:** spoke subnet → TGW → inspection attachment → firewall endpoint in the same AZ where
  possible → TGW → egress attachment → NAT/IGW → internet. Return traffic traverses both TGW legs
  and the same stateful firewall endpoint in reverse. A combined inspection/egress VPC is a valid
  simpler alternative, but do not draw separate VPCs and then omit their connecting routes.
- **East-west:** route only approved spoke-to-spoke prefixes through inspection. PrivateLink is
  usually safer for exposing one service than granting broad network reachability.
- **Hybrid:** advertise summarized, nonoverlapping prefixes through DX/VPN. Keep redundant customer
  routers/links and decide whether on-prem traffic is inspected in AWS or before entry—not twice by
  accident.
- **Inbound:** choose CloudFront/WAF, Global Accelerator, ALB/NLB and firewall insertion based on
  protocol and threat model. Do not force every application through a generic ingress appliance if
  the managed edge control already satisfies the requirement.

For stateful appliances behind GWLB or in an inspection VPC, enable the appropriate TGW appliance
mode and build AZ-aware routes. Verify symmetry; a forward flow through AZ-a and return through AZ-b
can be dropped even when every SG and NACL is open.

### Route-table artifact for the separate-VPC topology

Use exact workload prefixes in production; the broad destinations below emphasize path ownership.

| Route table / association | Destination | Next hop |
|---------------------------|-------------|----------|
| Spoke private subnet | `0.0.0.0/0` | TGW |
| TGW spoke-ingress table | `0.0.0.0/0` | Inspection VPC attachment |
| Inspection TGW-attachment subnet | `0.0.0.0/0` | Same-AZ firewall endpoint |
| Firewall subnet after inspection | `0.0.0.0/0` | TGW |
| TGW inspection-ingress table | `0.0.0.0/0` | Egress VPC attachment |
| Egress TGW-attachment subnet | `0.0.0.0/0` | Same-AZ NAT gateway |
| Egress public subnet | `0.0.0.0/0` | IGW |
| Egress public/NAT subnet | Spoke CIDRs | TGW |
| TGW egress-ingress table | Spoke CIDRs | Inspection VPC attachment |
| Inspection return path after firewall | Spoke CIDRs | TGW/spoke attachment |

The inspection attachment uses appliance mode and has subnets in every participating AZ. Validate
that propagation does not introduce a more-specific bypass and that both the firewall and NAT
subnet route tables contain the reverse path.

---

## 2. Central Evidence and Operations

| Evidence | Destination and purpose |
|----------|-------------------------|
| Organization CloudTrail | Log Archive S3 for durable API evidence; CloudWatch ingestion/pipelines and Logs Insights QL/SQL/PPL for current analytics. CloudTrail Lake is an existing-customer-only path after its 2026 new-customer availability change. |
| VPC/TGW/Resolver/Network Firewall logs | Central log destinations with account, Region, VPC and rule metadata; use scoped retention and queries |
| CloudWatch application metrics/logs | OAM links or central copies/subscriptions for live operations; workload teams retain local access |
| AWS Config | Organization aggregator and conformance packs in a delegated account; immutable history retained separately as required |
| Findings | Security Hub/GuardDuty/Inspector/Macie delegated administrator for prioritization and response |

KMS and destination policies must authorize each AWS service and source organization/account while
denying unintended writers/readers. Keep the Security Tooling account able to search and respond,
but make the Log Archive account difficult for it to erase. Alert on gaps in delivery, key disable or
deletion scheduling, bucket-policy changes, retention changes and delegated-administrator changes.

Log only what has an owner and a use: CloudTrail management events are a baseline; high-volume data
events, Flow Logs and DNS queries should be scoped according to detection, forensic and compliance
requirements. Validate field availability before promising a detection.

---

## 3. Deployment and Failure Tests

1. Allocate nonoverlapping CIDRs with IPAM and reserve headroom for attachments, firewall endpoints
   and recovery Regions. Check TGW routes/attachments, firewall capacity and endpoint quotas.
2. Build Network and Log Archive accounts before spokes. Deploy inspection endpoints across the AZs
   used by workload attachments and pin routes to avoid unnecessary cross-AZ charges.
3. Attach one nonproduction VPC. Validate allowed and denied synthetic flows in both directions,
   route-table evidence, firewall logs, Flow Logs and application telemetry.
4. Test loss of one firewall endpoint/AZ, a route withdrawal, a full NAT path and a log destination.
   Decide in advance whether each traffic class **fails closed**, uses a healthy inspected path, or
   accepts a documented bypass. Silent direct-to-internet fallback is not acceptable.
5. Add production attachments in waves with a reversible route change and a comparison of latency,
   error rate and cost before/after.

Troubleshoot a timeout in this order: DNS answer → source subnet route → TGW association/propagation
→ inspection route and state → destination route → SG/NACL → application listener. Reachability
Analyzer models supported AWS paths but does not prove external routers or live appliance health;
correlate it with Flow Logs, firewall logs and packet/application tests.

---

## 4. Decisions and Tradeoffs

Distributed inspection offers team isolation and shorter paths but duplicates policy and cost.
Central inspection improves consistency and specialist ownership but creates a shared dependency,
TGW/firewall/NAT processing charges and possibly cross-AZ transfer. Cloud WAN becomes attractive for
policy-driven multi-Region/global segmentation; TGW is often simpler for a smaller Regional hub.

Measure accepted/rejected traffic, endpoint health/capacity, bytes and cost per path, inter-AZ
transfer, route-change failure rate, log delivery delay and detection/response time. The architecture
is successful when controls stay observable during failure—not merely when the diagram is tidy.

**Related**: [Transit Gateway](../03_networking/06_transit_gateway_and_advanced.md) ·
[Monitoring](../12_monitoring/README.md) · [Landing zone](22_control_tower_landing_zone.md)
