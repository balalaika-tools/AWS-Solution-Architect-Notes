# Professional Build: A Migration-Wave Cutover

> **Scenario**: A customer portal runs on 18 VMware servers, Oracle, Active Directory and an
> on-premises file share. The target rehosts the application on EC2, replatforms Oracle to Aurora
> PostgreSQL, and moves files to FSx or S3. The business allows a 45-minute write outage and demands
> a tested rollback decision before customers create data in the new system.

A migration wave is a **dependency and business unit of change**, not a spreadsheet batch of
servers. Network, identity, data, application, monitoring and support must cross the finish line
together or remain connected through a deliberately temporary hybrid design.

---

## 1. Assess and Select the Wave

Collect inventory and dependency evidence with the CMDB, owners and the **AWS Transform Discovery
Tool**. Application Discovery Service stopped accepting new customers in November 2025, so use it
only for an existing in-flight project. Build current assessments in AWS Transform; an existing
Migration Evaluator Quick Insights export can be imported as an input. Capture utilization,
licenses, support status, criticality, RTO/RPO, data classification, peak calendar,
inbound/outbound calls and named owners. The cost model includes parallel run, network, license,
training and migration-factory costs rather than treating a tool recommendation as the decision.

Apply the seven migration strategies per application:

| Component | Strategy and target | Why |
|-----------|---------------------|-----|
| Web/app VMs | Rehost with Application Migration Service (MGN), then modernize after stabilization | Meets the window without combining an application rewrite with a platform move |
| Oracle database | Replatform using DMS Schema Conversion where supported, with AWS SCT as the desktop fallback for unsupported source/target features; use DMS full load + CDC for data | Removes license/operations burden, but only after incompatible code and performance are proven |
| File share | Replatform using DataSync to the file/object target selected from access semantics | Incremental sync reduces final cutover volume; ACL/metadata behavior must be tested |
| Unused report server | Retire after owner and retention approval | Avoids paying to migrate waste |
| Tightly coupled unsupported appliance | Retain temporarily | Its dependency and exit date remain explicit rather than hiding migration risk |

Place components in the same wave when latency or consistency makes a long hybrid phase unsafe.
Start with a lower-criticality, representative application to validate the landing zone and factory;
do not choose a trivial pilot that exercises none of the hard dependencies.

---

## 2. Entry Criteria and Rehearsal

The wave cannot enter production cutover until:

- accounts, subnets, nonoverlapping CIDRs, DX/VPN, Resolver rules, time synchronization and identity
  are tested from the same target paths the application will use;
- target IAM, SG/NACL, KMS, secrets, patching, vulnerability scanning, central logs, alarms, backups
  and restore tests meet the security baseline;
- MGN replication is healthy and a test launch passes boot, service, dependency and performance
  tests in an isolated subnet;
- DMS Schema Conversion/SCT exceptions are resolved, DMS full load + CDC is stable, validation/business reconciliation
  passes and source log retention covers a disruption;
- DataSync incremental runs meet metadata, integrity and duration requirements;
- DNS TTL is lowered early enough to affect cached answers, certificates are deployed, quotas and
  target capacity include failure headroom, and the support rota can assume its roles;
- a rehearsal records timings below the 45-minute window with go/no-go, abort, rollback and owner
  checkpoints.

Freeze unrelated infrastructure and schema changes during the final rehearsal/cutover period. Keep
the source protected; MGN and DMS replication do not create a general reverse-write rollback path.

### Checkpoint commands before the bridge opens

Use the actual ARNs and Region from the approved wave record. These read-only calls give the bridge
timestamped service evidence instead of a verbal “replication looks good”:

```bash
aws mgn describe-source-servers --region eu-west-1
aws dms describe-replication-tasks --region eu-west-1
aws dms describe-table-statistics --replication-task-arn <task-arn> --region eu-west-1
aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CDCLatencySource \
  --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> \
               Name=ReplicationTaskIdentifier,Value=<task-id> \
  --statistics Maximum --period 60 --start-time <iso-start> --end-time <iso-end> \
  --region eu-west-1
aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CDCLatencyTarget \
  --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> \
               Name=ReplicationTaskIdentifier,Value=<task-id> \
  --statistics Maximum --period 60 --start-time <iso-start> --end-time <iso-end> \
  --region eu-west-1
aws datasync describe-task-execution --task-execution-arn <execution-arn> --region eu-west-1
```

Archive the filtered source-server lag, DMS task status/table validation, both CDC latency metrics,
and DataSync bytes/files/errors with the cutover decision. Compare `CDCLatencySource` and
`CDCLatencyTarget` with the approved cutover threshold; use the equivalent prebuilt dashboard for a
DMS Serverless deployment. Do not paste secrets or the whole inventory into the incident channel.

---

## 3. Timestamped Cutover Runbook

| Time | Action and evidence | Decision owner |
|------|---------------------|----------------|
| T-60 | Confirm incident bridge, dashboards, replication/validation, target capacity, backups and no conflicting changes | Migration lead |
| T-45 | Drain sessions; stop schedulers and partner feeds; place application in maintenance/read-only mode | App owner |
| T-40 | Record source transaction/file checkpoints; wait for DMS CDC and final DataSync to reach agreed lag; reconcile counts and business totals | Data owner |
| T-25 | Launch/activate final EC2 targets; apply versioned config/secrets; run internal health and dependency tests | Platform lead |
| T-15 | Switch controlled DNS/load-balancer/API configuration and partner endpoints; verify resolver behavior from representative clients | Network lead |
| T-10 | Run authentication, read and controlled write synthetic transactions; verify logs, traces, alarms and downstream processing | Test lead |
| T-5 | **Go/no-go:** if any mandatory check fails, restore old routing and source writers before new production writes are accepted | Business owner |
| T+0 | Open traffic gradually; monitor errors, latency, saturation, replication queues, data correctness and support tickets | Operations lead |

The rollback deadline is tied to **data divergence**, not the wall clock alone. Before new target
writes, restore DNS/routing and source writers, then verify the source. After incompatible target
writes, rollback requires a rehearsed reverse replication or reconciliation process; otherwise the
safer choice may be a forward fix. State that constraint during approval.

---

## 4. Stabilization and Exit

For the agreed hypercare period, compare target SLOs, batch completion, data aggregates, security
findings and unit cost with the baseline. Maintain MGN/DMS resources only as long as the rollback and
evidence plan requires. Then raise DNS TTL, right-size from production measurements, select
commitments only after the modernization roadmap is stable, patch/backup the new targets, close
temporary firewall rules, archive evidence and decommission sources through an approved retention
process.

Run a retrospective on missed dependencies, manual steps, actual downtime and cost. Feed reusable
fixes into the landing zone, discovery questionnaire, IaC modules and later waves. A wave succeeds
when it improves the migration factory, not only when its servers boot in AWS.

**Related**: [Migration Services](../14_hybrid_migration_dr/02_migration_services.md) ·
[Landing zone](22_control_tower_landing_zone.md) · [DR exercise](27_tested_multi_region_application.md)
