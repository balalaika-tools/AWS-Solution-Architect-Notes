# Professional Build: Organization-Wide Config Remediation

> **Scenario**: Security requires that production security groups never expose SSH to
> `0.0.0.0/0` or `::/0`. The organization wants centralized detection and fast repair without an
> automation that can silently remove approved access or lock responders out during an incident.

AWS Config answers **whether recorded resource configuration complies with a rule**. Systems
Manager Automation can perform a bounded repair. EventBridge, CloudWatch and ticketing provide
workflow and evidence. Treat these as separate failure domains: a compliant dashboard is not proof
that recording or remediation is healthy.

---

## 1. Organization Design

```
Organizations
  └─ delegated Config administrator (Security Tooling account)
       ├─ organization aggregator: read-only query/compliance visibility
       ├─ organization conformance pack ─▶ rules/remediation in member accounts/Regions
       └─ central EventBridge bus ◀── forwarded member/Region Config events

member account/Region: Config rule → remediation configuration → local SSM Automation role

Log Archive account ◀── Config snapshots/history + CloudTrail API evidence
```

Enable a Config recorder and delivery channel in every governed account/Region before judging
compliance. Aggregate results centrally, but execute repair through a narrowly scoped member-account
role so the central account does not hold an unrestricted organization-wide mutation role.

An aggregator is **read-only**: it neither deploys rules nor initiates remediation. The organization
conformance pack creates those resources in each governed member account/Region. Forward Config
events from each Region to a central EventBridge bus whose resource policy admits only the intended
organization/accounts; the sender still needs permission to put events to that bus.

Use a conformance pack to distribute the rule and operational metadata. An SCP can block broad
future changes where API semantics allow it, but SCPs do not inspect all configuration state and do
not repair resources; preventive and detective controls complement each other.

---

## 2. Safe Rule and Runbook

The rule evaluates ingress permissions, IPv4 and IPv6, protocol/port ranges and an approved
exception tag or registry. Do not rely on the SG name. Scope automatic remediation to an exact
finding: remove only the public SSH permission that the evaluation identified, leaving unrelated
rules untouched.

Config remediation can dynamically pass the noncompliant resource's `RESOURCE_ID`; it cannot inject
an arbitrary offending permission object from the evaluation result. Therefore the versioned SSM
Automation document should:

1. accept the security-group ID from `RESOURCE_ID` plus Region and rule identity;
2. assume a remediation role limited to describing groups, revoking only ingress, writing evidence
   and updating the approved ticket/status path;
3. call `DescribeSecurityGroupRules`, recompute the exact public SSH rule IDs, and re-read the
   exception registry to prevent time-of-check/time-of-use mistakes;
4. abort if the group or rule no longer matches, ownership is missing, an approved exception is
   active, or the action would violate the emergency-access design;
5. revoke only the recomputed IPv4/IPv6 security-group rule IDs idempotently and verify they are absent;
6. emit before/after configuration, Automation execution ID and CloudTrail event for audit.

Use a human approval step first for production or ambiguous resources. Automatic repair is suitable
only after shadow/detect-only runs show low false positives and a canary OU proves the action. Set
remediation retry, concurrency and error thresholds low enough to stop a bad rule from changing the
entire organization.

> **Better architecture**: administer servers through Systems Manager Session Manager without
> inbound SSH. Then the remediation removes an obsolete exposure instead of disrupting the only
> recovery path.

### Remediation parameter wiring

The member-account template below shows the important mechanism. The Automation document itself
performs the re-query and exact-rule-ID checks described above.

```yaml
PublicSshRemediation:
  Type: AWS::Config::RemediationConfiguration
  Properties:
    ConfigRuleName: restricted-ssh
    TargetId: RemovePublicSshByRuleId
    TargetType: SSM_DOCUMENT
    Automatic: true
    MaximumAutomaticAttempts: 3
    RetryAttemptSeconds: 60
    ExecutionControls:
      SsmControls:
        ConcurrentExecutionRatePercentage: 2
        ErrorPercentage: 1
    Parameters:
      GroupId:
        ResourceValue:
          Value: RESOURCE_ID
      AutomationAssumeRole:
        StaticValue:
          Values:
            - arn:aws:iam::111122223333:role/ConfigSshRemediation
```

In an organization conformance pack, generate or parameterize the member-account role ARN rather
than hard-coding one account ID. If the workflow truly needs the evaluation annotation or a richer
permission object, route the event through a custom EventBridge/Lambda orchestrator that revalidates
current state before starting Automation.

---

## 3. Rollout and Failure Tests

1. Deploy recording and aggregation; alert on stopped recorders, delivery failures and stale
   account/Region coverage.
2. Deploy the rule in detect-only mode to a sandbox OU. Seed compliant, noncompliant, IPv6 and
   exception cases and compare expected results.
3. Run the Automation manually against disposable SGs. Test repeat execution, concurrent rule
   changes, missing permissions, expired exceptions and a denied remediation role.
4. Enable approval-based remediation in nonproduction, then automatic remediation for the narrow
   rule in production canary accounts. Expand in waves.
5. Simulate an EventBridge/ticket destination failure. Detection and repair evidence must remain
   recoverable even when notification is delayed.

If a valid rule is removed, stop automatic remediation, restore only the approved permission from
versioned evidence/IaC, attach a time-bounded exception and correct the evaluator or runbook before
reenabling. Do not disable the Config recorder to hide the false positive.

---

## 4. Success Measures, Cost, and Boundaries

Measure recorder coverage/freshness, evaluation delay, false-positive rate, mean time to remediate,
failed/stuck Automation executions, exception age and recurrence by source pipeline. Config item and
rule-evaluation volume plus Automation steps drive cost; tune recording frequency and rule scope
without creating visibility gaps.

Do not auto-remediate destructive, stateful or ambiguous findings merely because an SSM document
exists. KMS key deletion, route replacement, bucket data removal and database changes usually need
approval, backup and application-aware rollback. The best long-term remediation fixes the IaC or
self-service path that recreated the violation.

**Related**: [AWS Config](../12_monitoring/03_config.md) ·
[IaC delivery](24_iac_cicd_rollback.md) · [Landing zone](22_control_tower_landing_zone.md)
