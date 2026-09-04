# 02. OCI Queue, Notifications & Streaming Architecture

## 1. Problem
Building distributed systems in enterprise cloud environments requires high-throughput streaming and decoupled asynchronous coordination. When organizations attempt to run self-managed Apache Kafka or RabbitMQ clusters on cloud compute virtual machines, SRE teams spend hundreds of hours managing ZooKeeper/KRaft quorum nodes, rebalancing consumer group partition assignments, patching Linux operating systems, and recovering from broker disk corruption. Oracle Cloud Infrastructure provides a suite of managed messaging primitives—OCI Queue, OCI Notifications (ONS), and OCI Streaming Service—that eliminate operational maintenance while providing native Kafka API compatibility and sub-second event routing.

## 2. Cloud Concept
### The Triad of OCI Messaging Services
OCI provides three specialized messaging services mapped to distinct workload requirements:

1. **OCI Queue (Serverless At-Least-Once Point-to-Point Queue)**:
   - A fully managed, elastic message queuing service `[Doc: OCI Queue Overview, checked 2026-09-04]`.
   - **Ephemeral Message Locks**: Equivalent to the SQS Visibility Timeout. When a consumer reads a message, OCI Queue locks the message for a configurable window (up to 12 hours), preventing duplicate worker reads.
   - **Native Channels**: A unique OCI capability allowing a single queue to be logically partitioned into thousands of independent message channels (e.g., channel per customer ID), enabling ordered processing within a channel while scaling across channels.
2. **OCI Notifications (ONS - Enterprise Pub/Sub Broadcasting)**:
   - High-throughput Publish/Subscribe messaging engine.
   - Topics broadcast messages simultaneously to thousands of subscribers via HTTPS Webhooks, Email, Slack, SMS, PagerDuty, and direct invocation of **OCI Functions**.
   - Integrates natively with the **OCI Events Service** to broadcast cloud state changes (e.g., compute instance termination, autonomous database backup completed).
3. **OCI Streaming Service (OSS - Apache Kafka-Compatible Log Streaming)**:
   - Real-time, serverless append-only distributed commit log platform.
   - **100% Native Apache Kafka API Compatibility**: Developers can point existing Kafka applications, Kafka Connect plugins, and Kafka Streams microservices directly to OCI Streaming endpoints using standard Kafka SASL/SCRAM authentication without changing a single line of application code `[Doc: OCI Streaming Kafka Compatibility, checked 2026-09-04]`.
   - Partitioned log model: Messages within a partition are strictly ordered by **Offset**.

## 3. Mental Model
Think of OCI messaging services as corporate logistics channels:
- **OCI Queue** is an employee task ticketing queue. 10 customer service agents pull tickets from the pool. Each ticket is handled by one agent.
- **OCI Notifications (ONS)** is a company-wide emergency public address broadcast. The CEO speaks into the microphone (Topic), and speakers in every hallway and office (Subscribers) broadcast the announcement simultaneously.
- **OCI Streaming (Kafka-Compatible)** is an indelible financial ledger tape machine. Transactions are printed continuously onto an unchangeable paper spool. 5 different audit teams can each read the tape at their own speed, rewind the tape to last Tuesday, and replay the transactions without altering the original record.

## 4. Architecture Diagram
```text
OCI MESSAGING & STREAMING PIPELINE:

PRODUCERS: Microservices, IoT Sensors, OCI Audit Logs
                 │
         ┌───────┼───────────────────────────────┐
         │       │                               │
         ▼       ▼                               ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────────────┐
│ OCI QUEUE        │  │ OCI NOTIFICATIONS│  │ OCI STREAMING SERVICE (KAFKA API)│
│ (Point-to-Point) │  │ (Pub/Sub Topics) │  │ (Partitioned Distributed Log)    │
└────────┬─────────┘  └────────┬─────────┘  └────────────────┬─────────────────┘
         │                     │                             │
         │ Ephemeral Locks     ├──► HTTPS Webhook            ├──► Partition 1 (Offset 0..N)
         │ Channels            ├──► PagerDuty / Slack        ├──► Partition 2 (Offset 0..N)
         ▼                     ▼                             ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────────────┐
│ Worker Fleet     │  │ OCI Functions    │  │ Kafka Consumer Groups            │
│ (Load Leveling)  │  │ (Serverless App) │  │ (Real-time Analytics / Flink)    │
└──────────────────┘  └──────────────────┘  └──────────────────────────────────┘
```

