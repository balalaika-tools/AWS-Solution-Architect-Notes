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
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem"],
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

If a client times out after creating an order and retries, the conditional write
prevents creating a second order with the same `order_id`. This is the small
detail that keeps "retry safely" from becoming "bill the customer twice."

---

## 8. Troubleshooting

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

---

**Back to**: [Practical Examples](README.md)
