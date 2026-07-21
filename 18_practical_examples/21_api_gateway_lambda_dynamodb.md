# Serverless API: API Gateway -> Lambda -> DynamoDB

> **Who this is for**: Engineers prepping for SAA-C03 who need a practical
> serverless web API pattern: an HTTPS front door, a stateless compute layer, and
> a managed NoSQL table. Assumes you've read
> [API Gateway](../10_serverless/02_api_gateway.md),
> [Lambda](../10_serverless/01_lambda.md), and
> [DynamoDB](../06_databases/04_dynamodb.md).

> **Key insight**: For a simple Lambda-backed API, start with an **HTTP API**:
> lower cost, lower latency, and fewer moving parts than a REST API. Keep the
> Lambda **outside a VPC** when it only calls public AWS services like DynamoDB;
> adding VPC networking would add NAT/endpoints and security groups without
> improving inbound security.

---

## 1. What We're Building

A small orders API:

- `POST /orders` creates an order.
- `GET /orders/{orderId}` fetches one order by ID.
- `GET /customers/{customerId}/orders` lists a customer's recent orders.

The architecture:

```
Browser / mobile / service client
        |
        | HTTPS
        v
  API Gateway HTTP API
        | route + JWT/IAM authorizer + throttling
        v
  Lambda function (outside VPC)
        | IAM role permits DynamoDB calls
        v
  DynamoDB Orders table
        | PK: order_id
        | GSI: customer_id-created_at-index
```

This is a scale-to-zero pattern: no EC2 instances, no load balancer nodes, no
database instances. API Gateway, Lambda, and DynamoDB all scale with requests.

---

## 2. Why This Pattern

| Requirement | Service choice |
|-------------|----------------|
| Public HTTPS API with routes and auth | API Gateway HTTP API |
| Short stateless request handler | Lambda |
| Key-value/order lookup at high scale | DynamoDB |
| Unknown or spiky traffic | DynamoDB on-demand + Lambda autoscaling |
| Browser app callers | CORS on API Gateway |
| End-user auth | Cognito/JWT authorizer |
| Service-to-service auth | IAM authorizer with SigV4 |

Use a **REST API** instead of an HTTP API when the question requires REST-only
features such as API keys and usage plans, stage caching, request validation
models, direct WAF association, private API endpoints, VTL transformations, or
direct AWS service integrations that HTTP APIs do not support for your target
service.

---

## 3. DynamoDB Table Design

Design the table around the API access patterns:

| Access pattern | Key used |
|----------------|----------|
| Fetch one order by ID | table PK: `order_id` |
| Create an order idempotently | conditional write on `order_id` |
| List a customer's recent orders | GSI PK: `customer_id`, GSI SK: `created_at` |

```bash
aws dynamodb create-table \
  --table-name Orders \
  --billing-mode PAY_PER_REQUEST \
  --attribute-definitions \
      AttributeName=order_id,AttributeType=S \
      AttributeName=customer_id,AttributeType=S \
      AttributeName=created_at,AttributeType=S \
  --key-schema AttributeName=order_id,KeyType=HASH \
  --global-secondary-indexes '[
    {
      "IndexName": "customer_id-created_at-index",
      "KeySchema": [
        {"AttributeName": "customer_id", "KeyType": "HASH"},
        {"AttributeName": "created_at", "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "ALL"}
    }
  ]'
```

On-demand capacity is the simplest answer for new or unpredictable workloads.
For steady, predictable traffic, use provisioned capacity with auto scaling.

---

## 4. Lambda Execution Role

The Lambda role needs CloudWatch Logs and least-privilege DynamoDB permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:TransactWriteItems"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:111122223333:table/Orders"
    },
    {
      "Effect": "Allow",
      "Action": "dynamodb:Query",
      "Resource": "arn:aws:dynamodb:us-east-1:111122223333:table/Orders/index/customer_id-created_at-index"
    }
  ]
}
```

Attach `AWSLambdaBasicExecutionRole` or an equivalent custom policy for log
creation and log writes.

---

## 5. Lambda Handler (Proxy Integration)

With Lambda proxy integration, API Gateway passes the HTTP request as the
`event`, and Lambda returns the full HTTP response.

```python
import json
import os
import time
import uuid

