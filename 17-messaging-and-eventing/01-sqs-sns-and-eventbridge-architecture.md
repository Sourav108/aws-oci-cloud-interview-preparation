# 01. SQS, SNS & EventBridge Architecture

## 1. Problem
In tightly coupled synchronous microservices, Service A makes direct HTTP calls to Service B, which calls Service C. If Service C experiences an outage or a latency spike, TCP connection pools back up across the entire chain, exhausting thread pools and collapsing the upstream API gateway. Furthermore, when an event occurs (e.g., `OrderPlaced`), multiple independent downstream domains (Billing, Fraud Detection, Inventory, Shipping, Marketing Analytics) all require this data. Requiring the Order Service to synchronously execute separate HTTP calls to 5 different microservices multiplies network failures and turns a simple checkout into an operational nightmare. Cloud messaging and eventing primitives solve this by decoupling producers from consumers.

## 2. Cloud Concept
### The Three Core Cloud Messaging Topologies
Cloud architectures employ three distinct messaging paradigms based on consumer topology and delivery requirements:

```text
1. POINT-TO-POINT QUEUE (Amazon SQS / OCI Queue)
[Producer] ──► [Queue] ──► Pulled by ONE Worker in a pool (Load Leveling)

2. PUBLISH / SUBSCRIBE FAN-OUT (Amazon SNS / OCI Notifications)
               ┌──► [Subscription 1: Queue A]
[Producer] ──► [Topic] ──┼──► [Subscription 2: Queue B] (1-to-Many Broadcasting)
               └──► [Subscription 3: Webhook]

3. CENTRAL EVENT BUS ROUTER (Amazon EventBridge / OCI Events)
               ┌──(Rule: region=US)──► [Target 1: US Processor]
[Producers] ──► [Event Bus] ──┼──(Rule: amount>500)──► [Target 2: Fraud Lambda]
               └──(Rule: all)────────► [Target 3: CloudWatch Logs]
```

### Point-to-Point Queue Mechanics (Amazon SQS)
- **Standard Queues**:
  - *Virtually Unlimited Throughput*: Supports hundreds of thousands of transactions per second.
  - *At-Least-Once Delivery*: Messages may occasionally be delivered more than once.
  - *Best-Effort Ordering*: Messages are generally delivered in the order sent, but order is not mathematically guaranteed.
- **FIFO Queues (First-In, First-Out)**:
  - *Strict Ordering Guarantee*: Messages are delivered strictly in the exact order received.
  - *Exactly-Once Processing*: Duplicate messages sent within a 5-minute deduplication window (based on `MessageDeduplicationId` or content hashing) are automatically discarded.
  - *Throughput Ceiling*: Capped at **300 transactions per second** (or up to **3,000 transactions per second with batching**) `[Doc: Amazon SQS Quotas, checked 2026-09-04]`.
  - *Message Group ID*: Enables horizontal parallel processing within a FIFO queue. Messages sharing the same `MessageGroupId` are processed strictly in order, while distinct group IDs are processed concurrently by different worker threads.
- **The Visibility Timeout**:
  - When a consumer pulls a message (`ReceiveMessage`), the message is **not deleted** from the queue.
  - SQS hides the message from other consumers for the duration of the **Visibility Timeout** (default: 30 seconds, maximum: 12 hours).
  - *Success*: Consumer processes message and explicitly calls `DeleteMessage`.
  - *Failure*: Consumer crashes or times out. When the Visibility Timeout expires, the message automatically becomes visible again for another worker to retry.

### Publish/Subscribe Fan-Out (Amazon SNS)
- A push-based 1-to-Many broadcasting service.
- **The Fan-Out Pattern**: An SNS topic publishes to multiple SQS queues simultaneously. Each microservice (Billing, Inventory) owns its own private SQS queue subscribed to the topic.
- **Message Filtering Policies**: Allows subscribers to define JSON filter policies so consumers receive strictly the messages relevant to their domain (e.g., only orders where `status = "APPROVED"` and `amount > 1000`).

