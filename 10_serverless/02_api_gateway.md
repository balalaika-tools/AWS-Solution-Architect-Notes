# Amazon API Gateway — The Managed API Front Door

> **Who this is for**: Engineers who can write a Lambda and now need to expose it as a real
> HTTP API with auth, throttling, and routing. Read [01_lambda.md](01_lambda.md) first —
> Lambda is the most common integration target.

---

## 1. Prerequisite — What an API Gateway Does

An **API gateway** is the single entry point that sits in front of your backend services. It
is a reverse proxy with a job description:

| Responsibility | What it means |
|----------------|---------------|
| **Routing** | Map an incoming path/method (`GET /orders/{id}`) to a backend (a Lambda, an HTTP service) |
| **Authentication / authorization** | Reject unauthenticated/unauthorized callers *before* they reach the backend |
| **Throttling & quotas** | Protect the backend from traffic spikes; enforce per-client rate limits |
| **Transformation** | Reshape requests/responses, inject headers, validate payloads |
| **Cross-cutting concerns** | TLS termination, logging, metrics, caching, CORS |

> **Key insight**: The gateway lets each backend stay simple — it doesn't re-implement auth,
> rate limiting, or TLS. Those concerns move *up* to the front door. Amazon API Gateway is the
> fully managed, auto-scaling version of this pattern.

```
Clients ──▶ ┌──────────────────────────────┐ ──▶ Lambda
            │   API Gateway                 │ ──▶ HTTP backend / ALB
            │  auth · throttle · cache ·    │ ──▶ AWS service (e.g. SQS, DynamoDB)
            │  route · transform · CORS     │
            └──────────────────────────────┘
```

---

## 2. The Three API Types — REST vs HTTP vs WebSocket

API Gateway offers three distinct products. The exam loves REST vs HTTP.

| Feature | **REST API** | **HTTP API** | **WebSocket API** |
|---------|--------------|--------------|-------------------|
| Use case | Full-featured REST | Lightweight REST | Bidirectional, stateful (chat, live feeds) |
| Cost | Higher | **~70% cheaper** | Per message + connection-minute |
| Latency | Higher | **Lower** | n/a |
| Lambda / HTTP proxy | ✅ | ✅ | ✅ (via routes) |
| **API keys & usage plans** | ✅ | ❌ | ❌ |
| **Request/response transformation (VTL)** | ✅ | ❌ | limited |
| **Caching** | ✅ | ❌ | ❌ |
| **Request validation / models** | ✅ | ❌ | ❌ |
| **WAF integration** | ✅ | ❌ | ❌ |
| **Private endpoint** | ✅ | ✅ | ❌ |
| Authorizers | IAM, Cognito, Lambda | IAM, JWT (OIDC/Cognito), Lambda | IAM, Lambda |
| Edge-optimized endpoint | ✅ | ❌ | ❌ |

> **Rule**: Default to **HTTP API** for simple Lambda/HTTP proxying — cheaper and faster. Reach
> for **REST API** when you need API keys + usage plans, caching, request validation, WAF, or
> VTL transformations. Use **WebSocket API** for persistent, two-way connections.

---

## 3. Integrations — How the Gateway Reaches the Backend

| Integration | Description |
|-------------|-------------|
| **Lambda proxy** | Passes the whole request to Lambda as the `event`; Lambda returns the full HTTP response. Simplest and most common. |
| **Lambda (non-proxy)** | Uses mapping templates (VTL) to transform request/response. More control, more config. REST only. |
| **HTTP / HTTP proxy** | Forwards to any HTTP endpoint (a public API, an ALB, an on-prem service). |
| **AWS service** | Calls an AWS API directly — e.g. `PutItem` to DynamoDB or `SendMessage` to SQS — **no Lambda needed**. |
| **Mock** | Returns a canned response from the gateway itself (useful for stubbing / CORS preflight). |

