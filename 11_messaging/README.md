# Messaging & Events

> Decoupling is a core AWS architecture skill at both associate and professional level. This
> section starts with the *theory* — synchronous vs asynchronous, tight vs loose coupling, and the
> three messaging models (queue, pub/sub, stream) — then maps SQS, SNS, EventBridge, and Kinesis
> onto the guarantees and failure modes each service is designed to handle.

[![AWS](https://img.shields.io/badge/AWS-Messaging%20%26%20Events-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/messaging/)
[![SQS](https://img.shields.io/badge/SQS-Queue-FF4F8B.svg?logo=amazonsqs&logoColor=white)](https://aws.amazon.com/sqs/)
[![SNS](https://img.shields.io/badge/SNS-Pub%2FSub-E7157B.svg?logo=amazonsns&logoColor=white)](https://aws.amazon.com/sns/)
[![EventBridge](https://img.shields.io/badge/EventBridge-Event%20Bus-FF4F8B.svg?logo=amazoneventbridge&logoColor=white)](https://aws.amazon.com/eventbridge/)
[![Kinesis](https://img.shields.io/badge/Kinesis-Streaming-8C4FFF.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/kinesis/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_messaging_concepts.md](01_messaging_concepts.md) | Decoupling theory | Sync vs async, the three messaging models, delivery/order/replay decisions, back-pressure math, schema evolution, DR, trust boundaries, and async observability. |
| [02_sqs.md](02_sqs.md) | Managed queue | Standard vs FIFO, visibility timeout & the duplicate trap, retention, long vs short polling, DLQ redrive, 1 MiB limit + extended client, Lambda batches, cross-account/KMS controls, and backlog operations. |
| [03_sns.md](03_sns.md) | Pub/sub | SNS→SQS fan-out, filtering, FIFO scope, cross-account/KMS policies, protocol retries and logging, data protection, and multi-Region limits. |
| [04_eventbridge.md](04_eventbridge.md) | Event bus | Cross-account/organization buses, global endpoints, archive/replay, schema governance, Pipes, Scheduler, target retries/DLQs, and central governance. |
| [05_kinesis.md](05_kinesis.md) | Real-time streaming | Shard/EFO capacity math, hot-shard operations, producer/consumer recovery, cross-account/KMS controls, regional replication patterns, Firehose, Flink, and MSK cost trade-offs. |

---

## Reading Order

1. **Messaging & Decoupling Concepts** — the vocabulary and the three models everything else builds on. Read this first even if you already know queues.
2. **SQS** — the default managed queue and the Standard vs FIFO / visibility-timeout topics the exam loves.
3. **SNS** — pub/sub and the SNS→SQS fan-out pattern.
4. **EventBridge** — the event router for AWS-service and SaaS events, plus scheduling.
5. **Kinesis** — real-time streaming and how it differs from a queue.

---

## SAP-C02 Integration Design Path

Professional-level scenarios usually describe several valid services. Start with the workload's
failure and recovery contract, then eliminate services that cannot meet it.

| Decision | Questions to ask | Likely direction |
|----------|------------------|------------------|
| **Failure semantics** | Can work be lost? Can it run twice? Must each consumer have its own retry boundary? | Durable work queue → SQS; independent fan-out buffers → SNS + SQS; routed events → EventBridge; replayable log → Kinesis. |
| **Throughput and quotas** | What are peak records/s and MiB/s? What is the average record size? Is ordering global, per entity, or unnecessary? | Do queue TPS/batch math, SQS FIFO message-group concurrency, or Kinesis shard/read-consumer math before choosing a mode. |
| **Cross-account routing** | Who owns the publisher, transport, key, and consumer? Which Organizations boundary is trusted? | Use resource policies plus least-privilege identity policies; include KMS key policy and confused-deputy conditions in the design. |
| **Regional recovery** | What are the RTO/RPO? Where are undelivered events during failover? How are duplicates reconciled? | Deploy regional endpoints and consumers. EventBridge global endpoints provide managed custom-event failover; SNS, SQS, and Kinesis require an application-level regional strategy. |
| **Schema and change governance** | Who owns each event type? How are breaking changes detected and rolled out? | Use versioned, additive event contracts; validate in CI and keep consumers tolerant of unknown fields. EventBridge Schema Registry can catalog contracts but does not replace governance. |
| **Modernization boundary** | Is the application tied to JMS, AMQP, RabbitMQ, or broker transactions? Can it change message contracts now? | Preserve protocols with Amazon MQ/MSK when necessary; otherwise use a strangler path that publishes domain events and moves one consumer at a time to managed, loosely coupled services. |

For every design, draw the complete failure path: producer retry → transport retry → consumer
retry → DLQ/quarantine → controlled replay. Then add event IDs, correlation IDs, backlog/age alarms,
and a named owner for recovery. A diagram that shows only the happy path is not production-ready.

---

## Related Integration Services to Recognize

| Service | Exam clue |
|---------|-----------|
| **Amazon MQ** | Managed ActiveMQ/RabbitMQ broker for migrating existing message-broker apps; not the default for new cloud-native decoupling. |
| **Amazon MSK** | Managed Apache Kafka when protocol compatibility, Kafka tooling, long-lived topics, or MSK Replicator matter more than Kinesis's AWS-native operations. |
| **AppFlow** | No-code/low-code SaaS data transfer flows, such as Salesforce to S3. |
| **AppSync** | GraphQL API with real-time/mobile patterns; usually covered with serverless APIs. |

---

## Prerequisites

- [Compute](../04_compute/README.md) — Lambda and EC2/ASG (the usual producers and consumers)
- [HA & Scaling](../07_ha_scaling/README.md) — Auto Scaling Groups (SQS queue-depth scaling)
- Basic understanding of distributed systems and what a "service" is

---

**Next**: [01_messaging_concepts.md — Messaging & Decoupling Concepts](01_messaging_concepts.md)
