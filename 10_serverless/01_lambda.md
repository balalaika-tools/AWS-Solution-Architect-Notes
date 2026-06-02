# AWS Lambda — Event-Driven Serverless Compute

> **Who this is for**: Engineers who know EC2 and containers and want the serverless compute
> model for the exam — when Lambda fits, how it's invoked, how it scales, and the networking
> traps. Read [Compute](../04_compute/README.md) first if EC2 is unfamiliar.

---

## 1. Prerequisite — What "Serverless" Means

"Serverless" does **not** mean no servers. It means *you don't provision, patch, or scale
the servers* — AWS does. You hand over a unit of code and a trigger; AWS runs the code on
demand and bills you for the work, not for idle capacity.

Three defining properties:

| Property | What it means | Contrast with EC2 |
|----------|---------------|-------------------|
| **No servers to manage** | No OS, no patching, no capacity planning | EC2: you own the instance, AMI, scaling group |
| **Event-driven** | Code runs in response to an event (HTTP request, file upload, message) | EC2: a process you start runs continuously |
| **Pay-per-use** | Billed per request and per ms of compute; $0 when idle | EC2: billed per second the instance is *running*, idle or not |

> **Key insight**: Lambda's superpower is **scale-to-zero and scale-out automatically**. One
> request or ten thousand concurrent requests — you write the same code and AWS runs as many
> copies as needed. The flip side is the **15-minute hard ceiling** and **stateless** model.

```
EVENT SOURCE ──▶ LAMBDA SERVICE ──▶ spins up an execution environment ──▶ runs your handler
   (S3 upload)      (managed)            (micro-VM, your runtime)            (returns/ack)
```

---

## 2. Anatomy — Function, Runtime, Handler

A Lambda **function** is your code plus configuration (memory, timeout, role, env vars,
triggers). It runs inside a **runtime** (the language environment) and AWS invokes a specific
entry point called the **handler**.

```python
# handler.py — runtime: python3.12, handler configured as "handler.lambda_handler"
import json
import boto3

# Code OUTSIDE the handler runs ONCE per execution environment (cold start).
# Reuse SDK clients here — they survive across warm invocations.
s3 = boto3.client("s3")

def lambda_handler(event, context):
    """
    event   — the trigger payload (shape depends on the source: S3, API GW, SQS...)
    context — runtime metadata: request_id, remaining time, function name, memory limit
    """
    bucket = event["Records"][0]["s3"]["bucket"]["name"]
    key = event["Records"][0]["s3"]["object"]["key"]

    # context.get_remaining_time_in_millis() — guard long work against the timeout
    obj = s3.get_object(Bucket=bucket, Key=key)
    return {"statusCode": 200, "body": json.dumps({"size": obj["ContentLength"]})}
```

✅ Initialize clients and load config **outside** the handler — that code runs once per
environment and is reused on warm invocations.

❌ Don't open a new DB connection *inside* the handler on every call — you'll exhaust the
database's connection pool under concurrency (see RDS Proxy for this exact problem).

**Supported runtimes**: Node.js, Python, Java, .NET, Ruby, Go (via `provided` / custom
runtimes), plus a **custom runtime** API and **container image** packaging (up to 10 GB image).

---

## 3. Event Sources & Triggers

Lambda is only useful when something *invokes* it. The source determines the **event shape**
and the **invocation type**.

| Source | Invocation type | Notes |
|--------|-----------------|-------|
| **API Gateway / ALB** | Synchronous | Caller waits for the response (HTTP request/response) |
| **S3** (object created/deleted) | Asynchronous | Fire-and-forget; Lambda retries on failure |
| **SNS** | Asynchronous | Fan-out; one message → many subscribers |
| **EventBridge** (events / schedule) | Asynchronous | Cron jobs and event-pattern routing |
| **SQS** | Poll-based (event source mapping) | Lambda *polls* the queue and pulls batches |
| **DynamoDB Streams / Kinesis** | Poll-based (event source mapping) | Ordered, sharded stream processing |
| **Direct (`Invoke` API / SDK / CLI)** | Sync or async (caller chooses) | App code calling Lambda directly |

> **Rule**: For **SQS, DynamoDB Streams, and Kinesis**, Lambda is *not* pushed events. The
> Lambda service runs an **event source mapping** that polls on your behalf and invokes your
> function with a batch. You never write the polling loop.

---

## 4. Invocation Models — Synchronous vs Asynchronous vs Poll-Based

```
SYNCHRONOUS (API Gateway, ALB, direct Invoke)
  Caller ──request──▶ Lambda ──response──▶ Caller        (caller blocks; errors returned to caller)

ASYNCHRONOUS (S3, SNS, EventBridge)
  Source ──event──▶ [Lambda internal queue] ──▶ Lambda   (source gets a 202 immediately;
                          │ on failure: retry 2x                Lambda retries, then → DLQ / on-failure dest)

POLL-BASED (SQS, DynamoDB Streams, Kinesis — event source mapping)
  Queue/Stream ◀──poll── Lambda service ──invoke──▶ Lambda   (batches; failures returned to the poller)
```

