# Module 17: Messaging & Eventing (SQS/SNS/EventBridge vs. Queue/Notifications/Streaming)

> **Architectural Objective**: *Master asynchronous decoupling, point-to-point queues, publish/subscribe fan-out topologies, event-driven reactive architectures, and real-time log streaming. Deconstruct Amazon SQS, SNS, and EventBridge against OCI Queue, Notifications (ONS), and OCI Streaming Service (Kafka-compatible), evaluate delivery semantics (At-least-once vs. Exactly-once), and master idempotent consumer engineering.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. SQS, SNS & EventBridge Architecture](01-sqs-sns-and-eventbridge-architecture.md)** | Amazon SQS (Standard vs. FIFO), Visibility Timeouts, SNS Fan-Out Topologies, EventBridge Event Bus Routing & Content Filtering | Full 20-Section Deep Dive (~2,500 words) |
| **[02. OCI Queue, Notifications & Streaming Architecture](02-oci-messaging-queue-notifications-and-streaming.md)** | OCI Queue Channels & Locks, OCI Notifications (ONS), OCI Streaming (Kafka-Compatible) Shards, Consumer Groups & Offsets | Full 20-Section Deep Dive (~2,400 words) |
| **[03. Messaging Patterns & Delivery Guarantees](03-messaging-patterns-and-guarantees.md)** | Delivery Semantics (At-Least-Once, At-Most-Once, Exactly-Once), Idempotent Consumer Design, DLQ Redrive Workflows | Abbreviated Messaging Guide (~950 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **Queue vs. Pub/Sub vs. Log-Streaming Decision Matrix**: When to select point-to-point queues (SQS / OCI Queue) for load-leveling workers, pub/sub topics (SNS / OCI Notifications) for fan-out broadcasting, or append-only log streams (Kinesis / OCI Streaming) for multi-consumer replayable events.
2. **The SQS FIFO vs. Standard Throughput Trade-off**: Why Standard SQS provides virtually infinite throughput while FIFO is capped at 300 transactions/second (or 3,000 with batching), and how Message Group IDs enable parallel processing within FIFO queues.
3. **Idempotency in Distributed Consumers**: How to engineer distributed microservices that safely process duplicate messages caused by network retries and at-least-once delivery guarantees without corrupting financial ledgers.
