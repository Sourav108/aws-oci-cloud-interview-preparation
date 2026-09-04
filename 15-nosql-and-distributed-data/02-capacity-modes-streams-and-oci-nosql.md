# 02. Capacity Modes, Change Data Capture & OCI NoSQL

## 1. Problem
Under-provisioning NoSQL database capacity causes immediate HTTP 400 `ProvisionedThroughputExceededException` errors during traffic spikes, dropping customer transactions. Conversely, over-provisioning static capacity to withstand rare peak spikes results in massive cloud financial waste, paying for idle capacity 24 hours a day. Furthermore, modern microservices architectures require event-driven data propagation: when a record is updated in the database, downstream search indexes (Elasticsearch), caches (Redis), and event brokers (Kafka/Streaming) must be synchronized in real-time. Naive application-layer dual-writes introduce race conditions and silent data corruption. Hyperscale NoSQL engines solve this through dynamic capacity scaling modes and hardware-level Change Data Capture (CDC).

## 2. Cloud Concept
### The Capacity Units Economics (RCU & WCU Math)
In Amazon DynamoDB and OCI NoSQL, throughput is governed by standardized capacity unit equations based on item sizes and consistency models:

$$\text{Item Size Rounded Up to Nearest Unit Boundary}$$

1. **Write Capacity Units (WCU / WU)**:
   - **1 WCU = 1 write per second for an item up to 1 KB** `[Doc: Amazon DynamoDB RCU and WCU Math, checked 2026-09-04]`.
   - A 2.5 KB item rounds up to 3 KB $\longrightarrow$ requires **3 WCU**.
   - Transactional Writes (`TransactWriteItems`) consume **2x WCU** (a 1 KB write requires 2 WCU).
2. **Read Capacity Units (RCU / RU)**:
   - **Strongly Consistent Read**: 1 RCU = 1 read per second for an item up to 4 KB.
   - **Eventually Consistent Read**: 1 RCU = 2 reads per second for an item up to 4 KB (0.5 RCU per read).
   - **Transactional Read**: Consumes **2 RCU per 4 KB**.
   - *Example*: An application performs 1,000 eventually consistent reads per second on 6 KB items:
     $$\text{Size Rounded} = 8\text{ KB} \implies \frac{8\text{ KB}}{4\text{ KB}} \times 0.5\text{ RCU} = 1.0\text{ RCU per read} \implies \mathbf{1,000\text{ RCU required}}.$$

### Provisioned vs. On-Demand Capacity Modes
| Capacity Mode | Scaling Behavior | Ideal Workload | Financial Risk |
| :--- | :--- | :--- | :--- |
| **On-Demand (Pay-Per-Request)** | Instantly accommodates up to **2x previous peak traffic** without manual scaling. | Spiky, unpredictable, or brand new workloads with unknown traffic. | Can become 3x–5x more expensive than Provisioned for steady high traffic. |
| **Provisioned Capacity** | Pre-allocates fixed RCU/WCU pools. Autoscaling adjusts capacity via CloudWatch alarms. | Steady, predictable traffic or predictable diurnal cycles. | Prone to throttling if traffic spikes outpace the 5-minute autoscaling reaction lag. |

### Change Data Capture (CDC): DynamoDB Streams & Global Tables
- **DynamoDB Streams**:
  - An ordered, 24-hour time-stamped log of item-level modifications (INSERT, MODIFY, REMOVE) captured at the physical storage engine layer.
  - Zero performance impact on table RCU/WCU.
  - Stream View Types:
    1. `KEYS_ONLY`: Only the primary key attributes.
    2. `NEW_IMAGE`: The entire item as it appears after the modification.
    3. `OLD_IMAGE`: The entire item as it appeared before the modification.
    4. `NEW_AND_OLD_IMAGES`: Both pre-update and post-update item states (ideal for audit trails and cache invalidation).
- **DynamoDB Global Tables**:
  - Fully managed, active-active multi-region replication built on top of DynamoDB Streams.
  - Replicates writes across multiple AWS regions in **under 1 second**.
  - Conflict Resolution: Employs **Last-Writer-Wins (LWW)** based on internal NTP/vector clock timestamps.