import boto3
from boto3.dynamodb.conditions import Key
from botocore.exceptions import ClientError

ddb = boto3.resource("dynamodb")
table = ddb.Table(os.environ["TABLE_NAME"])


def response(status_code, body):
    return {
        "statusCode": status_code,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "https://app.example.com",
        },
        "body": json.dumps(body),
    }


def lambda_handler(event, context):
    method = event["requestContext"]["http"]["method"]
    path_params = event.get("pathParameters") or {}

    if method == "POST":
        payload = json.loads(event.get("body") or "{}")
        order_id = payload.get("order_id") or str(uuid.uuid4())
        created_at = payload.get("created_at") or time.strftime(
            "%Y-%m-%dT%H:%M:%SZ",
            time.gmtime(),
        )
        item = {
            "order_id": order_id,
            "customer_id": payload["customer_id"],
            "created_at": created_at,
            "status": "PENDING",
            "total": str(payload["total"]),
        }

        try:
            table.put_item(
                Item=item,
                ConditionExpression="attribute_not_exists(order_id)",
            )
        except ClientError as exc:
            if exc.response["Error"]["Code"] == "ConditionalCheckFailedException":
                return response(409, {"message": "order_id already exists"})
            raise

        return response(201, item)

    if method == "GET" and "orderId" in path_params:
        result = table.get_item(Key={"order_id": path_params["orderId"]})
        item = result.get("Item")
        if not item:
            return response(404, {"message": "order not found"})
        return response(200, item)

    if method == "GET" and "customerId" in path_params:
        result = table.query(
            IndexName="customer_id-created_at-index",
            KeyConditionExpression=Key("customer_id").eq(path_params["customerId"]),
            ScanIndexForward=False,
            Limit=25,
        )
        return response(200, {"items": result["Items"]})

    return response(404, {"message": "route not found"})
```

Do **not** put this Lambda in a VPC just to reach DynamoDB. DynamoDB is an AWS
service endpoint, and a normal Lambda can call it without NAT or VPC endpoints.
Use VPC attachment only when the function must reach private resources such as
RDS, ElastiCache, or an internal load balancer.

---

## 6. API Gateway HTTP API

Create the API and Lambda proxy integration:

```bash
aws apigatewayv2 create-api \
  --name orders-api \
  --protocol-type HTTP \
  --cors-configuration '{
    "AllowOrigins": ["https://app.example.com"],
    "AllowMethods": ["GET", "POST", "OPTIONS"],
    "AllowHeaders": ["content-type", "authorization"]
  }'

aws apigatewayv2 create-integration \
  --api-id api-1234 \
  --integration-type AWS_PROXY \
  --integration-uri arn:aws:lambda:us-east-1:111122223333:function:orders-api \
  --payload-format-version 2.0
```

Add routes:

```bash
aws apigatewayv2 create-route \
  --api-id api-1234 \
  --route-key "POST /orders" \
  --target integrations/int-1234

aws apigatewayv2 create-route \
  --api-id api-1234 \
  --route-key "GET /orders/{orderId}" \
  --target integrations/int-1234

aws apigatewayv2 create-route \
  --api-id api-1234 \
  --route-key "GET /customers/{customerId}/orders" \
  --target integrations/int-1234
```

Allow API Gateway to invoke the function:

```bash
aws lambda add-permission \
  --function-name orders-api \
  --statement-id apigw-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:111122223333:api-1234/*/*/*"
