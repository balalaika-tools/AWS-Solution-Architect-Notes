# Serverless

> Running code and exposing APIs without managing servers: Lambda for event-driven compute, API Gateway as the managed front door, Step Functions for orchestrating multi-step workflows, and Cognito for end-user authentication and federated AWS access.

[![AWS](https://img.shields.io/badge/AWS-Lambda%20%7C%20API%20GW%20%7C%20Step%20Functions%20%7C%20Cognito-FF9900.svg?logo=awslambda&logoColor=white)](https://aws.amazon.com/serverless/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_lambda.md](01_lambda.md) | Lambda | What "serverless" means; functions, runtimes, handlers; event sources; sync/async/poll invocation; Function URLs; error handling (DLQ vs Destinations); memory→CPU; timeout & `/tmp`; layers & zip-vs-container packaging; concurrency (reserved vs provisioned); cold starts & SnapStart; Lambda in a VPC; execution role; pricing; Lambda@Edge; limits. |
| [02_api_gateway.md](02_api_gateway.md) | API Gateway | What an API gateway does; REST vs HTTP vs WebSocket APIs; integrations; stages/deployments; authorizers (IAM/Cognito/Lambda); throttling, usage plans, API keys; caching; CORS; endpoint types; API Gateway vs ALB. |
| [03_step_functions.md](03_step_functions.md) | Step Functions | Orchestration vs choreography; state machines; Standard vs Express; states (Task/Choice/Parallel/Map/Wait/Pass/Fail); retries & error handling; service integrations; saga, ETL and long-running workflow use cases. |
| [04_cognito.md](04_cognito.md) | Cognito | App authentication, OIDC/OAuth/JWT primer; User Pools (directory, hosted UI, MFA, federation, JWTs) vs Identity Pools (federated identities → temporary AWS credentials); when to use each / both together. |

---

## Reading Order

1. **Lambda** — the compute primitive everything else here invokes or fronts. Start with what "serverless" actually means.
2. **API Gateway** — the managed HTTP front door that turns Lambda functions into public APIs with auth and throttling.
3. **Step Functions** — orchestrates many Lambdas (and other services) into reliable, observable workflows.
4. **Cognito** — authenticates the end users who hit your API Gateway and grants federated access to AWS resources.

---

## Related Serverless/App Services to Recognize

| Service | Exam clue |
|---------|-----------|
| **AppSync** | Managed GraphQL API, real-time subscriptions, offline/mobile data sync patterns. |
| **Amplify** | Frontend/mobile app hosting and backend scaffolding for web/mobile teams. |
| **Serverless Application Repository** | Deploy prebuilt serverless apps and patterns. |

---

## Prerequisites

- [Compute](../04_compute/README.md) — Lambda is the "no instance to manage" counterpart to EC2; the contrast clarifies the trade-offs.
- [IAM](../02_iam/README.md) — every Lambda runs under an execution role, every API Gateway authorizer maps to identities, and Cognito Identity Pools vend STS credentials. See especially [Organizations, STS & Federation](../02_iam/04_organizations_sts_federation.md).
- [Networking](../03_networking/README.md) — needed to understand why a Lambda in a VPC loses default internet access and when it needs a NAT gateway.
