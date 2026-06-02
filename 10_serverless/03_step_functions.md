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

## 8. Key Exam Points

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

---

## 9. Common Mistakes

- ❌ Writing manual retry/backoff and error-routing logic inside Lambdas. ✅ Use `Retry`/`Catch` in ASL.
- ❌ Choosing Standard for high-volume short events. ✅ Express is cheaper and faster there (≤ 5 min).
- ❌ Choosing Express for a workflow that must run hours/days or be exactly-once. ✅ Use Standard.
- ❌ Using `.waitForTaskToken` without a `TimeoutSeconds`. ✅ Always bound the wait.
- ❌ Forcing a multi-step process into one 15-minute Lambda. ✅ Orchestrate the steps.

---

## 10. Limits & Quick Facts

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
