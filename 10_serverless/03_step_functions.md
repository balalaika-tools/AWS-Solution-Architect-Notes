# AWS Step Functions — Orchestrating Serverless Workflows

> **Who this is for**: Engineers chaining multiple Lambdas (and other services) into a reliable,
> observable workflow. Read [01_lambda.md](01_lambda.md) first — Step Functions mostly coordinates
> Lambdas and service calls.

---

## 1. Prerequisite — Orchestration vs Choreography

When a business process spans several services, there are two ways to coordinate them:

| Approach | How it works | Where the logic lives |
|----------|--------------|-----------------------|
| **Orchestration** | A central controller tells each service what to do and in what order | One place — the orchestrator |
| **Choreography** | Each service reacts to events and emits its own; no central brain | Spread across all services (via SNS/EventBridge/SQS) |

```
ORCHESTRATION (Step Functions)          CHOREOGRAPHY (events)
   ┌────────────┐                          A ──event──▶ B ──event──▶ C
   │ controller │                              (each service decides
   └─────┬──────┘                               its own next move)
     ┌───┼───┐
     ▼   ▼   ▼
     A   B   C   ← controller drives order, retries, branching
```

- **Orchestration** gives you a single, visible flow with built-in retries, branching, and error
  handling — at the cost of a central component. **Step Functions is the orchestrator.**
- **Choreography** is more decoupled and scalable but the overall flow is implicit and harder to
  trace — that's the messaging/events world (SNS, EventBridge — next section).

> **Key insight**: Reach for Step Functions when you need to *see and control* a multi-step
> process — its order, its retries, its branches, and where it failed.

---

## 2. State Machines & Amazon States Language

A Step Functions workflow is a **state machine**: a graph of **states** defined in JSON
(**Amazon States Language**, ASL). Each state does work, makes a decision, or controls flow, then
transitions to the next.

```json
{
  "Comment": "Process an order",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:111122223333:function:validate",
      "Retry": [{ "ErrorEquals": ["States.TaskFailed"], "MaxAttempts": 3, "BackoffRate": 2.0 }],
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "NotifyFailure" }],
      "Next": "ChargePayment"
    },
    "ChargePayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:111122223333:function:charge",
      "Next": "FulfillOrder"
    },
    "FulfillOrder": { "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:111122223333:function:fulfill", "End": true },
    "NotifyFailure": { "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:111122223333:function:notify", "End": true }
  }
}
```

---

## 3. Standard vs Express Workflows

The single biggest Step Functions exam decision.

| Dimension | **Standard** | **Express** |
|-----------|--------------|-------------|
| Max duration | **1 year** | **5 minutes** |
| Execution model | Exactly-once | At-least-once (or at-most-once, sync) |
| Pricing | Per **state transition** | Per **request + duration/memory** |
| Rate | ~2,000 starts/s | **100,000+ starts/s** |
| Execution history | Full, in the console (90 days) | CloudWatch Logs only |
| Best for | Long-running, auditable, human-in-the-loop, low-to-moderate volume | High-volume, short, event-processing pipelines |

> **Rule**: **Standard** for durable, long-running, must-be-exactly-once workflows (order
> fulfillment, approvals, saga). **Express** for high-throughput, short-lived event processing
> (streaming/IoT/transform) where cost-per-event matters and the run finishes in ≤ 5 min.

---

## 4. State Types

| State | Purpose |
|-------|---------|
| **Task** | Do work — invoke a Lambda, call an AWS service, run an activity |
| **Choice** | Branch based on input (`if/else` / `switch`) |
| **Parallel** | Run multiple branches **concurrently**, join when all finish |
| **Map** | Run the same steps over **each item** in an array (fan-out; Inline or Distributed) |
| **Wait** | Pause for a duration or until a timestamp |
| **Pass** | Pass input through (optionally inject/transform data) — useful for shaping/debugging |
| **Succeed** | End the execution successfully |
| **Fail** | End the execution with an error (name + cause) |

```
        ┌───────────────┐
        │  ValidateOrder │ (Task)
        └───────┬───────┘
                ▼
          ┌───────────┐   valid?
          │  IsValid? │ (Choice)
          └─────┬─────┘
        yes ────┤──── no
                ▼          ▼
        ┌──────────────┐  ┌─────────────┐
        │ ChargePayment │  │ NotifyFailure│ (Fail path)
        └──────┬───────┘  └─────────────┘
               ▼
       ┌────────────────┐
       │   Parallel      │  ── branch 1: UpdateInventory
       │ (fan-out work)  │  ── branch 2: SendConfirmation
       └───────┬────────┘
               ▼
           ┌────────┐
           │ Succeed │
           └────────┘
```