```

For production, add an authorizer:

- **JWT/Cognito authorizer** for end-user APIs.
- **IAM authorizer** for service-to-service callers that can sign requests.
- **Lambda authorizer** only when auth logic is custom.

API keys are not authentication. If a question mentions API keys and usage plans,
that points to REST API, and you should still pair keys with real auth.

---

## 7. Scaling and Protection

| Layer | Control | What it protects |
|-------|---------|------------------|
| API Gateway | route/stage throttling | callers from overwhelming the API |
| Lambda | reserved concurrency | downstream DynamoDB or third-party services |
| DynamoDB | on-demand or provisioned auto scaling | table throughput |
| DynamoDB writes | conditional expressions | duplicate client retries |
| App code | idempotency token/order ID | repeated POST requests |

If the client supplies the same stable `order_id`, the conditional write prevents a duplicate.
If the server generated the UUID and the client never received it, that is insufficient: the
production idempotency-key/result pattern in §9 is what makes the ambiguous retry safe.

---

## 8. Production Edge and Authorization

Authentication proves a caller identity; the handler must still authorize that identity for the
specific order/customer.

- For a JWT authorizer, pin the trusted issuer and audience, require route scopes such as
  `orders:read`/`orders:write`, and derive `subject`/`tenant_id` from validated claims. Never trust
  a `customer_id` in the request body as proof the caller owns that customer.
- For IAM authorization, require SigV4 and grant `execute-api:Invoke` only for the intended stage,
  method, and route ARN. This fits AWS workloads; it is not a user-login system.
- Use a Lambda authorizer only for genuinely custom policy logic. Bound its cache key/TTL so an
  authorization decision for one tenant/token cannot be reused for another.
- Authorize again in Lambda before the DynamoDB key operation and include tenant ownership in the
  item/key or a conditional expression. Route authorization alone cannot enforce row-level access.

For example, create and attach a JWT authorizer rather than leaving the production routes open:

```bash
AUTHORIZER_ID=$(aws apigatewayv2 create-authorizer \
  --api-id api-1234 \
  --name orders-jwt \
  --authorizer-type JWT \
  --identity-source '$request.header.Authorization' \
  --jwt-configuration \
    Audience=orders-api,Issuer=https://id.example.com/ \
  --query AuthorizerId --output text)

aws apigatewayv2 update-route \
  --api-id api-1234 \
  --route-id route-post-orders \
  --authorization-type JWT \
  --authorizer-id "$AUTHORIZER_ID" \
  --authorization-scopes orders:write
```

Return `401` for absent/invalid authentication and `403` for an authenticated principal that is
not authorized. Do not log bearer tokens or full personal/payment payloads.

### TLS, custom domain, and WAF

Create a Regional API Gateway custom domain such as `api.example.com` with an ACM certificate in
the **same Region**; HTTP APIs currently use a TLS 1.2 security policy. Map the production stage,
create a Route 53 alias, and disable the default `execute-api` endpoint when clients must use only
the governed custom domain.

AWS WAF associates directly with API Gateway **REST API stages**, not HTTP APIs. If direct WAF is a
requirement, choose a REST API. To keep an HTTP API, put CloudFront with a WAF web ACL in front and
prevent bypass of the CloudFront path with a controlled origin/header/authorization design. That
adds CloudFront caching, logging, certificate-Region, and deployment responsibilities; it is not a
checkbox on the HTTP API. Use WAF managed rules in count mode first, rate-based rules for abusive
clients, request-size limits, and sampled/logged requests with sensitive-field redaction.

Set API route/stage throttles and, for public clients, per-identity business limits in the
application. WAF rate limits and API throttles reduce abuse/cost but do not replace authorization.

---

## 9. Retries, Idempotency, and DynamoDB Safety

An API Gateway→Lambda invocation is synchronous, but partial failure is still ambiguous: Lambda
can commit DynamoDB after the client/API timeout, the SDK can retry a throttled call, and the
client/CDN can retry a POST. The same request must return the same business result.

### Use a client idempotency key, not only a client-chosen order ID

Require `Idempotency-Key` on `POST /orders`, scope it to the authenticated subject/tenant, hash the
canonical request, and atomically write the result mapping with the order:

```text
TransactWriteItems
  Put order_id=<generated UUID>, customer_id=<authorized claim>, ...
      condition attribute_not_exists(order_id)
  Put order_id="IDEMP#<tenant>#<idempotency-key>",
      request_hash=<hash>, result_order_id=<UUID>, expires_at=<retention>
      condition attribute_not_exists(order_id)