| Model | Retries on error | Where failures go | Caller sees errors? |
|-------|------------------|-------------------|---------------------|
| **Synchronous** | None (caller decides) | Returned to caller | Yes |
| **Asynchronous** | Automatic, 2 retries | Dead-letter queue / on-failure destination | No (got 202 already) |
| **Poll-based** | Batch re-delivered until success or expiry | SQS DLQ (redrive) / stream retry config | No (poller handles it) |

⚠️ **Exam trap**: with **asynchronous** invocation the caller has *already moved on*. To
capture failures you must configure a **DLQ** or an **on-failure destination** — otherwise
failed events vanish after retries.

---

## 5. Memory, CPU & Timeout

Lambda has **one tuning knob: memory** (128 MB → 10,240 MB). CPU is **allocated
proportionally** to memory — you cannot set CPU directly.

```
128 MB ........ fractional vCPU
1,769 MB ...... ~1 full vCPU
10,240 MB ..... ~6 vCPUs
```

> **Key insight**: More memory often makes a CPU-bound function **cheaper**, not more
> expensive — it finishes faster (fewer billed ms) and gets more CPU. Always benchmark; the
> cheapest memory setting is rarely 128 MB.

**Timeout**: default 3 s, **maximum 15 minutes (900 s)**. Past that, AWS kills the
invocation. Anything longer needs Step Functions, Fargate, or breaking the work into pieces.

**`/tmp` ephemeral storage**: 512 MB by default, configurable up to **10,240 MB**. Scratch
space only — it persists across *warm* invocations on the same environment but is **not
durable** and is **not shared** between concurrent environments.

---

## 6. Layers, Environment Variables & Packaging

**Layers** — a `.zip` of shared libraries/dependencies mounted at `/opt`. Decouple
dependencies from function code, share across functions, shrink deployment packages.

- Up to **5 layers** per function; combined unzipped size counts toward the **250 MB** limit.

**Environment variables** — key/value config injected at runtime. Encrypt sensitive values
with KMS; better still, fetch secrets from Secrets Manager / SSM Parameter Store at init.

```bash
# Publish a layer and attach it
aws lambda publish-layer-version --layer-name shared-deps \
  --zip-file fileb://layer.zip --compatible-runtimes python3.12

aws lambda update-function-configuration --function-name image-resizer \
  --layers arn:aws:lambda:us-east-1:111122223333:layer:shared-deps:3 \
  --environment "Variables={LOG_LEVEL=INFO,TABLE_NAME=images}"
```

💡 Container image packaging (up to 10 GB) is the alternative when your dependencies blow past
the 250 MB zip limit (e.g. ML libraries).

---

## 7. Concurrency — Reserved vs Provisioned

**Concurrency** = the number of invocations running *at the same time*. Account default:
**1,000 concurrent executions per Region** (soft limit, raisable).

| Setting | What it does | Solves | Costs |
|---------|--------------|--------|-------|
| **Reserved concurrency** | Caps *and* guarantees a slice of the account pool for one function | Protects downstream (e.g. a small DB) from being overwhelmed; prevents one function starving others | Free (just partitions the pool) |
| **Provisioned concurrency** | Keeps N environments **pre-initialized and warm** | **Cold starts** for latency-sensitive APIs | Charged for the warm capacity, always-on |

```
Account pool: 1,000
 ├── function-A: reserved 200  ← A can never exceed 200; the 200 is fenced off for A
 └── unreserved: 800           ← all other functions share this
```

⚠️ Setting **reserved concurrency to 0** is a kill switch — it disables the function entirely.

---

## 8. Cold Starts

A **cold start** happens when Lambda must create a *new* execution environment: download code,
start the runtime, and run your init code (everything outside the handler).

```
COLD: [create micro-VM] → [init runtime] → [run init code] → [run handler]   (slower)
WARM: ─────────────────────────────────────────────────────▶ [run handler]   (env reused)
```

What makes cold starts worse: large packages, heavy init (DB pools, big imports), Java/.NET
(JVM/CLR spin-up), and **VPC-attached functions** historically (now much faster via Hyperplane
ENIs, but still a factor at first ENI creation).

**Mitigations**: Provisioned Concurrency (eliminates them), smaller packages, lighter init,
keeping clients outside the handler, and SnapStart (Java) which snapshots a warmed environment.

---

## 9. Lambda in a VPC

By default a Lambda function runs in an **AWS-managed VPC** with **outbound internet access**.
Attach it to **your** VPC and that changes completely.

```
DEFAULT (no VPC config):
  Lambda ──▶ Internet / public AWS APIs ✅   (AWS-managed network, has internet)

ATTACHED TO YOUR VPC:
  Lambda ──ENI──▶ your private subnet ──▶ ??? 
                                          ├─ RDS/ElastiCache in the VPC ✅ (that's the point)
                                          └─ Internet / public S3 / public APIs ❌ by default
                                             needs: NAT gateway (private subnet) for internet,
                                             or VPC endpoints for AWS services
```

- When VPC-attached, Lambda creates **Hyperplane ENIs** in your subnets so functions can reach
  private resources (RDS, ElastiCache, internal services).