## 5. AWS Implementation
Comparing with AWS's streaming and messaging stack:
- AWS provides **Amazon Kinesis Data Streams** and **Amazon Managed Streaming for Apache Kafka (Amazon MSK)**:
  - *Amazon Kinesis Data Streams*: AWS's proprietary streaming engine. Uses shards ($1\text{ MB/s}$ in, $2\text{ MB/s}$ out). Requires AWS SDKs and custom Kinesis Client Library (KCL) integrations.
  - *Amazon MSK*: Managed Apache Kafka running on dedicated virtual machines. Requires choosing instance sizes, managing storage volume expansions, and planning broker capacity.
- In contrast, **OCI Streaming Service combines both concepts**: it is fully serverless (billed per partition/throughput like Kinesis) while speaking native Apache Kafka protocols (like MSK).

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **OCI Streaming Partition Economics**:
  - Each partition provides:
    - **Data Ingress**: Up to **1 MB per second** (or 1,000 write records/sec) `[Doc: OCI Streaming Service Limits, checked 2026-09-04]`.
    - **Data Egress**: Up to **2 MB per second**.
  - Partitions scale elastically: a stream pool with 50 partitions sustains $50\text{ MB/s}$ ($4.32\text{ TB/day}$) of continuous real-time ingestion.
  - **Retention Period**: Data is retained from **24 hours up to 7 days** in high-durability storage across 3 Fault Domains.
- **OCI Queue Channel Mechanics**:
  - Unlike standard queues where consumer concurrency is difficult to coordinate, OCI Queue introduces **Channels**.
  - A producer tags a message with `channelId = "tenant_100"`.
  - A worker can request messages strictly from `channelId = "tenant_100"`, guaranteeing FIFO order for that tenant while allowing 500 other workers to process other tenants in parallel on the same queue!
- **OCI Service Connector Hub (Zero-Code Integration)**:
  - A fully managed serverless orchestration service that moves data between OCI services with zero custom code:
    - Moves events from **OCI Streaming** $\longrightarrow$ **OCI Object Storage** (data lake archiving).
    - Moves logs from **OCI Logging** $\longrightarrow$ **OCI Functions** or third-party SIEM (Splunk/Datadog).

## 7. Configuration
Comparing messaging configuration in Terraform across AWS and OCI:

### AWS Kinesis Stream & SQS Queue (Terraform)
```hcl
# AWS Kinesis Stream with On-Demand Capacity
resource "aws_kinesis_stream" "telemetry_stream" {
  name        = "device-telemetry-stream"
  shard_count = 4

  retention_period = 48 # 48-hour retention
  shard_level_metrics = [
    "IncomingBytes",
    "OutgoingBytes"
  ]

  encryption_type = "KMS"
  kms_key_id      = var.kms_key_arn
}
```

### OCI Streaming Pool & Stream (Terraform)
```hcl
# 1. OCI Stream Pool (Kafka Endpoint & SASL Authentication)
resource "oci_streaming_stream_pool" "telemetry_pool" {
  compartment_id = var.compartment_id
  name           = "telemetry-stream-pool"

  kafka_settings {
    auto_create_topics_enable = false
    log_retention_in_hours    = 48
    num_partitions            = 4
  }

  custom_encryption_key_id = var.vault_key_id
}

# 2. OCI Stream (Kafka Topic equivalent)
resource "oci_streaming_stream" "telemetry_stream" {
  name           = "device-telemetry-topic"
  stream_pool_id = oci_streaming_stream_pool.telemetry_pool.id
  partitions     = 4

  retention_in_hours = 48
}
```