---

## 5. Error Handling & Retries

This is *the* reason to use Step Functions instead of chaining Lambdas by hand — declarative,
no boilerplate.

- **`Retry`** — retry a failed state with configurable `MaxAttempts`, `IntervalSeconds`, and
  `BackoffRate` (exponential backoff). Match specific errors via `ErrorEquals`.
- **`Catch`** — on a matching error, transition to a recovery state instead of failing the whole
  execution (the equivalent of `try/catch`).
- **`States.Timeout`, `States.TaskFailed`, `States.ALL`** — built-in error names to match on.

```json
"Retry": [
  { "ErrorEquals": ["States.Timeout"], "MaxAttempts": 5, "IntervalSeconds": 2, "BackoffRate": 2.0 }
],
"Catch": [
  { "ErrorEquals": ["States.ALL"], "Next": "Rollback", "ResultPath": "$.error" }
]
```

💡 `ResultPath` in a `Catch` injects the error info into the state input so the recovery state
(e.g. a saga compensation/rollback) knows what failed.

---

## 6. Service Integrations

Task states don't only call Lambda. Step Functions integrates directly with 200+ AWS services
via three patterns:

| Pattern | Behavior | Use case |
|---------|----------|----------|
| **Request/Response** | Call the service, move on immediately | Fire-and-forget API calls |
| **Run a Job (`.sync`)** | Call the service, **wait** until the job completes | Wait for an ECS task / Glue / EMR / Batch job to finish |
| **Wait for Callback (`.waitForTaskToken`)** | Pause until an external system calls back with a token | **Human approval**, third-party async callbacks |

```
"Approve": {
  "Type": "Task",
  "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
  "Parameters": { "QueueUrl": "...", "MessageBody": { "TaskToken.$": "$$.Task.Token" } },
  "Next": "Fulfill"
}
```

⚠️ With `.waitForTaskToken`, the workflow **pauses indefinitely** (up to the workflow's max
duration) until something calls `SendTaskSuccess`/`SendTaskFailure` with the token. Set a
`TimeoutSeconds` so a missing callback doesn't hang forever.

---

## 7. Use Cases

- **Long-running workflows** — order processing, onboarding, multi-stage approvals (Standard,
  up to 1 year, with human-in-the-loop via task tokens).
- **Saga pattern** — coordinate a distributed transaction; on failure, run **compensating
  actions** (refund the charge, release the inventory) via `Catch` → rollback states. Step
  Functions makes the otherwise-painful rollback logic explicit and reliable.
- **ETL / data pipeline coordination** — orchestrate Glue jobs, EMR clusters, Lambda transforms,
  and Athena queries in sequence with `.sync` integrations.
- **Beating the 15-minute Lambda limit** — break long work into steps; the *workflow* runs for up
  to a year even though each Lambda still caps at 15 minutes.
- **Fan-out processing** — `Map` over a large array (Distributed Map handles millions of items).

> **Rule**: If your "solution" is one giant Lambda doing 6 sequential things with manual retry
> code, or a Lambda that calls a Lambda that calls a Lambda — that's a Step Functions workflow.

---

## 8. Production Workflow Architecture

### Delivery semantics do not remove idempotency

Standard workflows durably record transitions and avoid duplicate workflow
executions for an idempotent `StartExecution` request with the same name while
that execution is running. A retried Task can still call a downstream API more
than once. Express asynchronous workflows use at-least-once execution; synchronous
Express uses at-most-once execution. Make every side-effecting task idempotent
with a business key, conditional write, or downstream idempotency token.

Choose Standard for durable audit history, long waits, human approval, and
redrive. Choose Express for short, high-rate processing after confirming that
duplicate or missing execution behavior is acceptable and CloudWatch logging is
configured. Nest an Express child workflow inside a Standard parent when a
durable business process contains a high-volume short transformation.

### Integration patterns and human control

- Use request/response for a short API call, `.sync` for a managed job whose
  completion gates the next state, and `.waitForTaskToken` for an external
  callback.