## 3. Mental Model
Think of NoSQL capacity and CDC streams as highway toll plazas:
- **Provisioned Capacity** is renting 10 dedicated lanes at the toll booth. If 10 cars arrive per second, they cruise through. If 50 cars arrive in a sudden flash mob, 40 cars are forced to wait or get turned away.
- **On-Demand Capacity** is an automated magical toll plaza that expands from 10 lanes to 100 lanes in milliseconds as cars approach, billing you a few cents per car.
- **Change Data Capture (Streams)** is a high-speed camera above each toll lane that snaps a photo of every vehicle's license plate as it passes and immediately beams the photo to the police database, traffic monitors, and tax agency simultaneously.

## 4. Architecture Diagram
```text
EVENT-DRIVEN ARCHITECTURE VIA CHANGE DATA CAPTURE (DYNAMODB STREAMS & OCI):

[Client Application] ──► PUT Item (Orders Table)
                               │
                               ▼
┌────────────────────────────────────────────────────────────────────────┐
│ DYNAMODB / OCI NOSQL STORAGE ENGINE                                   │
│ 1. Transaction Committed to Local SSD Storage (Paxos Consensus)        │
│ 2. Asynchronous hardware-level tap writes to STREAM LOG                │
└──────────────────────────────┬─────────────────────────────────────────┘
                               │ Ordered 24-Hour Change Log (Zero DB overhead!)
                               ▼
┌────────────────────────────────────────────────────────────────────────┐
│ DYNAMODB STREAMS / OCI STREAMING CONSUMER ENGINE                       │
│                                                                        │
│   ┌────────────────────────────────┐  ┌──────────────────────────────┐ │
│   │ Consumer 1: Cache Invalidation │  │ Consumer 2: Search Indexing  │ │
│   │ (AWS Lambda / OCI Function)    │  │ (Flushes to OpenSearch/Redis)│ │
│   └────────────────────────────────┘  └──────────────────────────────┘ │
│   ┌────────────────────────────────┐  ┌──────────────────────────────┐ │
│   │ Consumer 3: Global Tables      │  │ Consumer 4: Analytics Stream │ │
│   │ (Replicates to eu-west-1 < 1s) │  │ (Kinesis Data Firehose to S3)│ │
│   └────────────────────────────────┘  └──────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS DynamoDB:
- **DynamoDB Streams Sharding**:
  - Streams are organized into **Shards**. Each shard holds a sequence of records from a specific storage partition.
  - AWS Lambda integrates seamlessly via **Event Source Mapping**: Lambda polls stream shards and triggers function handlers with microsecond latency.
- **DynamoDB Global Tables v2**:
  - Operates across up to 30 AWS regions.
  - Enables local sub-10ms read and write operations for globally distributed users.
  - Writes committed in `us-east-1` are automatically streamed and committed in `ap-southeast-1` in under 1 second.
- **Auto-Scaling Mechanics**:
  - Provisioned capacity auto-scaling uses AWS Application Auto Scaling.
  - CloudWatch alarms track `ConsumedReadCapacityUnits` against target utilization (typically 70%). Takes **5 to 15 minutes** to scale up new partitions.

## 6. OCI Implementation
In Oracle Cloud Infrastructure NoSQL Database Service:
- **Capacity Units Model (Read Units & Write Units)**:
  - Sized via **Read Units (RU)**, **Write Units (WU)**, and **Storage Capacity (GB)** `[Doc: OCI NoSQL Capacity Units, checked 2026-09-04]`:
    - *Read Unit (RU)*: 1 KB of data read per second (eventually consistent) or 2 KB strongly consistent.
    - *Write Unit (WU)*: 1 KB of data written per second.
  - Customers configure independent sliders for Read Units, Write Units, and Disk Storage in GB.
- **OCI On-Demand vs. Provisioned Modes**:
  - Supports switching between On-Demand and Provisioned modes dynamically via OCI CLI or Terraform.
  - On-Demand charges per gigabyte of read/write volume, eliminating capacity management overhead.
- **OCI Change Data Capture & Streaming Integration**:
  - OCI NoSQL tables integrate with **OCI Streaming Service** (Kafka-compatible) and **OCI Events Service**.
  - Row modifications emit CloudEvents payloads that trigger **OCI Functions** for real-time downstream cache invalidation and search re-indexing.
- **OCI Global Active Delivery**:
  - Supports multi-region table replication across OCI commercial regions.
  - Provides active-active cross-region synchronization with automated conflict detection.

## 7. Configuration
Comparing capacity and stream configuration in Terraform across AWS and OCI:

### AWS DynamoDB Table with Streams & Autoscaling (Terraform)
```hcl
# DynamoDB Table with Streams Enabled
resource "aws_dynamodb_table" "orders_table" {
  name             = "OrdersCore"
  billing_mode     = "PROVISIONED"
  read_capacity    = 100
  write_capacity   = 100
  hash_key         = "order_id"

  attribute {
    name = "order_id"
    type = "S"
  }

  # Enable Change Data Capture Stream
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES" # Full audit trail!
}