## 8. Data Flow
```text
OCI Streaming Kafka Consumer Group Traversal:
1. Producer publishes IoT message to OCI Stream Pool over Kafka protocol (Port 9092):
   Topic: device-telemetry-topic, Key: "sensor_42", Payload: { temp: 88.5 }
2. OCI Streaming hashes Key "sensor_42" ──► Writes to Partition 2 at Offset 10492.
3. Consumer Group "analytics-workers" (3 workers) active:
   - Worker B assigned Partition 2.
   - Worker B fetches records: poll(Duration.ofMillis(100)).
   - Reads Offset 10492 in 2.1 milliseconds.
4. Worker B commits offset 10493 back to OCI Streaming.
5. Independent Consumer Group "fraud-detection":
   - Can read the exact same message from Offset 10492 without interfering with Analytics!
```

## 9. Security
- **Authentication & Authorization**:
  - OCI Queue and Notifications authenticate using OCI IAM policies, Instance Principals, and OCI Workload Identity.
  - OCI Streaming supports standard Kafka client authentication via **SASL/SCRAM** using OCI Auth Tokens.
- **Private Subnet Endpoints**:
  - Deploy Private Endpoints for OCI Stream Pools and Queues, preventing messaging traffic from traversing the public internet.

## 10. Reliability
- **Triple Fault Domain Durability**:
  - When a message is published to OCI Queue or OCI Streaming, the data payload is synchronously written across **three distinct Fault Domains** within the region before acknowledging success, guaranteeing survival against physical chassis failure.

## 11. Scaling
- **Partition Rebalancing in OCI Streaming**:
  - Consumer groups automatically coordinate partition assignments using standard Kafka protocols.
  - If a worker crashes, OCI Streaming rebalances that worker's assigned partitions to healthy surviving workers in the group.

## 12. Observability
- **Key Metrics in OCI Monitoring**:
  - `PutMessagesBytes` / `GetMessagesBytes`: Ingress and egress throughput per partition.
  - `ConsumerLag`: The difference between the latest offset committed by the producer and the current offset read by the consumer.
  - `QueueVisibleMessages`: Queue depth.

## 13. Cost
- **OCI Streaming Financial Advantage**:
  - OCI charges **\$0.025 per partition-hour** + \$0.0015 per million requests `[Doc: OCI Streaming Pricing, checked 2026-09-04]`.
  - Data transfer within the OCI region is **100% Free**.
  - Compared to running an equivalent 3-node Amazon MSK Kafka cluster (\$250+/month), OCI Streaming costs less than **\$75/month** for equivalent throughput, while completely eliminating operating system patching.

## 14. Failure Modes
- **The Consumer Offset Lag Spiral**: A Kafka consumer worker processing OCI Streaming events begins executing slow external API calls. Consumer lag climbs from 100 messages to 5,000,000 messages. If lag exceeds the configured **retention period (e.g., 24 hours)**, OCI Streaming purges unread records from disk, resulting in permanent data loss!
- **The Unbalanced Partition Key Hotspot**: Producing 100,000 messages per second using a static hardcoded key (e.g., `key = "CONSTANT"`). 100% of traffic routes to Partition 0. Partition 0 throttles at 1 MB/s, while Partitions 1, 2, and 3 sit completely idle.

## 15. Troubleshooting
When Kafka consumers cannot connect to OCI Streaming:
1. **Verify SASL/SCRAM Credentials**:
   - Username format in OCI Streaming must be: `<tenancy_name>/<username>/<stream_pool_ocid>`. Omitting the stream pool OCID results in `AuthenticationException: SASL authentication failed`.
2. **Inspect Consumer Lag**:
   - Query consumer group offsets using standard Kafka CLI:
     ```bash
     kafka-consumer-groups.sh --bootstrap-server <endpoint>:9092 --describe --group analytics-workers
     ```
   - Check the `LAG` column across all partitions.
3. **Verify Partition Throughput Limits**: Check if `PutMessagesThrottled` metric is non-zero.

## 16. Common Mistakes
- **Treating a Stream Like a Queue**: Expecting messages in OCI Streaming to be deleted once read by a consumer. Streams are append-only commit logs; data persists on disk until the retention window expires, regardless of how many consumers read it.
- **Using Small Retentions on Fragile Consumer Fleets**: Setting a 24-hour retention period on critical streaming topics without monitoring consumer lag alarms.