- **Activities** let a worker poll Step Functions for assigned work. Prefer
  service integrations or task-token callbacks for new designs unless a polling
  worker is the actual requirement.
- Treat a task token as a secret capability: send it only to the intended
  approver/worker, set `TimeoutSeconds` and a heartbeat where supported, record
  who approved, and verify business authorization before calling
  `SendTaskSuccess`.

For cross-account work, configure a Task to assume a narrowly scoped role in the
target account. The target role trusts the workflow account/role and grants only
the called resource action. Keep orchestration history in the owning account and
use CloudTrail in both accounts to prove the role assumption and target change.

### Retries, compensation, and redrive

Retry only transient errors with bounded exponential backoff and jitter where the
integration supports it. Catch permanent validation/authorization errors
immediately. A saga is not a database rollback: record which forward steps
committed, then run idempotent compensating actions in reverse dependency order.
Compensation can fail, so alarm and provide an operator repair path.

Standard failed executions can be **redriven** from the failed/aborted/timed-out
event instead of starting the entire workflow again where the execution is
eligible. Redrive preserves context and avoids repeating successful steps, but
tasks must remain idempotent and external state may have changed since failure.

### Distributed Map and quotas

Use Inline Map for a modest in-memory array and **Distributed Map** for large S3
datasets or high-concurrency child executions. Set maximum concurrency from the
downstream quota, not from the largest number Step Functions accepts. Capture
per-item failure thresholds/results, isolate poison inputs, and estimate state
transition, request, log, and child-execution cost before fan-out.

Important quotas include execution starts, state transitions, open executions,
history size, payload size, Map concurrency, and service API throttles. Some are
adjustable and vary by Region. Monitor throttled events, execution time/failure,
Map run progress, callback age, and downstream capacity through Service Quotas
and CloudWatch rather than memorizing one number.

---

## 9. Key Exam Points

- **Step Functions = orchestration** (central control, visible flow, built-in retries/branching),
  versus **choreography** (decoupled events via SNS/EventBridge).
- **Standard vs Express**: Standard = up to **1 year**, exactly-once, per-transition pricing,
  long/auditable workflows. Express = up to **5 minutes**, high-volume, per-request pricing.
- **Retry + Catch** give declarative error handling and exponential backoff — no boilerplate.
- **`.sync`** waits for a job (ECS/Glue/EMR/Batch) to finish; **`.waitForTaskToken`** pauses for an
  external callback (human approval).
- Use Step Functions to **exceed the 15-minute Lambda limit** by composing steps.
- **Saga pattern** = use `Catch` to trigger compensating/rollback actions on failure.
- **Map** state fans out over array items; **Parallel** runs distinct branches concurrently.
- Task retries can duplicate side effects even in Standard workflows; design every mutating task for idempotency.
- Standard execution redrive resumes eligible failures; Distributed Map handles large/high-concurrency datasets but must be capped to downstream quotas.
- Cross-account Tasks assume a target role; task tokens need timeout, secrecy, and an auditable authorization step.

---

## 10. Common Mistakes

- ❌ Writing manual retry/backoff and error-routing logic inside Lambdas. ✅ Use `Retry`/`Catch` in ASL.
- ❌ Choosing Standard for high-volume short events. ✅ Express is cheaper and faster there (≤ 5 min).
- ❌ Choosing Express for a workflow that must run hours/days or be exactly-once. ✅ Use Standard.
- ❌ Using `.waitForTaskToken` without a `TimeoutSeconds`. ✅ Always bound the wait.
- ❌ Forcing a multi-step process into one 15-minute Lambda. ✅ Orchestrate the steps.
- ❌ Reading "exactly once" as proof that a retried Task cannot repeat an external side effect.
- ❌ Setting Distributed Map concurrency above a downstream API/database quota and moving throttling into every child execution.
- ❌ Treating compensation as infallible or redrive as safe without checking changed external state.

---

## 11. Limits & Quick Facts

| Limit | Value |
|-------|-------|
| Standard max duration | **1 year** |
| Express max duration | **5 minutes** |
| Standard execution start rate | ~2,000/s |
| Express execution start rate | 100,000+/s |
| Standard execution history retention | 90 days (in console) |
| Express logging | CloudWatch Logs only |
| Standard pricing | Per state transition |
| Express pricing | Per request + duration/memory |

---

**Next**: [04_cognito.md — Cognito: User Pools, Identity Pools & Federated Access](04_cognito.md)