# Lambda Event Source Mapping (CDC Consumer)
resource "aws_lambda_event_source_mapping" "stream_consumer" {
  event_source_arn  = aws_dynamodb_table.orders_table.stream_arn
  function_name     = var.lambda_function_arn
  starting_position = "LATEST"
  batch_size        = 100
  maximum_retry_attempts = 3
}
```

### OCI NoSQL Table with Provisioned Capacity (Terraform)
```hcl
# OCI NoSQL Table with Provisioned Read/Write Units
resource "oci_nosql_table" "orders_table" {
  compartment_id = var.compartment_id
  name           = "OrdersCore"

  ddl_statement = <<EOF
    CREATE TABLE IF NOT EXISTS OrdersCore (
      order_id STRING,
      customer_id STRING,
      amount DOUBLE,
      status STRING,
      updated_at TIMESTAMP,
      PRIMARY KEY (order_id)
    )
  EOF

  # Provisioned Capacity Sliders
  table_limits {
    capacity_mode      = "PROVISIONED"
    read_units         = 500  # 500 RUs
    write_units        = 250  # 250 WUs
    max_storage_in_gbs = 50   # 50 GB storage
  }
}
```

## 8. Data Flow
```text
DynamoDB Streams / CDC Processing Sequence:
1. Client updates order status: PUT Item (order_123, status: "PAID")
2. Storage node commits update to Paxos consensus group.
3. DynamoDB Stream engine appends change event to stream shard:
   - SequenceNumber: "0000000000049210"
   - ApproximateCreationDateTime: 1725450000
   - Keys: { order_id: "order_123" }
   - OldImage: { status: "PENDING", amount: 150 }
   - NewImage: { status: "PAID",    amount: 150 }
4. AWS Lambda polls stream shard every 100ms:
   - Evaluates OldImage vs. NewImage (detects status changed to "PAID").
   - Dispatches fulfillment event to Amazon EventBridge.
   - Invalidates Redis cache key: cache.del("order:order_123").