```

On retry, read the idempotency item consistently, reject the same key with a different request
hash, and return the stored order/result. DynamoDB TTL can clean old idempotency records, but TTL
deletion is asynchronous—expiration logic must be checked by the application and the retention
must exceed the longest credible client retry. If the operation calls an external payment or
publishes an event, pass the same key downstream or write a transactional outbox; a DynamoDB
record cannot make a third-party side effect idempotent by itself.

For updates, use optimistic concurrency:

```text
UpdateItem order_id=<id>
  SET status=:new, version=:next
  CONDITION tenant_id=:caller_tenant AND version=:expected
```

Return `409` on a failed version condition so a stale client cannot overwrite a newer state.
`TransactWriteItems` protects all local items in the operation; keep transactions small and do not
retry a non-idempotent external side effect inside the transaction retry loop.

### Protect keys and indexes from skew

The base `order_id` UUID spreads traffic, but the GSI partition key is `customer_id`. One very
large customer can make that index partition hot even when table-wide capacity looks healthy.
Use pagination (`LastEvaluatedKey`), bounded date ranges, and projection of only required
attributes. If load tests show a hot tenant, write a calculated shard suffix such as
`customer_id#00..N` and query shards in parallel with a bounded merge, or partition by a natural
sub-entity/time bucket. Do not pre-shard without evidence; it adds read fan-out and migration work.

DynamoDB adaptive/on-demand capacity helps uneven traffic but cannot make an unbounded single-key
access pattern safe. Alarm on table **and each GSI** throttling, consumed/provisioned capacity,
successful request latency, system errors, and Contributor Insights hot-key evidence where used.

### Coordinate quotas as one admission-control system

Estimate steady Lambda concurrency as request rate × execution duration and add burst/headroom.
Then align:

```text
API route throttle
  ≤ Lambda account headroom / function reserved concurrency
  ≤ safe downstream concurrency
  ≤ DynamoDB table + GSI key/capacity envelope
```

Reserved concurrency protects the account and downstream but also creates Lambda `429` responses.
API Gateway quotas/throttles, Lambda burst/account concurrency, payload/timeout limits, and
DynamoDB table/index/account quotas are independent. Request increases and run a full ramp test
before launch; a serverless service scaling automatically does not mean every quota is unlimited.
Use exponential backoff with jitter for retryable throttles and cap retries within the caller's
latency budget. Reject overload early rather than allowing synchronized retries to become a storm.

---

## 10. Observability, Safe Deployment, and Cost

Enable JSON API access logs with request ID, route, status, response/integration latency,
integration error, authorizer subject/tenant surrogate, client/user-agent, and WAF/edge request ID
where available. Apply retention and data protection/redaction. Lambda should emit structured logs
with the API request ID, trace ID, tenant surrogate, idempotency key hash, order ID, attempt,
dependency latency, and error class—never tokens or full order payloads.

Enable Lambda active tracing and propagate a trace/correlation ID through DynamoDB SDK calls and
any outbox/async work. If API Gateway stage-level X-Ray trace integration is required, verify the
chosen API type supports it; REST API has capabilities that HTTP API intentionally omits. Use
CloudWatch service metrics plus application EMF/OpenTelemetry metrics rather than trying to infer
business success from `Lambda Invocations`.

Minimum alarms:

| Layer | Alarm/evidence |
|-------|----------------|
| API Gateway/edge | 5XX, unexpected 4XX/429, p95/p99 latency and integration latency, WAF blocks/rate limits |
| Lambda | errors, throttles, duration near timeout, concurrent executions/reserved limit, iterator/async age if later added |
| DynamoDB | read/write throttles and system errors for table **and GSI**, latency, consumed capacity/hot keys, replication lag in multi-Region |
| Business | order-create success rate, duplicate/conflict rate, idempotency replay/mismatch, stale order/outbox age |

Use Lambda **versions and an alias** as the API integration target. Deploy a small weighted canary
with CodeDeploy (or an equivalent versioned pipeline), gate on API/Lambda/business alarms, bake,
then shift traffic. Configure automatic rollback to the previous alias version. Keep schema/item
changes backward compatible across both versions; deploy readers before writers for a new field,
and never couple rollback to deleting data the old version still needs. Scope API Gateway's invoke
permission to the alias and make the integration ARN point to that alias; an unqualified function
ARN bypasses the controlled version shift.