### Centralized Event Bus (Amazon EventBridge)
- Serverless event bus routing events using declarative **JSON Content-Based Filtering Rules**.
- Features **Schema Discovery & Registry**: Automatically infers JSON schemas and generates strongly-typed code bindings (Java/TypeScript/Python).
- **Archive & Replay**: Allows recording all events passing through the bus to durable storage and replaying historical events into development or staging environments to debug production issues.

## 3. Mental Model
Think of messaging topologies as commercial parcel shipping:
- **SQS (Queue)** is an office inbox tray. Five mailroom clerks take envelopes from the pile. Each envelope is processed by exactly one clerk.
- **SNS (Pub/Sub Fan-Out)** is a printing press sending daily newspapers. The press prints one story, and 10,000 subscribers all receive their own personal copy on their doorstep.
- **EventBridge (Event Bus)** is a smart automated international customs sorting facility. It inspects the package label (JSON payload): packages containing food go to Agriculture inspection, packages containing electronics go to Customs Clearance, and packages flagged as suspicious go to Security.

## 4. Architecture Diagram
```text
ENTERPRISE PUB/SUB FAN-OUT WITH EVENTBRIDGE & SQS QUEUES:

[Order Service API]
        │
        ├──► 1. Publishes Event: OrderCreated ($1,200, EU Region)
        │
        ▼
┌────────────────────────────────────────────────────────────────────────┐
│ AMAZON EVENTBRIDGE / OCI NOTIFICATIONS BUS                             │
│                                                                        │
│   RULE 1: { "detail-type": ["OrderCreated"] }                          │
│   ├──► Target A: [Billing SQS Queue] ──► [Billing Microservice]        │
│   │                                                                    │
│   RULE 2: { "detail": { "amount": [{ "numeric": [">", 1000] }] } }     │
│   ├──► Target B: [Fraud Audit SQS Queue] ──► [Fraud Engine Lambda]     │
│   │                                                                    │
│   RULE 3: { "detail": { "region": ["EU"] } }                           │
│   └──► Target C: [GDPR Compliance Queue] ──► [Compliance Worker]       │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS:
- **SQS Long Polling vs. Short Polling**:
  - *Short Polling (`WaitTimeSeconds = 0`)*: Samples a subset of storage servers. May return empty responses even if messages exist, burning billable API requests.
  - *Long Polling (`WaitTimeSeconds = 20`)*: The API call waits up to 20 seconds for messages to arrive. Slashes empty receives by 90% and reduces AWS bills.
- **Visibility Timeout Heartbeats (`ChangeMessageVisibility`)**:
  - If a worker pulls a batch of video files and processing takes 4 minutes, but the Visibility Timeout is 60 seconds, SQS will release the message to a second worker at second 61!
  - *The Architectural Fix*: The worker runs a background heartbeat thread that periodically calls `ChangeMessageVisibility(newTimeout = 60)` every 30 seconds while processing is ongoing.
- **Dead-Letter Queue (DLQ) Redrive**:
  - AWS SQS supports automated **DLQ Redrive Tasks**: allows inspecting poisoned messages, fixing bugs in code, and redriving the DLQ messages back to the source queue with a single click.

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **OCI Queue**:
  - A fully managed, serverless, at-least-once message queue service `[Doc: OCI Queue Architecture, checked 2026-09-04]`.
  - Supports **Ephemeral Message Locks** (equivalent to SQS Visibility Timeout).
  - Features native **Channels**: allows logically grouping messages within a single queue, enabling consumers to listen exclusively to specific channels for ordered processing.
  - Native Dead-Letter Queue (DLQ) support and automated poison pill isolation.
- **OCI Notifications (ONS)**:
  - High-throughput Publish/Subscribe messaging platform.
  - Endpoints include: HTTPS Webhooks, Email, Slack/PagerDuty, SMS, and **OCI Functions**.
- **OCI Streaming Service (The Kafka-Compatible Engine)**:
  - High-throughput, real-time event streaming service fully compatible with the **Apache Kafka API** `[Doc: OCI Streaming Overview, checked 2026-09-04]`.
  - Applications can use standard Kafka producer/consumer client libraries pointing to OCI Streaming endpoints.
  - Partitioned log architecture: Each partition provides **1 MB/s write throughput** (1,000 records/sec) and **2 MB/s read throughput**.

## 7. Configuration
Comparing messaging architectures in Terraform across AWS and OCI:

### AWS SQS FIFO Queue with DLQ & SSE-KMS (Terraform)
```hcl
# Dead-Letter Queue (DLQ)
resource "aws_sqs_queue" "orders_dlq" {
  name                      = "orders-dlq.fifo"
  fifo_queue                = true
  message_retention_seconds = 1209600 # 14 days!
  kms_master_key_id         = var.kms_key_arn
}

