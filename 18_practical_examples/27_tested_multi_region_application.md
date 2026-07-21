# Professional Build: A Tested Multi-Region Application

> **Scenario**: A customer API must recover from a Regional outage with an RTO of 15 minutes and
> a committed-business-operation RPO of zero. It uses API Gateway, Lambda, DynamoDB, S3 objects and
> asynchronous events. That RPO forces synchronous durability and a durable outbox; ordinary
> asynchronous S3 replication and Regional SQS alone cannot meet it.
> The business accepts active/passive compute, but recovery must not depend on operators creating
> infrastructure in the failed Region.

Replication is only one part of recovery. The design needs a reachable front door, pre-deployed
compute, explicit write authority, replicated configuration and keys, asynchronous reconciliation,
capacity, operator access and a repeatedly measured failback path.

---

## 1. Steady-State Architecture

```
                         Route 53 failover/latency aliases + ARC controls
                                      │
                     ┌────────────────┴────────────────┐
                     ▼                                 ▼
             Region A — active                 Region B — passive-ready
          API Gateway + Lambda                 API Gateway + Lambda
                    │                                  │
                    └──── DynamoDB MRSC global table ──┘── Region C witness
                      dual-Region durable S3 objects
                   MRSC outbox → Regional queues/consumers

     independent regional logs/alarms ──▶ central operations and durable evidence
```

Deploy the same immutable application artifact and IaC version to both Regions. Keep the passive
API synthetically tested but out of normal writes. Pre-create custom domains/certificates, IAM
roles, WAF, queues, event-source mappings, dashboards, alarms and enough concurrency/quota/capacity
to meet the RTO. A failover pipeline stored only in Region A is not a recovery mechanism.

This scenario uses a **multi-Region strong consistency (MRSC)** DynamoDB global table with two
replica Regions plus a witness in a supported third Region. It provides zero-RPO strongly consistent
state at the cost of higher write latency, topology/Region constraints and cross-Region coupling.
If those constraints do not fit, use MREC and publish the **measured** business-state RPO instead of
claiming zero. The partition-key design must still distribute load.

S3 CRR is asynchronous, and even S3 Replication Time Control has a 15-minute SLA, so it cannot prove
this scenario's zero-RPO acknowledgement. The write path puts an immutable, checksum-addressed
object in both Regional buckets before committing its MRSC manifest and acknowledging the client;
CRR remains a repair/backfill control. If either durable copy is unavailable, the API rejects or
holds the uncommitted request—the explicit availability tradeoff for the RPO. Preserve versioned
backups because corruption/deletes can replicate. Replicate or independently provision Secrets
Manager secrets/config. A KMS
multi-Region key shares key material and ID with replicas, but policies, grants, aliases and enabled
state remain Regional and must be deployed/tested in both Regions.

---

## 2. Application Failure Semantics

- Give each request an idempotency key stored with a conditional DynamoDB write. API Gateway/Lambda
  retries, client retries and failover must return the stored result rather than repeat the effect.
- Store the business state and pending outbox payload in **one MRSC item**. Regional publishers
  place that payload on SQS and mark the item published idempotently. After failover, Region B finds
  and republishes pending items. SQS remains a delivery buffer, not the RPO source of truth. Do not
  assume a multi-item DynamoDB transaction is atomically observed across global-table Regions.
- Bound Lambda timeouts below caller timeouts, use reserved concurrency to isolate critical paths,
  and coordinate API throttles, concurrency and DynamoDB capacity. Alarm on throttles at every layer.
- Emit a correlation ID, Region and deployment version in structured logs/traces and API responses.
  Use external or cross-Region canaries so a failed Region cannot declare itself healthy.

Health checks should measure a safe end-to-end dependency, not only an API Gateway `200`. Fully
automatic failover risks moving traffic during a dependency or deployment fault shared by both
Regions. Use Route 53 aliases for the Regional API Gateway custom domains, with ARC routing controls
and safety rules for guarded movement. Global Accelerator cannot use API Gateway directly as a
standard endpoint; it is an alternative only if the Regional front door is a supported resource
such as an ALB, NLB, EC2 instance or Elastic IP.

### Enforce one write epoch