```

## 9. Security
- **IAM Policy for Streams Access**:
  - Reading a DynamoDB Stream requires explicit IAM permissions: `dynamodb:DescribeStream`, `dynamodb:GetRecords`, `dynamodb:GetShardIterator`, `dynamodb:ListStreams`.
  - Stream data is encrypted at rest using the exact same KMS key protecting the base table.

## 10. Reliability
- **Stream Retention & Shard Expiration**:
  - Records in DynamoDB Streams are retained for **strictly 24 hours**. After 24 hours, unread records are permanently purged.
  - If a consumer Lambda function crashes continuously on a poison pill record, the stream consumer lags. If lag reaches 24 hours, data is lost!
  - *Reliability Rule*: Always configure `BisectBatchOnFunctionError: true` and a **Dead-Letter Queue (DLQ)** on the Lambda event source mapping.

## 11. Scaling
- **The On-Demand Scaling Ceiling**:
  - DynamoDB On-Demand capacity can instantly accommodate up to **double the previous peak traffic**.
  - If a table previously peaked at 10,000 WCU, On-Demand will scale up to 20,000 WCU instantly.
  - However, if traffic surges instantly from 1,000 to 50,000 WCU without gradual ramp-up, the table will throttle until partitions can be split!

## 12. Observability
- **Monitoring Stream Consumer Lag**:
  - CloudWatch metric: `IteratorAgeMilliseconds`.
  - Measures the age of the oldest record in the stream being processed by the consumer.
  - If `IteratorAgeMilliseconds` is increasing monotonically, consumers are failing to keep pace with ingestion volume.

## 13. Cost
- **Financial Calculation: On-Demand vs. Provisioned**:
  - A table handles steady traffic: 1,000 WCU and 2,000 RCU continuously 24/7.
  - **On-Demand Cost**:
    - Writes: $1,000 \times 3,600 \times 730 = 2.628\text{ billion writes} \times \$1.25/\text{million} = \mathbf{\$3,285/\text{month}}$.
    - Reads: $2,000 \times 3,600 \times 730 = 5.256\text{ billion reads} \times \$0.25/\text{million} = \mathbf{\$1,314/\text{month}}$.
    - Total: **\$4,599/month**.
  - **Provisioned Cost (with Auto-Scaling)**:
    - Writes: $1,000\text{ WCU} \times 730\text{ hrs} \times \$0.00065 = \mathbf{\$474.50/\text{month}}$.
    - Reads: $2,000\text{ RCU} \times 730\text{ hrs} \times \$0.00013 = \mathbf{\$189.80/\text{month}}$.
    - Total: **\$664.30/month**.
  - *Staff FinOps Insight*: For steady workloads, **Provisioned Capacity is 85% cheaper than On-Demand** (\$664 vs. \$4,599), saving almost \$4,000 per month on a single table!

## 14. Failure Modes
- **The Dual-Write Race Condition Corruption**: An application updates DynamoDB and then attempts to update Elasticsearch. If the app server crashes between the two calls, Elasticsearch contains stale data permanently. *Remediation: Never perform dual-writes in application code. Use DynamoDB Streams to asynchronously synchronize external search systems.*
- **The Poison Pill Stream Deadlock**: A malformed event in a DynamoDB Stream causes the consumer Lambda function to throw an unhandled exception. Because streams guarantee in-order delivery within a shard, Lambda retries the exact same batch forever, blocking all subsequent records until the 24-hour retention window purges the shard.

## 15. Troubleshooting
When DynamoDB Stream consumers fall behind:
1. **Inspect CloudWatch `IteratorAgeMilliseconds`**:
   - If $> 60,000\text{ms}$ (1 minute), the consumer is lagging.
2. **Increase Parallelization Factor**:
   - In AWS Lambda event source mapping, set `ParallelizationFactor` (between 1 and 10).
   - Allows up to 10 concurrent Lambda invocations per stream shard, multiplying consumer throughput 10x while preserving item-level partition ordering!
3. **Verify Function Errors**: Inspect Lambda `Errors` metric to detect unhandled exceptions.

## 16. Common Mistakes
- **Leaving Production Tables on On-Demand Forever**: Launching a production system on On-Demand during initial testing and forgetting to convert steady workloads to Provisioned Capacity, wasting tens of thousands of dollars annually.
- **Assuming Global Tables Eliminates Conflict Resolution**: Believing Global Tables provides multi-region distributed locking. Global Tables uses Last-Writer-Wins; concurrent writes to the same item in two regions will overwrite each other based on the latest timestamp.

## 17. Trade-offs
| Dimension | On-Demand Mode | Provisioned Mode (Auto-Scaling) |
| :--- | :--- | :--- |
| **Operational Overhead**| Zero management (True serverless) | Requires capacity planning & alert tuning |
| **Instant Spike Resilience**| High (Scales instantly to 2x peak) | Moderate (5–15 minute autoscaling lag) |
| **Unit Cost** | High (\$1.25 / million writes) | **Ultra-Low (Up to 85% cheaper)** |
| **Throttling Risk** | Near zero (Except sudden 10x surges)| High during instantaneous spikes |

## 18. Interview Questions
1. *A client application writes to Amazon DynamoDB and then synchronously updates an in-memory Redis cache. Why is this application dual-write pattern an anti-pattern, and how do you redesign it using Change Data Capture (DynamoDB Streams / OCI Streaming)?*
2. *Calculate the exact RCU and WCU required for an application performing 500 strongly consistent reads per second of 7 KB items and 250 writes per second of 3.2 KB items.*
3. *What is `IteratorAgeMilliseconds` in AWS Lambda / DynamoDB Streams, and what architectural knobs do you turn to resolve a lagging consumer?*

## 19. Interview Answer
**Exemplary Answer to Question 2**:
> "To calculate the exact Read and Write Capacity Units required, we break down each operation based on cloud storage unit boundaries:
>
> 1. **Write Capacity Unit (WCU) Calculation**:
>    - **Rule**: 1 WCU delivers 1 write per second for items up to 1 KB.
>    - **Item Size**: 3.2 KB. In DynamoDB, item sizes round up to the next 1 KB boundary:
>      $$\text{Rounded Write Size} = \lceil 3.2\text{ KB} \rceil = \mathbf{4\text{ KB}}.$$
>    - Each write requires 4 WCU.
>    - **Target Rate**: 250 writes per second:
>      $$\text{Total Required WCU} = 250 \times 4\text{ WCU} = \mathbf{1,000\text{ WCU}}.$$
>
> 2. **Read Capacity Unit (RCU) Calculation**:
>    - **Rule**: 1 RCU delivers 1 Strongly Consistent read per second for items up to 4 KB.
>    - **Item Size**: 7 KB. In DynamoDB, read item sizes round up to the next 4 KB boundary:
>      $$\text{Rounded Read Size} = \lceil 7\text{ KB} / 4\text{ KB} \rceil \times 4\text{ KB} = \mathbf{8\text{ KB}}.$$
>    - For a **Strongly Consistent Read**, an 8 KB item requires:
>      $$\frac{8\text{ KB}}{4\text{ KB}} \times 1\text{ RCU} = \mathbf{2\text{ RCU per read}}.$$
>    - **Target Rate**: 500 strongly consistent reads per second:
>      $$\text{Total Required RCU} = 500 \times 2\text{ RCU} = \mathbf{1,000\text{ RCU}}.$$
>
> 3. **Final Provisioning Plan**:
>    - The table must be provisioned with **1,000 WCU** and **1,000 RCU** to sustain this workload without throttling."

## 20. Hands-on Exercise
**Objective**: Enable DynamoDB Streams and inspect stream event records using the AWS CLI.

### Verification Steps
1. Enable streams on an existing table:
   ```bash
   aws dynamodb update-table --table-name OrdersCore \
     --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
   ```
2. Insert a test item:
   ```bash
   aws dynamodb put-item --table-name OrdersCore --item '{"order_id": {"S": "ord_99"}, "status": {"S": "PENDING"}}'
   ```
3. Update the item to "PAID":
   ```bash
   aws dynamodb update-item --table-name OrdersCore --key '{"order_id": {"S": "ord_99"}}' \
     --update-expression "SET #s = :new_status" --expression-attribute-names '{"#s": "status"}' \
     --expression-attribute-values '{":new_status": {"S": "PAID"}}'
   ```
4. Query the stream shard iterator:
   ```bash
   STREAM_ARN=$(aws dynamodb describe-table --table-name OrdersCore --query "Table.LatestStreamArn" --output text)
   SHARD_ID=$(aws dynamodbstreams describe-stream --stream-arn $STREAM_ARN --query "StreamDescription.Shards[0].ShardId" --output text)
   ITERATOR=$(aws dynamodbstreams get-shard-iterator --stream-arn $STREAM_ARN --shard-id $SHARD_ID --shard-iterator-type LATEST --query "ShardIterator" --output text)
   aws dynamodbstreams get-records --shard-iterator $ITERATOR
   ```
5. Confirm that the stream record displays both the `OldImage` (`status: PENDING`) and `NewImage` (`status: PAID`).