# Main SQS FIFO Queue
resource "aws_sqs_queue" "orders_queue" {
  name                        = "orders-processing.fifo"
  fifo_queue                  = true
  content_based_deduplication = true
  visibility_timeout_seconds  = 60
  receive_wait_time_seconds   = 20 # Enforces Long Polling!

  # Dead-Letter Queue Redrive Policy
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.orders_dlq.arn
    maxReceiveCount     = 3 # Quarantine poison pill after 3 fails!
  })

  kms_master_key_id = var.kms_key_arn
}
```

### OCI Queue with Dead-Letter Queue (Terraform)
```hcl
# OCI Dead-Letter Queue
resource "oci_queue_queue" "orders_dlq" {
  compartment_id     = var.compartment_id
  display_name       = "orders-dlq"
  retention_in_seconds = 1209600
}

# OCI Queue with Visibility Lock & Channel Support
resource "oci_queue_queue" "orders_queue" {
  compartment_id     = var.compartment_id
  display_name       = "orders-processing-queue"
  visibility_in_seconds = 60
  timeout_in_seconds    = 20 # Long polling timeout!

  # Dead-Letter Queue routing
  dead_letter_queue_delivery_count = 3
  custom_encryption_key_id         = var.vault_key_id
}
```

## 8. Data Flow
```text
The Fan-Out Message Lifecycle:
1. Client POSTs: /api/v1/orders ──► API Gateway returns HTTP 202 Accepted.
2. Producer writes JSON to SNS Topic: "OrderPlaced"
3. SNS Fan-Out Engine evaluates subscriptions:
   - Evaluates filter policy: Does message match?
   - Pushes message copy to SQS Queue 1 (Billing Service).
   - Pushes message copy to SQS Queue 2 (Inventory Service).