Store `{activeRegion, epoch, state}` in an MRSC control item. **MRSC does not support DynamoDB
transactional read/write APIs**, so it cannot atomically condition a separate business item on that
control item. Every handler strongly reads the control immediately before one conditional write of
the combined business/outbox item:

```text
control = strongly_consistent_read("CONTROL#writer")
reject unless control.state == "ACTIVE"
              and request.region == control.activeRegion

conditional_put_one_item(
  business state + idempotency key + pending outbox payload
  + observed writer epoch,
  condition: this request id has not already completed
)
```

There is still a race after a handler reads the control. Failover first conditionally changes the
MRSC control to `FENCING` and increments `epoch`, then disables Region-A ingress/event sources and
waits until in-flight executions reach zero. The wait is longer than the explicitly bounded API,
Lambda and SDK retry lifetime. Only then does it set Region B to `ACTIVE` and enable its writers.
If the business requires instantaneous, zero-overlap fencing without this bounded drain, this
DynamoDB design cannot provide it; choose a write architecture with an atomic external lease.

---

## 3. Executable Failover Runbook

1. **Declare and drain.** Name an incident commander, pause deployments, conditionally set the MRSC
   control to `FENCING`, disable Region-A ingress/event sources, and wait past the bounded retry/
   execution lifetime with no in-flight writers. Then set Region B to `ACTIVE` with a new epoch.
2. **Validate Region B.** Confirm its synthetic result, quotas/concurrency, configuration/secrets,
   KMS state, DynamoDB replica/witness state, throttles/errors/capacity and a strong read/write probe,
   plus S3 replication failures and queue consumers. MRSC has no `ReplicationLatency` metric and no
   MREC last-writer-wins conflict-resolution path.
3. **Enable dependencies in order.** Scale/enable consumers and write Lambdas, then run internal
   read and controlled idempotent-write tests. Record the newest consistent business transaction.
4. **Shift traffic.** Change the approved ARC routing control/Route 53 failover state.
   Move a small percentage first when supported; observe errors, latency, throttles and correctness.
5. **Reconcile.** Replay or compensate stranded asynchronous work and compare counts/business
   aggregates. Publish the measured RTO and RPO, including any accepted loss—not the design target.
6. **Stabilize.** Keep Region A fenced, preserve evidence and increase Region-B capacity if actual
   demand or a concurrent AZ failure requires it.

Abort the shift if Region B fails its write test, lacks required secrets/keys/capacity, or produces
inconsistent data. The rollback is to the last known single writer only when that Region is healthy;
never enable two writers merely to make the health dashboard green.

---

## 4. Failback and Repeated Proof

Failback is a planned migration. Repair and redeploy Region A, confirm data/object/config
convergence, keep one write authority, shift a canary, then move traffic while monitoring. Reconcile
queues and pending outbox/object work before removing extra Region-B capacity.

Run quarterly and after material dependency changes:

| Test | Evidence |
|------|----------|
| Disable one Lambda dependency or exhaust reserved concurrency | Back-pressure, alarms, idempotent retry and rollback behave as designed |
| Block/impair one AZ dependency | Regional service continues within SLO and no hidden zonal singleton appears |
| Controlled Regional game day | Timed fencing, promotion/enablement, traffic shift, client reconnection and capacity checks |
| Data integrity test | MRSC combined business/outbox item IDs, dual-bucket checksums and business aggregates prove the stated RPO |
| Failback exercise | Single-writer control, reverse convergence and gradual traffic return complete safely |

Track actual RTO/RPO, failover step duration, human approvals, DNS/client convergence, error-budget
burn, replication lag, queue age, conflict/reconciliation count and recovery cost. Feed every missed
dependency or slow manual step into IaC, alarms and the next exercise.

Active/passive duplicates core infrastructure and data transfer but costs less than two fully loaded
Regions. Active/active is justified only when latency/availability value exceeds its conflict,
deployment and operational complexity. Backup/restore or pilot light is better when the stated RTO
allows infrastructure/data restoration.

**Related**: [Serverless API](21_api_gateway_lambda_dynamodb.md) ·
[DR strategies](19_disaster_recovery_strategies.md) · [Disaster recovery concepts](../14_hybrid_migration_dr/04_disaster_recovery.md)