💡 **Direct AWS service integration** is an exam favorite: "ingest high-volume events into SQS
with minimal cost/latency" → API Gateway → SQS directly, skipping a Lambda hop entirely.

```python
# Lambda proxy: the function receives the full request and returns the full response.
def lambda_handler(event, context):
    order_id = event["pathParameters"]["id"]      # from GET /orders/{id}
    qs = event.get("queryStringParameters") or {}
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"orderId": order_id, "format": qs.get("format", "json")}),
    }
```

---

## 4. Stages & Deployments

API Gateway separates *defining* the API from *publishing* it.

```
[ API definition: routes, methods, integrations ]
        │  deploy
        ▼
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │  dev    │   │  test   │   │  prod   │   ← stages: each is a named, versioned snapshot
   └─────────┘   └─────────┘   └─────────┘     with its own URL, throttling, caching, vars
```

- A **deployment** is a snapshot of the API definition pushed to a **stage** (`dev`, `prod`).
- Each stage has its own **invoke URL**, **stage variables** (config like which Lambda alias to
  call), throttling settings, logging, and cache.
- **Canary deployments** (REST) shift a percentage of traffic to a new deployment for safe rollout.

---

## 5. Authorizers — Controlling Who Gets In

| Authorizer | How it works | When to use |
|------------|--------------|-------------|
| **IAM** | Caller signs requests with SigV4; gateway checks IAM permissions | Service-to-service, internal AWS callers, IAM-based clients |
| **Cognito User Pool** | Caller sends a Cognito JWT; gateway validates it | End-user apps where Cognito manages sign-up/sign-in |
| **Lambda authorizer** | Your Lambda inspects the token/headers and returns an IAM policy (allow/deny) | Custom auth — third-party JWTs, OAuth, header-based, anything bespoke |
| **JWT authorizer** (HTTP API only) | Native OIDC/OAuth JWT validation, no Lambda needed | Any OIDC IdP (incl. Cognito) on HTTP APIs |

```
Client ──token──▶ API GW ──▶ Authorizer ──allow/deny──▶ (if allow) Backend integration
                              │
                              └─ Lambda authorizer can return a policy + cache it (TTL)
```

💡 **Lambda authorizer results are cached** by token (configurable TTL) so you don't invoke the
authorizer on every request — important for cost and latency.

---

## 6. Throttling, Usage Plans & API Keys

API Gateway protects backends with **token-bucket throttling**: a steady **rate** (requests/sec)
plus a **burst** (bucket size for spikes).

```
Account-level limit  ──▶  Stage / method limit  ──▶  Usage plan limit (per API key)
   (broadest)                                            (per-client quota)
```

