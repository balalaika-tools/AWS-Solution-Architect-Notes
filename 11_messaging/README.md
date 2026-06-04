# Messaging & Events

> Decoupling is one of the most heavily tested architectural ideas on the SAA-C03 exam. This section starts with the *theory* — synchronous vs asynchronous, tight vs loose coupling, and the three messaging models (queue, pub/sub, stream) — then maps each AWS service (SQS, SNS, EventBridge, Kinesis) onto the problem it solves.

[![AWS](https://img.shields.io/badge/AWS-Messaging%20%26%20Events-FF9900.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/messaging/)
[![SQS](https://img.shields.io/badge/SQS-Queue-FF4F8B.svg?logo=amazonsqs&logoColor=white)](https://aws.amazon.com/sqs/)
[![SNS](https://img.shields.io/badge/SNS-Pub%2FSub-E7157B.svg?logo=amazonsns&logoColor=white)](https://aws.amazon.com/sns/)
[![EventBridge](https://img.shields.io/badge/EventBridge-Event%20Bus-FF4F8B.svg?logo=amazoneventbridge&logoColor=white)](https://aws.amazon.com/eventbridge/)
[![Kinesis](https://img.shields.io/badge/Kinesis-Streaming-8C4FFF.svg?logo=amazonaws&logoColor=white)](https://aws.amazon.com/kinesis/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_messaging_concepts.md](01_messaging_concepts.md) | Decoupling theory | Sync vs async, tight vs loose coupling, why decouple, the three models (queue / pub-sub / stream), producers & consumers, delivery semantics, fan-out. |
| [02_sqs.md](02_sqs.md) | Managed queue | Standard vs FIFO, visibility timeout & the duplicate trap, retention, long vs short polling, DLQ + `maxReceiveCount`, 256 KB limit + extended client, delay queues, ASG scaling on queue depth. |
| [03_sns.md](03_sns.md) | Pub/sub | Topics & subscriptions, the SNS→SQS fan-out pattern, message filtering policies, FIFO topics, DLQ, encryption, SNS+SQS vs EventBridge. |
| [04_eventbridge.md](04_eventbridge.md) | Event bus | Event buses, rules & event patterns, targets, scheduled (cron) rules, schema registry, archive/replay, SaaS partner sources, EventBridge vs SNS vs SQS. |
| [05_kinesis.md](05_kinesis.md) | Real-time streaming | Streaming vs batch, Data Streams (shards, partition keys, KCL, enhanced fan-out), Amazon Data Firehose, Managed Service for Apache Flink, Kinesis vs SQS. |

---

## Reading Order

1. **Messaging & Decoupling Concepts** — the vocabulary and the three models everything else builds on. Read this first even if you already know queues.
2. **SQS** — the default managed queue and the Standard vs FIFO / visibility-timeout topics the exam loves.
3. **SNS** — pub/sub and the SNS→SQS fan-out pattern.
4. **EventBridge** — the event router for AWS-service and SaaS events, plus scheduling.
5. **Kinesis** — real-time streaming and how it differs from a queue.

---

## Related Integration Services to Recognize

| Service | Exam clue |
|---------|-----------|
| **Amazon MQ** | Managed ActiveMQ/RabbitMQ broker for migrating existing message-broker apps; not the default for new cloud-native decoupling. |
| **AppFlow** | No-code/low-code SaaS data transfer flows, such as Salesforce to S3. |
| **AppSync** | GraphQL API with real-time/mobile patterns; usually covered with serverless APIs. |

---

## Prerequisites

- [Compute](../04_compute/README.md) — Lambda and EC2/ASG (the usual producers and consumers)
- [HA & Scaling](../07_ha_scaling/README.md) — Auto Scaling Groups (SQS queue-depth scaling)
- Basic understanding of distributed systems and what a "service" is

---

**Next**: [01_messaging_concepts.md — Messaging & Decoupling Concepts](01_messaging_concepts.md)