Manage API routes, authorizers, stage settings, custom domain, WAF/CloudFront, table/indexes,
alarms, and permissions through IaC. Use explicit production stages/deployments rather than an
unreviewed auto-deploy path. Run contract, authorization, idempotency, throttling, and rollback
tests against the canary.

Cost controls include choosing HTTP API only when its smaller feature set fits, right-sizing log
volume/retention and trace sampling, setting WAF/CloudFront expectations, comparing DynamoDB
on-demand with provisioned auto scaling after traffic stabilizes, projecting smaller GSI items,
and using Budgets/Cost Anomaly Detection. Throttling is not only protection—it caps runaway
Lambda/DynamoDB/log spend during abuse or a retry loop.

---

## 11. Multi-Region Variant

Deploy the complete API stack independently in two Regions:

```text
                    Route 53 latency/failover records
                  api.example.com (health-checked aliases)
                         /                    \
               Region A                       Region B
        ACM + API + Lambda alias        ACM + API + Lambda alias
                  \                         /
                   DynamoDB Global Table
             + Regional secrets/config/KMS/logs
```

Use the same versioned IaC/artifact, but Regional ARNs for ACM, Lambda, DynamoDB, logs, KMS, and
Secrets Manager. A Regional custom domain/certificate must exist in each Region. Route 53 can
route to the Regional API endpoints; use a dedicated deep-health path or a CloudWatch/calculated
health signal that includes critical dependencies without exposing sensitive details. CloudFront
origin failover is not a general solution for retrying state-changing POST requests.

### Choose the global-table consistency and write model

- **MREC (multi-Region eventual consistency)** is the default Global Tables mode. Both replicas
  accept writes and changes normally replicate quickly, but concurrent updates converge with
  last-writer-wins. A local conditional write does not globally serialize simultaneous Region A/B
  writes. Pin each tenant/order to a home Region, use idempotency/version semantics, or make
  updates commutative when losing one concurrent value is unacceptable.
- **MRSC (multi-Region strong consistency)** synchronously replicates and can provide zero RPO,
  but it requires an eligible fixed three-Region topology (three replicas or two plus a witness),
  adds cross-Region write/read latency, and has feature/Region constraints. Conflicting concurrent
  writes can fail and must be retried. Choose it only after verifying those constraints and the
  latency/availability trade-off for the API.

For the common two-Region MREC design, state a measurable objective rather than “global means no
loss.” Example target: **RTO ≤ 10 minutes and RPO ≤ 1 minute**, validated by a replicated canary
sequence under representative load. Actual RPO is the latest intact replicated write at failure;
actual RTO includes decision, DNS/client caching, health validation, and traffic recovery.

### Replicate every non-table dependency

- Deploy Lambda versions/layers, API configuration, authorizers, WAF/edge policy, alarms, and
  quotas in both Regions before failover.
- Replicate Secrets Manager secrets to the recovery Region and use its local KMS key. Rotation on
  the primary propagates normally; promoting a replica to standalone during an outage changes the
  lifecycle and requires later reconciliation.
- Deploy AppConfig/Parameter Store/config explicitly per Region. Parameter Store is not a global
  configuration database. Multi-Region KMS replicas share key material/rotation but have separate
  policies, grants, aliases, and ARNs.
- Use a globally available identity provider or a tested Regional identity/federation recovery
  design. A JWT issuer tied only to the failed Region can make a healthy API unusable.
- Centralize dashboards/finding routing while retaining local logs, alarms, and rollback paths
  that work if the central Region is impaired.

### Failover and failback

1. Freeze deployments and fence the failed Region's write path if it may still be reachable.
2. Check Global Table replica state/replication lag, last canary sequence, secret/config readiness,
   Lambda alias version, quotas, and an end-to-end synthetic create/read in the target Region.
3. Change the Route 53 failover/weight and watch API errors/latency, Lambda throttles, DynamoDB/GSI
   throttles, conflict/idempotency metrics, and business success. Remember existing clients can
   cache DNS and retry against either Region during convergence.
4. Reconcile missing/conflicting orders with the outbox/audit trail; do not assume last-writer-wins
   matched business intent.