- **API keys** identify a *client* (not for authentication — they're identifiers, not secrets).
- **Usage plans** (REST only) attach rate limits + **quotas** (e.g. 10,000 calls/day) to API keys
  — the basis for monetizing or tiering an API.
- Exceeding a limit returns **429 Too Many Requests**.

⚠️ API keys are **not authorization**. Don't rely on them to secure an API — pair them with an
authorizer. They're for identifying and metering clients in a usage plan.

---

## 7. Caching (REST API only)

A stage-level cache stores integration responses keyed by request parameters, so repeat
requests skip the backend entirely.

```
GET /products/42 ──▶ [cache HIT] ──▶ cached response   (backend not called)
                 └─▶ [cache MISS] ─▶ backend ─▶ store in cache (TTL) ─▶ response
```

- Configurable size (0.5 GB → 237 GB) and **TTL**; reduces backend load and latency.
- Clients can be allowed to bypass with `Cache-Control: max-age=0` (guard this with auth).

---

## 8. CORS & Endpoint Types

**CORS** (Cross-Origin Resource Sharing) — browsers block calls from a web app on origin A to an
API on origin B unless the API returns the right `Access-Control-Allow-*` headers. API Gateway
handles the preflight `OPTIONS` request and adds these headers when CORS is enabled.

⚠️ "My browser app gets a CORS error but `curl` works fine" → enable CORS on the API. `curl`
doesn't enforce CORS; browsers do.

**Endpoint types** (where the API is reachable):

| Type | Reachable from | Use when |
|------|----------------|----------|
| **Edge-optimized** (REST) | Internet, via CloudFront edge | Geographically dispersed clients |
| **Regional** | Internet, in one Region | Clients in-Region, or you front it with your own CloudFront |
| **Private** | Only inside a VPC (via an interface VPC endpoint) | Internal APIs that must never touch the internet |

A private API is reached through an `execute-api` **interface VPC endpoint**.
That endpoint creates ENIs in your VPC and has a security group. You can also
use an API resource policy to restrict calls to a specific VPC or VPC endpoint.
See [ENIs, Security Groups & Service Networking](../03_networking/07_enis_security_groups_and_service_networking.md)
for the endpoint-side model.

---

## 9. API Gateway vs ALB

Both can front Lambda and HTTP backends. Exam picks one based on the *feature set*.

| Need | Choose |
|------|--------|
| Rich API features: API keys, usage plans, request validation, caching, transformations, WebSocket | **API Gateway** |
| Per-request pricing, scale-to-zero, no infrastructure | **API Gateway** |
| Cognito/JWT/Lambda authorizers, fine-grained throttling | **API Gateway** |
| High, steady traffic where per-request pricing gets expensive | **ALB** (hourly + LCU) |
| Routing to EC2/ECS/IP targets and Lambda with simple host/path rules | **ALB** |
| Layer-7 load balancing within a VPC | **ALB** |

> **Rule of thumb**: **API Gateway = an API management layer** (auth, throttling, keys, caching).
> **ALB = a load balancer** that can also invoke Lambda. Spiky/low traffic with API features →
> API Gateway. High constant traffic, plain routing → ALB is cheaper.

---

## 10. Key Exam Points

- **HTTP API**: cheaper, faster, simpler — default for Lambda/HTTP proxying. **REST API**: choose
  it for API keys + usage plans, caching, request validation, WAF, or VTL transformations.
- **Direct AWS service integration** (e.g. API GW → SQS/DynamoDB) avoids a Lambda hop — favored
  for cost/latency on simple ingest patterns.
- **Authorizers**: IAM (SigV4), Cognito User Pool (JWT), Lambda authorizer (custom), JWT (HTTP API).
- **API keys ≠ authentication** — they identify/meter clients in usage plans; pair with an authorizer.
- **Caching and usage plans/API keys are REST-only.**
- **Stages** are deployable, independently-configured versions of an API.
- **CORS errors are browser-side** — enable CORS on the API; it doesn't affect non-browser callers.
- **Private endpoint** = reachable only via an interface VPC endpoint, never the internet.
- API Gateway vs ALB: gateway for API management & spiky traffic; ALB for high steady traffic & plain routing.

---

## 11. Common Mistakes

- ❌ Picking REST API by default. ✅ Use HTTP API unless you need a REST-only feature.
- ❌ Treating API keys as a security mechanism. ✅ Authenticate with an authorizer; keys are for metering.
- ❌ Adding a Lambda just to drop a message in SQS. ✅ Use direct AWS service integration.
- ❌ Forgetting CORS, then debugging the "broken" backend. ✅ Enable CORS for browser clients.
- ❌ Using API Gateway for very high, constant traffic without checking cost. ✅ Compare against ALB.

---

## 12. Limits & Quick Facts

| Limit | Value |
|-------|-------|
| Default throttle (account, REST) | 10,000 req/s, 5,000 burst |
| Integration timeout (REST/HTTP) | 29 seconds (max) |
| Payload size | 10 MB |
| Cache size (REST) | 0.5 GB – 237 GB |
| Caching / usage plans / API keys | REST API only |
| WAF integration | REST API only |
| Edge-optimized endpoint | REST API only |

---

**Next**: [03_step_functions.md — Step Functions: Orchestrating Serverless Workflows](03_step_functions.md)
