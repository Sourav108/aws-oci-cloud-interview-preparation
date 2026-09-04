# 01. DynamoDB Architecture & Single-Table Data Modeling

## 1. Problem
Traditional relational database systems scale vertically. As datasets grow to hundreds of gigabytes or billions of rows, executing complex relational joins (`JOIN users ON orders.user_id = users.id`) across massive tables requires disk-bound table scans, lock contention, and exponential query degradation. When high-traffic internet applications attempt to handle 100,000 queries per second on relational databases, query latency jumps from 5 milliseconds to 500 milliseconds, and connection pools collapse. Distributed NoSQL databases—such as Amazon DynamoDB and OCI NoSQL Database Service—deliver predictable, single-digit millisecond latency at any scale by replacing relational joins with pre-computed, partition-based key-value and document storage.

## 2. Cloud Concept
### Distributed Storage Partitioning & Consensus
- **The Partition**:
  - The fundamental unit of data storage and throughput allocation in DynamoDB and OCI NoSQL.
  - A partition is a physical storage allocation backed by SSDs that holds up to **10 GB of data** and delivers up to **1,000 Write Capacity Units (WCU)** and **3,000 Read Capacity Units (RCU)** `[Doc: Amazon DynamoDB Limits, checked 2026-09-04]`.
  - As a table grows beyond 10 GB or requires more throughput, the storage engine automatically splits the table across dozens or thousands of partitions.
- **Request Routers & Consistent Hashing**:
  - When a client sends a read or write request, the request lands on a fleet of stateless **Request Routers**.
  - The router runs an MD5/SHA-256 consistent hash function on the item's **Partition Key** to determine which physical storage partition holds that item.
- **Paxos Consensus Replication**:
  - Within each physical partition, data is replicated across **3 storage nodes spanning 3 independent Availability Zones / Fault Domains**.
  - One node acts as the **Leader Replica**, while the other two act as **Follower Replicas**.
  - Write operations require a **Paxos Quorum (2 of 3 nodes)** before acknowledging success, guaranteeing durability against complete AZ failure.

### Keys & Indexing Primitives
1. **Partition Key (PK / Hash Key)**: Determines the physical partition where the item is stored. All items with the same partition key reside on the same storage node.
2. **Sort Key (SK / Range Key)**: Sorts items within the partition physically on disk in ascending binary/alphanumeric order (B-tree structure).
3. **Composite Primary Key**: `Primary Key = (Partition Key + Sort Key)`. Enables rich 1-to-Many query patterns using range expressions (`begins_with`, `between`, `<`, `>`).
4. **Global Secondary Indexes (GSIs)**:
   - Creates a completely independent secondary index with a new Partition Key and optional Sort Key.
   - Replicated **asynchronously** from the base table.
   - Has its own provisioned or on-demand throughput capacity.
5. **Local Secondary Indexes (LSIs)**:
   - Shares the same Partition Key as the base table, but defines a different Sort Key.
   - Replicated **synchronously** within the same physical partition.
   - *Severe Operational Limitation*: Restricts the total size of all items sharing that partition key to a maximum of **10 GB** `[Doc: Amazon DynamoDB Secondary Indexes, checked 2026-09-04]`! Because of this rigid constraint, LSIs are rarely used in modern cloud architectures in favor of GSIs.

### The Single-Table Design Paradigm
In relational databases, you normalize data across dozens of separate tables (`Users`, `Orders`, `OrderItems`, `Products`). In hyperscale NoSQL:
- Normalize data into a **Single Physical Table** using generic primary key attribute names: `PK` (String) and `SK` (String).
- Pre-compute access patterns by grouping related parent and child entities under the same Partition Key (**Item Collections**).
- Retrieve an entire user profile, their latest 5 orders, and their shipping addresses in a **single, lightning-fast round-trip query** (`Query(PK="USER#100")`) without a single database join!

## 3. Mental Model
Think of distributed NoSQL data modeling as an automated warehouse sorting facility:
- **Partition Key** is the labeled shipping bin number (Bin 42). All items for Customer 42 are thrown into Bin 42.
- **Sort Key** is the alphabetical file divider inside Bin 42 (`PROFILE`, `ORDER#20260901`, `ORDER#20260902`).
- **Relational Joins** is an employee walking across a 5-mile warehouse floor searching through 50 different rooms to collect 5 items for an order.
- **Single-Table Design** is placing all 5 items inside the exact same bin ahead of time. The employee opens Bin 42 and grabs everything in 1 second.

