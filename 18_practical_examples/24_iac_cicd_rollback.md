# Professional Build: IaC Delivery with Safe Rollback

> **Scenario**: A platform team deploys shared network and workload stacks across development,
> staging and production accounts. Every change needs review, policy checks, staged rollout,
> immutable evidence and a recovery path that also works for data/schema changes.

Infrastructure as code makes changes repeatable; it does not make them safe automatically. A safe
pipeline separates **artifact creation**, **change evaluation**, **deployment**, **verification** and
**rollback**, with different trust roles at each boundary.

---

## 1. Delivery Flow

```
signed source commit
   └─▶ lint/unit/schema tests
       └─▶ synthesize immutable template/artifact + SBOM/checksums
           └─▶ security/policy/cost checks
               └─▶ deploy ephemeral/test account
                   └─▶ integration + failure tests
                       └─▶ change set and approval
                           └─▶ canary account/Region
                               └─▶ production waves
                                   └─▶ post-deploy SLO and drift checks
```

Store artifacts in a versioned, encrypted artifact account and promote the **same digest** between
environments. The pipeline assumes a narrowly scoped deployment role in each target account; target
accounts do not trust arbitrary branches, personal principals or an unrestricted tooling account.
Record source revision, template/artifact digest, approver, change set, execution and test evidence.

Use CloudFormation StackSets for centrally managed, repeated stacks across accounts/Regions. Use
Service Catalog when teams need governed self-service products with approved parameters. These
tools complement each other; neither is a reason to put application and organization baselines in
one failure-prone stack.

### Enforce the source-to-production trust chain

The source role below can be assumed only by the `main` branch of one GitHub repository. In a full
pipeline it builds and signs the artifact; a separate release-controller role, reachable only after
the approval gate, assumes the target account's deployment role. The target role trusts that exact
controller ARN—not the whole tooling account and not GitHub directly.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::111122223333:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:acme/platform-infra:ref:refs/heads/main"
      }
    }
  }]
}
```

Upload the synthesized template under its SHA-256 digest and record the S3 `VersionId`. Create the
CloudFormation change set from that versioned URL, not from a mutable branch path. Execute it with a
CloudWatch rollback trigger and monitoring window. Protect stateful resources explicitly:

```yaml
OrdersTable:
  Type: AWS::DynamoDB::Table
  DeletionPolicy: Retain
  UpdateReplacePolicy: Retain
  Properties:
    BillingMode: PAY_PER_REQUEST
    AttributeDefinitions: [{AttributeName: pk, AttributeType: S}]
    KeySchema: [{AttributeName: pk, KeyType: HASH}]
```

`Retain` prevents stack rollback from deleting the table; it does not roll its data backward or
assign an owner to an orphan. The release runbook records retained resources and either imports,
reconciles or removes them after review.

---

## 2. Test and Policy Gates

Before deployment, fail the pipeline on malformed templates, known secret patterns, disallowed
public exposure, missing encryption/logging/ownership tags, IAM wildcards outside an approved rule,
unsupported Regions and quota assumptions without headroom. Generate a change set and classify:

- replacement or deletion of a stateful resource;
- IAM, KMS, organization, route, DNS or security-boundary changes;
- expected downtime and rollback behavior;
- new recurring and data-transfer cost;
- database/schema compatibility and application-version ordering.

After deployment, test actual behavior: TLS and authorization, synthetic transactions, alarms/logs,
backup registration, failure response and an SLO comparison. CloudFormation success only proves that
the resource APIs accepted the desired state.

---

## 3. Release and Rollback Strategy

Choose the smallest blast radius that can prove the change:

1. deploy to an ephemeral or dedicated integration account;
2. deploy one canary account/AZ/Region and hold for a measured observation period;
3. expand in production waves with concurrency and failure tolerance below the business limit;
4. stop automatically on stack failure, synthetic failure, error-budget burn, capacity exhaustion or
   a security regression.

CloudFormation can roll back failed resource changes, but **rollback is not universal**. A database
replacement, destructive custom resource, externally consumed DNS change or forward-only schema
migration may not be reversible. Protect important data with deletion/update-replace policies and
backups, but do not retain abandoned resources indefinitely without ownership.

For application plus schema releases, use **expand/contract**:

- add backward-compatible schema and deploy code that can use old or new forms;
- migrate/backfill with checkpoints and observability;
- shift traffic with a canary or blue/green mechanism;
- remove old schema only after the rollback window and all old consumers are gone.

Rollback means redeploying a known-good artifact and restoring traffic/configuration. If data has
accepted incompatible new writes, use a rehearsed forward fix or reconciliation path; blindly
restoring an old snapshot can discard valid business transactions.

---

## 4. Drift, Break Glass, and Recovery

Run scheduled CloudFormation drift detection for supported resources and AWS Config for compliance
across managed and unmanaged resources. Decide whether detected drift is imported into code,
reverted automatically, or granted a time-bound exception. Never let a pipeline overwrite an
incident responder's emergency change before ownership is established.

The break-glass role needs strong MFA, short sessions, alerting and a follow-up requirement to
reconcile code. Keep source, artifacts, KMS access and deploy roles recoverable if the primary CI/CD
Region or tooling account is unavailable; otherwise a multi-Region workload has a single-Region
control plane.

### Acceptance evidence

- An untrusted branch cannot assume a production deploy role or alter the artifact.
- A dangerous replacement is visible and blocked before approval.
- A failed canary stops later waves and restores the previous version within the stated objective.
- A manual drift is detected, attributed and reconciled.
- Deployment duration, failure rate, rollback time, SLO impact and unplanned drift trend downward.

**Related**: [Deployment comparisons](../17_exam_patterns/02_service_comparison_cheatsheets.md) ·
[Config remediation](26_config_automated_remediation.md) · [Container delivery](../09_containers/README.md)