## 17. Trade-offs
| Service | Consumption Model | Ordering | Data Retention | Best Workload |
| :--- | :--- | :--- | :--- | :--- |
| **OCI Queue** | Pull (Queue Workers) | Best-effort / Channel FIFO | Ephemeral (Until deleted) | Work distribution, background jobs |
| **OCI Notifications (ONS)**| Push (HTTP/Functions) | No ordering | Zero retention (Push & drop)| Immediate alerts, fan-out triggers |
| **OCI Streaming (Kafka)** | Pull (Offset-based) | **Strictly ordered by partition**| **24 hours to 7 days** | Event replay, data lake streaming, CDC |

## 18. Interview Questions
1. *What is the architectural distinction between a Message Queue (Amazon SQS / OCI Queue) and an Event Stream (Amazon Kinesis / OCI Streaming)? When is a queue the wrong choice?*
2. *Explain how OCI Streaming achieves 100% Apache Kafka API compatibility. How does authentication and consumer group offset management operate under the hood?*
3. *What are Channels in OCI Queue, and how do they solve the consumer coordination problem in multi-tenant background processing?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "The fundamental architectural distinction between a **Message Queue** (Amazon SQS / OCI Queue) and an **Event Stream** (Amazon Kinesis / OCI Streaming) lies in **Consumption State, Message Deletion, and Consumer Cardinality**:
>
> 1. **Message Queue Architecture (SQS / OCI Queue)**:
>    - **Destructive Pull Model**: A message is placed in a queue with the expectation that **exactly one consumer** will pull it, process it, and **explicitly delete it**.
>    - **State Management**: The queuing engine manages state: it tracks message visibility timeouts, lock leases, and retry counts on behalf of consumers.
>    - **Scaling**: Excellent for **Load Leveling** across dynamic worker pools (e.g., resizing images, processing orders).
>    - *When it Fails*: A queue is the wrong choice when **multiple independent microservices need to read the exact same event stream**, or when consumers need to **rewind and replay historical data** from yesterday to rebuild state.
>
> 2. **Event Stream Architecture (Kinesis / OCI Streaming)**:
>    - **Append-Only Distributed Log**: Messages are appended to ordered, partitioned disk files.
>    - **Non-Destructive Offset Reads**: Reading a message does **not** delete it. Multiple completely independent consumer groups (Analytics, Fraud Detection, Archival) can read the exact same message concurrently at their own speed.
>    - **State Management**: The consumer manages state by tracking its own **Offset** (the sequential pointer in the log). A consumer can reset its offset to zero to replay an entire week's worth of transactions after a code bug fix.
>    - **Ordering**: Messages within a partition are guaranteed to be strictly ordered by offset.
>
> 3. **The Staff Architectural Rule**:
>    - If the goal is **Job Distribution to a single worker pool** $\longrightarrow$ Choose **Queue (SQS / OCI Queue)**.
>    - If the goal is **Event Sourcing, Multi-Consumer Analytics, or Historical Replay** $\longrightarrow$ Choose **Stream (Kinesis / OCI Streaming)**."

## 20. Hands-on Exercise
**Objective**: Create an OCI Streaming pool and stream, and verify message production via the OCI CLI.

### Verification Steps
1. Create an OCI Stream Pool and Stream with 1 partition.
2. Publish a test JSON message to the stream using OCI CLI:
   ```bash
   oci streaming stream message put --stream-id <stream-ocid> \
     --messages '[{"key": "c2Vuc29yXzE=", "value": "eyJ0ZW1wIjogODguNX0="}]'
   ```
   *(Note: Key and Value are base64-encoded strings: `sensor_1` and `{"temp": 88.5}`)*.
3. Create a cursor to consume from the stream starting at the beginning:
   ```bash
   CURSOR=$(oci streaming stream cursor create-cursor --stream-id <stream-ocid> \
     --partition "0" --type TRIM_HORIZON --query "data.value" --output text)
   ```
4. Read the message back:
   ```bash
   oci streaming stream message get --stream-id <stream-ocid> --cursor $CURSOR
   ```
5. Confirm that the message is returned with its assigned Offset and partition timestamp.