For failback, first redeploy and verify the recovered Region, wait for Global Table replication and
secrets/config to converge, and run synthetic traffic. Shift a canary before normal traffic. Keep
one authoritative write-routing rule until the bake period passes; DNS reversal without data and
dependency reconciliation creates a second incident.

---

## 12. Troubleshooting

**API returns 500, Lambda logs show "Malformed Lambda proxy response"**

- Lambda did not return `statusCode`, `headers`, and a string `body`.
- `body` must be JSON-encoded text, not a raw object.

**API Gateway returns 403**

- Missing/failed authorizer, bad IAM SigV4 signature, or Lambda permission does
  not allow the API's `execute-api` ARN to invoke the function.

**Lambda gets `AccessDeniedException` from DynamoDB**

- Execution role is missing table/index permissions.
- Querying the GSI requires permissions on the **index ARN**, not just the table
  ARN.

**Browser gets a CORS error but `curl` works**

- CORS is a browser rule. Configure allowed origins/methods/headers on API
  Gateway and return CORS headers from Lambda on error responses too.

**Requests are throttled**

- `429` from API Gateway means gateway throttling or quota.
- `TooManyRequestsException` from Lambda means Lambda concurrency throttling.
- `ProvisionedThroughputExceededException` from DynamoDB means table or index
  capacity throttling, usually provisioned capacity or a hot key.

**Lambda times out after being attached to a VPC**

- VPC-attached Lambda lost normal public AWS-service egress. Add a DynamoDB
  gateway endpoint or NAT, or remove VPC attachment if no private resource is
  needed.

**List-by-customer is slow or expensive**

- Do not `Scan` the table and filter in code. Query the GSI with
  `customer_id = :customer_id` and a sort-key order on `created_at`.

**A retried POST returns 409 even though the client never received the first result**

- A conditional order ID prevents duplicates but does not replay the original response. Use a
  tenant-scoped idempotency key and atomic idempotency-result record; reject key reuse only when
  the canonical request hash differs.

**Table has spare capacity but one customer is throttled**

- The `customer_id` GSI partition is hot. Confirm with per-index metrics/Contributor Insights,
  bound/paginate queries, then shard that access pattern if measured load requires it.

**Regional API is healthy but users cannot authenticate after failover**

- JWT issuer, key discovery, Cognito/federation, secrets, or custom-domain certificate remained a
  primary-Region dependency. Include identity and configuration in readiness canaries.

**MREC table returned an older/lost concurrent update**

- Cross-Region writes are eventually replicated and last-writer-wins. Use Region pinning,
  application versions/idempotency/merge semantics, or evaluate MRSC if its topology and latency
  constraints fit.

---

## Key Exam Points

- HTTP API is the default for simple Lambda proxy APIs; REST API is for
  REST-only features such as usage plans/API keys, caching, request validation,
  direct WAF association, private API endpoints, and VTL transformations.
- Lambda outside a VPC can call DynamoDB directly. VPC-attach only for private
  resources; otherwise you create unnecessary NAT/endpoints work.
- DynamoDB tables are designed from access patterns. Use `Query` on keys and
  indexes; avoid `Scan` for request-path reads.
- The Lambda execution role needs table permissions and separate GSI ARN
  permissions for `Query`.
- Use conditional writes or idempotency tokens for safe client retries.
- Authorizers secure the API; API keys only identify/meter clients.
- Throttling can happen at API Gateway, Lambda concurrency, or DynamoDB capacity
  layers; identify which layer returned the error.
- Production APIs need route and row-level authorization, a custom TLS domain, an explicit WAF
  pattern, structured logs/traces/business alarms, and canary deployment with alarm rollback.
- Idempotency must replay the stored result after ambiguous timeout; conditional writes and
  optimistic versions protect DynamoDB state, while external effects need the same token/outbox.
- Coordinate API throttles, Lambda concurrency, and DynamoDB table/GSI capacity/quotas from load
  evidence, including hot-key behavior and cost limits.
- A multi-Region API needs a full Regional deployment plus DNS routing, Global Table conflict/
  consistency choice, Regional identity/secrets/config/KMS, measured RTO/RPO, and tested
  failover/failback—not only adding a table replica.

---

**Back to**: [Practical Examples](README.md)