## 4. Architecture Diagram
```text
DYNAMODB STORAGE ENGINE & SINGLE-TABLE ITEM COLLECTION:

CLIENT REQUEST: Query(PK = "USER#101", SK begins_with "ORDER#")
       │
       ▼
┌────────────────────────────────────────────────────────────────────────┐
│ STATELESS REQUEST ROUTER FLEET                                         │
│ * Hashes PK "USER#101" ──► Maps to Partition 3                         │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ Routes directly to Leader Node
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│ PHYSICAL PARTITION 3 (SSD Storage Fleet - Max 10 GB)                   │
│                                                                        │
│   PAXOS REPLICATION REPLICA GROUP:                                     │
│   [Storage Node A - AZ1] ◄──► [Node B (Leader) - AZ2] ◄──► [Node C AZ3]│
│   (Synchronous Paxos Quorum 2/3 required for write commits)            │
│                                                                        │
│   PHYSICAL ON-DISK B-TREE SORTED STORAGE:                              │
│   ┌──────────┬──────────────────┬────────────────────────────────────┐ │
│   │ PK       │ SK               │ Attributes (Data Payload)          │ │
│   ├──────────┼──────────────────┼────────────────────────────────────┤ │
│   │ USER#101 │ METADATA         │ Name: "Sourav", Email: "s@test.com"│ │
│   │ USER#101 │ ORDER#2026-09-01 │ Total: $150, Status: "SHIPPED"     │ │
│   │ USER#101 │ ORDER#2026-09-03 │ Total: $85,  Status: "PENDING"     │ │
│   │ USER#102 │ METADATA         │ Name: "Alex",   Email: "a@test.com"│ │
│   └──────────┴──────────────────┴────────────────────────────────────┘ │
│                                                                        │
│   * Single disk seek reads User 101's profile and both orders in 2ms!  │
│   * ZERO RELATIONAL JOINS! ZERO TABLE LOCKS!                           │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS DynamoDB:
- **Item Size Limits**:
  - Maximum individual item size is **400 KB** (including attribute names and binary values) `[Doc: Amazon DynamoDB Limits, checked 2026-09-04]`.
  - For items $> 400\text{ KB}$ (e.g., large documents, images), store the binary payload in **Amazon S3** and store the S3 object URL in DynamoDB.
- **Consistency Options**:
  - *Eventually Consistent Reads (Default)*: Queries any of the 3 storage nodes. May return stale data if a write committed within the last 500ms. Consumes **0.5 RCU per 4 KB**.
  - *Strongly Consistent Reads*: Queries the Leader node or verifies quorum. Guarantees latest committed data. Consumes **1.0 RCU per 4 KB**.
  - *Transactional Reads/Writes*: ACID transactions across up to 100 items or 4 MB of data. Consumes **2x RCU/WCU**.
- **DynamoDB Accelerator (DAX)**:
  - Fully managed, in-memory cache cluster deployed directly in front of DynamoDB.
  - Slashes read latency from single-digit milliseconds down to **sub-millisecond microsecond latency** ($< 200\mu\text{s}$) with zero application query rewrites.

## 6. OCI Implementation
In Oracle Cloud Infrastructure NoSQL Database Service:
- **Table Architecture & Data Types**:
  - OCI NoSQL Database is a fully managed cloud database service designed for predictable low-latency document, columnar, and key-value workloads `[Doc: OCI NoSQL Database Overview, checked 2026-09-04]`.
  - **Native Multi-Model Engine**: Supports structured relational tabular data, flexible schemaless **JSON documents**, and raw key-value pairs in the same table.
  - Allows querying JSON attributes directly using SQL syntax:
    ```sql
    SELECT u.profile.address.city FROM Users u WHERE u.profile.age > 25;
    ```
- **Primary Key Structure**:
  - In OCI NoSQL, the primary key consists of one or more **Shard Keys** (equivalent to DynamoDB's Partition Key) and optional **Identity/Ordering Columns** (equivalent to Sort Keys).
  - Items sharing the same Shard Key are co-located on the same physical storage partition.
- **Capacity Sizing Primitives**:
  - Sized via **Read Units (RU)** and **Write Units (WU)**:
    - 1 Read Unit = 1 KB read per second (Eventually Consistent) or 2 KB strongly consistent.
    - 1 Write Unit = 1 KB write per second.
  - Supports both **Provisioned Capacity** and **On-Demand Capacity**.
- **Secondary Indexing in OCI NoSQL**:
  - Allows creating secondary indexes on top-level columns and deeply nested fields inside JSON documents!
  - OCI manages index synchronization automatically across distributed shards.

## 7. Configuration
Comparing table definitions in Terraform across AWS and OCI:

### AWS DynamoDB Single-Table Design (Terraform)
```hcl
# AWS DynamoDB Single-Table with GSI and TTL
resource "aws_dynamodb_table" "enterprise_single_table" {
  name         = "EnterpriseCoreTable"
  billing_mode = "PAY_PER_REQUEST" # On-Demand auto-scaling!
  hash_key     = "PK"
  range_key    = "SK"

  attribute {
    name = "PK"
    type = "S"
  }

  attribute {
    name = "SK"
    type = "S"
  }

  attribute {
    name = "GSI1_PK"
    type = "S"
  }

  attribute {
    name = "GSI1_SK"
    type = "S"
  }

  # Global Secondary Index for inverted access patterns
  global_secondary_index {
    name            = "GSI1"
    hash_key        = "GSI1_PK"
    range_key       = "GSI1_SK"
    projection_type = "ALL"
  }

  # Automated Time-to-Live (TTL) for ephemeral sessions
  ttl {
    attribute_name = "expires_at"
    enabled        = true
  }

  point_in_time_recovery {
    enabled = true # Continuous PITR backups!
  }
}
```

### OCI NoSQL Table with JSON Indexing (Terraform)
```hcl
# OCI NoSQL Table with On-Demand Capacity
resource "oci_nosql_table" "enterprise_table" {
  compartment_id = var.compartment_id
  name           = "EnterpriseCoreTable"

  # Table DDL: Shard Key + Order Key + JSON payload
  ddl_statement = <<EOF
    CREATE TABLE IF NOT EXISTS EnterpriseCoreTable (
      pk STRING,
      sk STRING,
      payload JSON,
      created_at TIMESTAMP,
      PRIMARY KEY (SHARD(pk), sk)
    )
  EOF

  table_limits {
    capacity_mode      = "ON_DEMAND"
    max_storage_in_gbs = 100
  }
}