4. Billing Worker pulls message from Queue 1 (Visibility Timeout = 60s).
5. Billing Worker executes charging logic.
6. Worker calls: DeleteMessage(ReceiptHandle).
7. Message permanently removed from Queue 1; Queue 2 processes independently!
```

## 9. Security
- **Strict IAM Resource Policies on Queues**:
  - By default, only the queue creator has access.
  - Enforce least-privilege IAM policies permitting SNS or EventBridge to invoke `sqs:SendMessage` with condition keys locking access to the specific source topic ARN.
- **KMS Envelope Encryption**:
  - Messages are encrypted using AES-256 before being written to persistent disk storage.

## 10. Reliability
- **Poison Pill Quarantine via DLQ**:
  - If a message contains malformed JSON that crashes the worker runtime, SQS retries it.
  - With `maxReceiveCount = 3`, after the 3rd consecutive crash, SQS moves the toxic message to the **Dead-Letter Queue (DLQ)**, preventing consumer crash loops.

## 11. Scaling
- **Standard SQS Scaling Elasticity**:
  - Amazon SQS Standard queues scale automatically to hundreds of thousands of messages per second without manual sharding or capacity provisioning.
  - Worker fleets scale dynamically using **AWS EC2 Auto Scaling** based on the CloudWatch metric `ApproximateNumberOfMessagesVisible`.

## 12. Observability
- **Key Queue Metrics**:
  - `ApproximateNumberOfMessagesVisible`: Queue depth (messages waiting for consumers).
  - `ApproximateNumberOfMessagesNotVisible`: Messages currently in-flight being processed by workers.
  - `ApproximateAgeOfOldestMessage`: Age of the oldest unconsumed message. If this climbs above 5 minutes, worker capacity is under-provisioned!

## 13. Cost
- **Queue Economics**:
  - AWS SQS: First 1 million requests per month are **Free**; \$0.40 per million requests thereafter `[Doc: Amazon SQS Pricing, checked 2026-09-04]`.
  - Enforcing **Long Polling (`WaitTimeSeconds = 20`)** slashes billable empty `ReceiveMessage` API calls by over 90%, reducing monthly messaging costs from hundreds of dollars to a few dollars.

## 14. Failure Modes
- **The Short Polling Bill Shock**: An application deploys 50 worker containers executing `ReceiveMessage` in an infinite `while(true)` loop with default Short Polling (`WaitTimeSeconds = 0`). During the weekend when the queue is completely empty, the 50 workers execute millions of empty API requests per hour, generating thousands of dollars in wasted cloud bills for zero messages! *Remediation: Mandatory Long Polling (`WaitTimeSeconds = 20`).*
- **The In-Flight Visibility Timeout Duplication Race**: A worker pulls an image resizing task. The task takes 75 seconds to process. Because the queue Visibility Timeout was set to 60 seconds, SQS makes the message visible again at second 61. A second worker pulls the message and begins processing it in parallel, resulting in duplicate processing and resource waste.

## 15. Troubleshooting
When SQS messages appear stuck or consumers report duplicates:
1. **Inspect `ApproximateAgeOfOldestMessage`**:
   - If age is climbing, check whether worker processes are crashing or throwing unhandled exceptions before calling `DeleteMessage`.
2. **Verify Visibility Timeout vs. Processing Duration**:
   - The queue's Visibility Timeout must be at least **6x the average processing duration** of your worker task!
3. **Inspect Dead-Letter Queue**:
   ```bash
   aws sqs get-queue-attributes --queue-url <dlq-url> --attribute-names ApproximateNumberOfMessages
   ```

## 16. Common Mistakes
- **Forgetting to Delete Processed Messages**: Assuming that pulling a message automatically removes it from the queue. SQS only hides the message; you must explicitly call `DeleteMessage` using the `ReceiptHandle` after processing completes.
- **Using FIFO Queues for Massively Scalable Telemetry**: Attempting to push 50,000 IoT sensor events per second through an SQS FIFO queue without batching. FIFO queues hard-cap at 300 msgs/s. High-throughput streaming must use **Amazon Kinesis** or **OCI Streaming**.

## 17. Trade-offs
| Dimension | Amazon SQS (Queue) | Amazon SNS (Pub/Sub) | Amazon EventBridge (Bus) |
| :--- | :--- | :--- | :--- |
| **Model** | Pull (Polling by workers) | Push (HTTP/Lambda/SQS) | Push (Rules routing) |
| **Consumer Cardinality**| 1 Consumer per message | 1-to-Many Fan-Out | 1-to-Many Content Filtered |
| **Ordering** | FIFO guaranteed (300/s) | No ordering guarantees | No ordering guarantees |
| **Latency** | 10–50ms (Polling interval)| Sub-30ms | 50–200ms (Rule evaluation)|
| **Content Filtering** | Basic message attributes | Basic attribute filtering | **Deep nested JSON filtering** |

## 18. Interview Questions
1. *Explain the architectural mechanics of the SQS Visibility Timeout. What catastrophic failure occurs if a worker takes longer to process a message than the configured Visibility Timeout, and how do you prevent it in production?*
2. *Compare Amazon SQS Standard Queues against FIFO Queues across throughput limits, ordering guarantees, and deduplication mechanics. When is FIFO an anti-pattern?*
3. *How does the Pub/Sub Fan-Out pattern (SNS fanning out to multiple SQS queues) deliver architectural decoupling in event-driven microservices?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "The **SQS Visibility Timeout** is the fundamental cloud primitive that guarantees at-least-once message processing without distributed locking:
>
> 1. **The Visibility Timeout Lifecycle**:
>    - When a consumer worker issues a `ReceiveMessage` API call, SQS retrieves the message from storage and immediately starts the **Visibility Timeout timer** (e.g., 60 seconds).
>    - Crucially: **the message is NOT deleted from the queue**. SQS merely hides the message from other workers.
>    - If the worker successfully processes the message within 60 seconds, it calls `DeleteMessage(ReceiptHandle)`. SQS permanently purges the message.
>
> 2. **The Catastrophic Failure Sequence (Worker Processing > Visibility Timeout)**:
>    - Suppose a worker pulls an order processing task, but downstream database latency causes execution to take **75 seconds**.
>    - At second 60, the SQS Visibility Timeout expires. SQS assumes Worker 1 crashed and died.
>    - SQS immediately makes the message visible again in the queue.
>    - Worker 2 pulls the exact same message and begins processing it, while Worker 1 is still in the middle of executing its database transaction.
>    - *Consequences*: Duplicate processing, race conditions, potential double-billing of customer credit cards, and wasted compute resources.
>
> 3. **The Production Engineering Mitigations**:
>    - **Rule of Thumb**: Configure the queue Visibility Timeout to at least **6x the average processing duration** of the task (e.g., 300 seconds for a 45-second task).
>    - **Implement Dynamic Visibility Heartbeats (`ChangeMessageVisibility`)**:
>      For tasks with variable execution times (e.g., video rendering), the worker launches a background heartbeat thread that periodically calls `ChangeMessageVisibility(newTimeout = 60)` every 30 seconds while the work continues, continuously extending the lock until processing completes.
>    - **Mandatory Consumer Idempotency**: All consumers must implement idempotent write keys (e.g., using database unique constraints or DynamoDB conditional writes) to guarantee that duplicate deliveries cause zero side effects."

## 20. Hands-on Exercise
**Objective**: Demonstrate SQS Long Polling and Dead-Letter Queue quarantine behavior using the AWS CLI.

### Verification Steps
1. Create a Dead-Letter Queue:
   ```bash
   aws sqs create-queue --queue-name TestDLQ
   ```
2. Create a Main Queue with `maxReceiveCount = 2` pointing to TestDLQ:
   ```bash
   DLQ_ARN=$(aws sqs get-queue-attributes --queue-url <dlq-url> --attribute-names QueueArn --query "Attributes.QueueArn" --output text)
   aws sqs create-queue --queue-name MainQueue --attributes \
     "{\"RedrivePolicy\":\"{\\\"deadLetterTargetArn\\\":\\\"$DLQ_ARN\\\",\\\"maxReceiveCount\\\":\\\"2\\\"}\",\"VisibilityTimeout\":\"5\"}"
   ```
3. Send a test message:
   ```bash
   aws sqs send-message --queue-url <main-queue-url> --message-body "Poison Pill Payload"
   ```
4. Read the message twice without deleting it (simulate 2 worker crashes):
   ```bash
   aws sqs receive-message --queue-url <main-queue-url>
   sleep 6
   aws sqs receive-message --queue-url <main-queue-url>
   sleep 6
   ```
5. Inspect the Dead-Letter Queue:
   ```bash
   aws sqs receive-message --queue-url <dlq-url>
   ```
6. Confirm that the message was automatically quarantined to the DLQ after 2 failed attempts, preserving the main queue from crash loops.