- A Lambda in a **private subnet has no route to the internet** unless you add a **NAT gateway**
  (NAT lives in a public subnet; the Lambda's subnet routes `0.0.0.0/0` to it).
- Reaching public AWS services (S3, DynamoDB) from a private-subnet Lambda is cheaper via **VPC
  gateway/interface endpoints** than routing through NAT.
- Putting the function in a **public subnet** does not give it internet access. Lambda VPC ENIs
  do **not** receive public IPs, so they cannot use an IGW directly.
- The Lambda SG needs outbound access to the destination; the destination SG (RDS, Aurora,
  ElastiCache, internal ALB, interface endpoint) must allow inbound from the **Lambda SG** on the
  service port. Function invocation itself is not inbound traffic to the ENI.

> **Rule**: Only put a Lambda in a VPC if it needs to reach **VPC-private** resources. It adds
> ENI management and removes default internet. If it just calls public AWS APIs, leave it out.

See the full walkthrough: [Lambda Inside vs Outside a VPC](../18_practical_examples/14_lambda_in_vpc.md).
For the broader ENI/security-group map, see
[ENIs, Security Groups & Service Networking](../03_networking/07_enis_security_groups_and_service_networking.md).

---

## 10. Execution Role (IAM)

Every Lambda assumes an **IAM execution role** to get temporary credentials for the AWS calls
it makes. This is **what the function can do**, distinct from **who can invoke it** (a
resource policy / caller's permissions).

```
Caller permission / resource policy ──▶ "may you INVOKE this function?"
Execution role                       ──▶ "what may the function's CODE do once running?"
```

- The managed `AWSLambdaBasicExecutionRole` grants CloudWatch Logs write access — the bare
  minimum so the function can log.
- Add least-privilege statements for whatever the code touches (read one S3 bucket, write one
  DynamoDB table).
- VPC-attached functions also need EC2 ENI permissions (covered by `AWSLambdaVPCAccessExecutionRole`).

---

## 11. Pricing Model

You pay on two axes, with **no charge when idle**:

1. **Requests** — per number of invocations (first 1M/month free).
2. **Duration** — GB-seconds = (memory in GB) × (billed ms, rounded to 1 ms).

```
cost ≈ requests × $per_request  +  invocations × duration_ms × memory_GB × $per_GB_second
```

💡 This is why right-sizing memory matters: doubling memory may halve duration, leaving cost
flat while latency improves. Provisioned Concurrency adds a separate always-on charge.

---

## 12. Key Exam Points

- **Serverless = no servers to manage + event-driven + pay-per-use**, scales automatically to
  zero and out; the trade-offs are the 15-min ceiling and statelessness.
- **Max timeout 15 minutes.** Longer work → Step Functions, Fargate, or chunking.
- **Memory is the only knob; CPU scales with it.** More memory can be cheaper for CPU-bound code.
- **Three invocation models**: synchronous (API GW/ALB), asynchronous (S3/SNS/EventBridge, with
  DLQ/destination for failures), poll-based (SQS/DynamoDB Streams/Kinesis via event source mapping).
- **Reserved concurrency** caps/guarantees a slice (protects downstreams); **Provisioned
  concurrency** pre-warms environments to kill cold starts.
- **VPC-attached Lambda loses default internet** — needs a **NAT gateway** for internet or **VPC
  endpoints** for AWS services. Only attach to a VPC to reach private resources.
- **Public subnet is not enough for VPC Lambda internet** — Lambda ENIs do not get public IPs.
- **Execution role** = what the function may do; **resource policy / caller perms** = who may invoke it.
- `/tmp` is ephemeral scratch (512 MB default, up to 10 GB), not durable storage.

---

## 13. Common Mistakes

- ❌ Opening DB connections inside the handler → connection-pool exhaustion under concurrency.
  ✅ Initialize clients outside the handler; use RDS Proxy for relational DBs.
- ❌ Assuming async invocations surface errors to the caller. ✅ Configure a DLQ/on-failure destination.
- ❌ Putting every Lambda in a VPC "for security." ✅ Only VPC-attach when private resources are needed.
- ❌ Leaving memory at 128 MB to "save money." ✅ Benchmark — more memory often lowers total cost.
- ❌ Using Lambda for jobs that may exceed 15 min. ✅ Orchestrate with Step Functions or run on Fargate.

---

## 14. Limits & Quick Facts

| Limit | Value |
|-------|-------|
| Max execution timeout | **15 minutes (900 s)** |
| Memory | 128 MB – 10,240 MB (CPU scales with it) |
| `/tmp` ephemeral storage | 512 MB – 10,240 MB |
| Deployment package (zip, unzipped) | 250 MB (incl. layers); 50 MB zipped direct upload |
| Container image | 10 GB |
| Layers per function | 5 |
| Synchronous payload | 6 MB (request + response) |
| Asynchronous payload | 256 KB |
| Default concurrency / Region | 1,000 (soft limit) |
| Environment variables | 4 KB total |

---

**Next**: [02_api_gateway.md — API Gateway: The Managed API Front Door](02_api_gateway.md)