# Secondary Index on nested JSON attribute
resource "oci_nosql_index" "email_index" {
  table_name_or_id = oci_nosql_table.enterprise_table.id
  name             = "idx_user_email"
  keys {
    column_name = "payload.email"
    json_path   = "payload.email"
    json_field_type = "STRING"
  }
}
```

## 8. Data Flow
```text
Single-Table Query Execution Path:
1. Application queries: "Fetch User #101 profile and all active orders"
2. SDK dispatches Query request:
   KeyConditionExpression: PK = :pk AND SK begins_with :prefix
   ExpressionAttributeValues: { ":pk": "USER#101", ":prefix": "ORDER#" }
3. Request Router hashes "USER#101" ──► Identifies Partition 3 Leader.
4. Storage Node initiates single B-tree index seek on disk:
   - Scans contiguous blocks on local NVMe SSD.
   - Reads:
     * USER#101 / ORDER#2026-09-01
     * USER#101 / ORDER#2026-09-03
5. Data returned in a single HTTP payload in 2.4 milliseconds.
6. ZERO joins performed; total RCU consumed = 1.5 RCU.
```

## 9. Security
- **Item-Level & Attribute-Level IAM Access Control**:
  - DynamoDB supports fine-grained IAM policies using the `dynamodb:LeadingKeys` condition key.
  - Allows an application user to execute queries **strictly against items where the Partition Key equals their own User ID** (`${www.amazon.com:user_id}`), preventing multi-tenant data leaks at the database layer!
- **KMS Encryption at Rest**:
  - Every table, GSI, and automated backup is encrypted by default using AES-256 with AWS KMS or OCI Vault Customer Managed Keys.

## 10. Reliability
- **Point-in-Time Recovery (PITR)**:
  - Provides continuous backups of DynamoDB and OCI NoSQL tables.
  - Restores to any second within the past 35 days with zero impact on production table throughput.

## 11. Scaling
- **Horizontal Elasticity**:
  - DynamoDB partitions scale indefinitely. Tables holding 50 TB of data scale across 5,000 physical partitions, supporting **millions of concurrent read and write operations per second** with flat latency.

## 12. Observability
- **CloudWatch Metrics for DynamoDB**:
  - `ConsumedReadCapacityUnits` / `ConsumedWriteCapacityUnits`.
  - `ProvisionedReadCapacityUnits` / `ProvisionedWriteCapacityUnits`.
  - `ThrottledRequests` / `ReadThrottleEvents`: High values indicate hot partitions or exhausted capacity.
- **OCI NoSQL Metrics**: Track `ReadUnitsConsumed`, `WriteUnitsConsumed`, and `StorageUtilization` in OCI Monitoring.

## 13. Cost
- **Provisioned vs. On-Demand Pricing (AWS)**:
  - *On-Demand*: \$1.25 per million write request units, \$0.25 per million read request units `[Doc: Amazon DynamoDB Pricing, checked 2026-09-04]`. Best for unpredictable or spiky workloads.
  - *Provisioned*: \$0.00065 per WCU-hour, \$0.00013 per RCU-hour. Up to **70% cheaper** for steady, predictable production traffic.
- **OCI NoSQL Pricing**:
  - Extremely cost-effective: First 133 million reads, 133 million writes, and 25 GB storage per month are **100% Free** under the OCI Always Free tier `[Doc: OCI Free Tier NoSQL, checked 2026-09-04]`.

## 14. Failure Modes
- **The Hot Partition Throttling Bottleneck**: Storing real-time IoT device logs with `PK = Date (2026-09-04)`. All 100,000 devices write to the exact same partition key simultaneously. Because a single partition is capped at **1,000 WCU**, DynamoDB rejects 99,000 writes with **`ProvisionedThroughputExceededException`**, even if the overall table has 100,000 WCU provisioned!
- **The GSI Write Throttle Drag**: A base table has 10,000 WCU provisioned, but its Global Secondary Index (GSI) is configured with only 500 WCU. Because GSI updates are asynchronous but backpressured, **throttling on the GSI will throttle writes on the base table** to prevent the index from lagging infinitely!

## 15. Troubleshooting
When DynamoDB returns HTTP 400 `ProvisionedThroughputExceededException`:
1. **Determine Throttling Distribution**:
   - Inspect CloudWatch `ThrottledRequests` by `Operation` (PutItem vs. Query).
2. **Identify Hot Partition Keys via CloudWatch Contributor Insights**:
   - Enable **DynamoDB Contributor Insights**.
   - CloudWatch identifies the top 5 most accessed partition keys in real-time, immediately isolating the runaway hot key.
3. **Inspect GSI Metrics**: Verify whether `ConsumedReadCapacityUnits` on any GSI has reached 100% saturation.

## 16. Common Mistakes
- **Using Scan Instead of Query**: Executing `Scan` operations in production application APIs. `Scan` reads every single item in the entire physical table across all partitions, burning thousands of RCUs and taking seconds to finish. Production APIs must strictly use **`Query`** or **`GetItem`**.
- **Creating LSIs on Large Production Tables**: Adding an LSI to a table. Once the item collection for a partition key exceeds 10 GB, DynamoDB physically blocks all subsequent writes to that partition key with `ItemCollectionSizeLimitExceededException`!

## 17. Trade-offs
| Feature | Amazon DynamoDB | OCI NoSQL Database | Relational SQL (Aurora/ATP) |
| :--- | :--- | :--- | :--- |
| **Data Model** | Key-Value / JSON Document | Key-Value / Relational Table / JSON | Normalized Relational Tables |
| **Joins** | **Zero Joins (Single-Table Design)**| Zero Joins (Pre-indexed) | Native complex SQL Joins |
| **Max Scale** | Millions of RPS; 100s of TB | Millions of RPS; 10s of TB | Tens of thousands of RPS |
| **Latency** | 1–4ms p99 at any scale | 1–5ms p99 at any scale | 5–50ms (CPU/Lock dependent) |
| **Schema Flexibility**| Schemaless (Attributes dynamic)| Schemaless JSON or strict DDL | Strict relational schema DDL |

## 18. Interview Questions
1. *What is Single-Table Design in Amazon DynamoDB? Why does relational normalization fail at hyperscale, and how do you model 1-to-Many relationships in a single physical table?*
2. *Explain the architectural differences between a Global Secondary Index (GSI) and a Local Secondary Index (LSI). Why do Staff Cloud Architects strongly avoid LSIs in production?*
3. *How does OCI NoSQL Database Service handle JSON document querying, and how does its primary key sharding architecture compare to Amazon DynamoDB?*

## 19. Interview Answer
**Exemplary Answer to Question 2**:
> "The architectural distinctions between a Global Secondary Index (GSI) and a Local Secondary Index (LSI) in DynamoDB are profound, and understanding them is why Staff Architects strictly avoid LSIs in modern system design:
>
> 1. **Partitioning & Synchronization Mechanics**:
>    - **Local Secondary Index (LSI)**:
>      - An LSI **must share the exact same Partition Key** as the base table; it only allows defining an alternative Sort Key.
>      - Data in an LSI is updated **synchronously** within the exact same physical storage partition as the base item.
>      - Reads against an LSI can be configured as **Strongly Consistent**.
>      - It shares the base table's provisioned RCU and WCU capacity.
>    - **Global Secondary Index (GSI)**:
>      - A GSI can define a **completely different Partition Key and Sort Key**, allowing you to invert or reshape your data access patterns.
>      - Data is replicated **asynchronously** across independent storage partitions.
>      - Reads are strictly **Eventually Consistent**.
>      - A GSI maintains its own dedicated, independent provisioned or on-demand throughput capacity.
>
> 2. **The Fatal Architectural Flaw of LSIs (The 10 GB Partition Trap)**:
>    - The fatal limitation of an LSI is that **it enforces a hard 10 GB limit on the aggregate size of all items sharing the same Partition Key** (an Item Collection).
>    - If you design an application where a `customer_id` is the Partition Key, and that customer's transaction history grows to 10.01 GB, DynamoDB **rejects all subsequent writes** to that customer with `ItemCollectionSizeLimitExceededException`!
>    - Furthermore, **an LSI cannot be added or deleted after table creation**; it must be defined at initial table provisioning. If your access patterns evolve, you cannot drop an unused LSI.
>
> 3. **The Staff Architectural Recommendation**:
>    - In modern cloud architectures, **we mandate GSIs over LSIs in 100% of production scenarios**.
>    - GSIs have no 10 GB partition limit, can be created or deleted dynamically online at any time without downtime, and scale independently from the base table, delivering true hyperscale resilience."

## 20. Hands-on Exercise
**Objective**: Model a 1-to-Many relationship in a single DynamoDB table and query parent and child items in a single request.

### Verification Steps
1. Create a DynamoDB table with `PK` (String) and `SK` (String).
2. Insert parent user profile and two child orders:
   ```bash
   aws dynamodb put-item --table-name EnterpriseCoreTable --item '{"PK": {"S": "USER#500"}, "SK": {"S": "PROFILE"}, "Name": {"S": "Jane"}}'
   aws dynamodb put-item --table-name EnterpriseCoreTable --item '{"PK": {"S": "USER#500"}, "SK": {"S": "ORDER#2026-09-01"}, "Amount": {"N": "120"}}'
   aws dynamodb put-item --table-name EnterpriseCoreTable --item '{"PK": {"S": "USER#500"}, "SK": {"S": "ORDER#2026-09-03"}, "Amount": {"N": "45"}}'
   ```
3. Execute a single `Query` call retrieving all items for User 500:
   ```bash
   aws dynamodb query --table-name EnterpriseCoreTable \
     --key-condition-expression "PK = :pk" \
     --expression-attribute-values '{":pk": {"S": "USER#500"}}'
   ```
4. Confirm that the single query returns all 3 records in under 3ms with zero relational joins.
