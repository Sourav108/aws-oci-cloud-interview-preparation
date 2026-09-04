# 03. Hot Partitions & NoSQL Query Anti-Patterns

## 1. Problem
In distributed NoSQL systems, provisioning massive aggregate throughput (e.g., 50,000 WCU) does not guarantee that your workload can push 50,000 writes per second. If data access patterns funnel disproportionate traffic toward a single Partition Key (such as a celebrity user profile, a flash sale product ID, or a date string), that single underlying physical storage partition hits its hardware ceiling ($1,000\text{ WCU} / 3,000\text{ RCU}$ in DynamoDB). The storage node throttles requests, rejecting traffic with HTTP 400 `ProvisionedThroughputExceededException`, even though 99% of the overall table's capacity sits completely idle.

## 2. Cloud Concept: The Hot Partition Architecture
```text
HOT PARTITION BOTTLENECK (Low-Cardinality Key):
All 10,000 writes target PK = "STATUS_ACTIVE"
       │
       ▼ Request Router: MD5("STATUS_ACTIVE") ──► Maps to Partition 1 ONLY!
┌────────────────────────────────────────────────────────────────────────┐
│ PARTITION 1 (Hard Hardware Limit: 1,000 WCU / 3,000 RCU)              │
│ 10,000 incoming writes ──► 1,000 writes committed ──► 9,000 THROTTLED! │
└────────────────────────────────────────────────────────────────────────┘
│ Partition 2 (Idle) │ Partition 3 (Idle) │ Partition 4 (Idle) │ (Wasted!)│
└────────────────────┴────────────────────┴────────────────────┴──────────┘

WRITE SHARDING / KEY SALTING SOLUTION:
Distributes writes across PK = "STATUS_ACTIVE.{0..9}"
       │
       ▼ Request Router: Hashes salt suffix across 10 partitions evenly!
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Partition 1: #0  │  │ Partition 2: #1  │  │ Partition 3: #2  │ ... (All 10,000
│ 1,000 writes/sec │  │ 1,000 writes/sec │  │ 1,000 writes/sec │      succeed!)
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

- **Partition Limits**: Every physical partition is an independent storage unit capped at **1,000 WCU and 3,000 RCU** `[Doc: Amazon DynamoDB Partition Limits, checked 2026-09-04]`.
- **Adaptive Capacity**: DynamoDB features **Adaptive Capacity**, which dynamically borrows unused capacity from quiet partitions to boost a hot partition up to 3,000 RCU / 1,000 WCU. However, **Adaptive Capacity cannot exceed the physical hardware ceiling of a single partition**! If traffic exceeds 1,000 WCU, throttling is guaranteed without architectural sharding.

## 3. Partition Key Salting & Write Sharding Engineering
When an entity naturally receives massive write volume, use **Write Sharding (Key Salting)**:
1. **Randomized Suffix**: Append a random integer between 0 and $N$ to the partition key:
   $$\text{Salted PK} = \text{"DEVICE\_INGEST\_" } + \text{random}(0, 19)$$
   Writes are distributed across 20 distinct physical partitions, expanding write throughput from 1,000 WCU to **20,000 WCU**!
2. **Calculated / Hash Suffix**: When data must be queryable by sub-attribute (e.g., User ID), use a deterministic hash:
   $$\text{Salted PK} = \text{"ORDERS\_" } + (\text{user\_id} \pmod{10})$$

## 4. OCI NoSQL Shard Distribution
In OCI NoSQL:
- Tables use **Composite Shard Keys**: `PRIMARY KEY (SHARD(region, tenant_id), log_id)`.
- The `SHARD` expression explicitly tells OCI NoSQL which attributes to feed into the consistent hashing function to distribute records across storage nodes evenly.
- If an application uses an un-sharded low-cardinality primary key, OCI NoSQL throttles writes on that shard.

## 5. Production Failure Modes: Scan Cascades & Pagination Traps
- **The Accidental Table Scan Meltdown**: A developer writes a dashboard query: `Scan(FilterExpression: "status = 'FAILED'")`. The table contains 500 million records ($100\text{ GB}$). Even though only 5 records match the filter, DynamoDB must physically read all 100 GB from disk, consuming **25,000 RCU** in seconds, exhausting the table capacity, and starving production checkout APIs!
  - *Fix*: Never scan. Create a Global Secondary Index (GSI) on `status`.
- **Deep Pagination Pagination Memory Exhaustion**: Iterating through 100,000 items via sequential `LastEvaluatedKey` calls in a single synchronous API request.

## 6. Troubleshooting & Contributor Insights
1. **Enable CloudWatch Contributor Insights**:
   - Analyzes real-time metrics to identify the exact Partition Keys and Sort Keys generating the highest percentage of read/write throttling events.
2. **Inspect Throttling Metrics**:
   ```bash
   aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name WriteThrottleEvents \
     --dimensions Name=TableName,Value=OrdersCore --start-time 2026-09-04T00:00:00Z \
     --end-time 2026-09-04T01:00:00Z --period 60 --statistics Sum
   ```

## 7. Senior Interview Question & Defense
**Question**: *Your e-commerce application is hosting a global flash sale for a single viral product. 100,000 users attempt to purchase this single item simultaneously, generating 50,000 writes per second. How do you design the DynamoDB / NoSQL architecture to prevent hot partition collapse while guaranteeing atomic inventory decrement?*

**Staff-Level Defense**:
> "Attempting to write 50,000 transactions directly to a single item (`PK = PRODUCT#123`) will instantly fail because a single DynamoDB partition has a hard physical hardware ceiling of **1,000 WCU**:
>
> 1. **The Write Sharding Architecture (Distributed Inventory Counters)**:
>    - Instead of maintaining a single inventory counter item, we split the product's inventory across **50 independent Inventory Shards**:
>      - `PK: PRODUCT#123#COUNTER`, `SK: SHARD#00` (Holds 2,000 units)
>      - `PK: PRODUCT#123#COUNTER`, `SK: SHARD#01` (Holds 2,000 units)
>      - ...
>      - `PK: PRODUCT#123#COUNTER`, `SK: SHARD#49` (Holds 2,000 units)
>    - Because each counter shard has a unique Partition/Sort key, the 50 shards distribute evenly across distinct physical storage partitions.
>
> 2. **Atomic Inventory Decrement via Conditional Writes**:
>    - When a purchase request arrives, the application randomly picks a shard number: `shard_id = random(0, 49)`.
>    - It executes an atomic `UpdateItem` with a **Conditional Expression**:
>      ```text
>      UpdateItem(
>        Key: { PK: "PRODUCT#123#COUNTER", SK: "SHARD#" + shard_id },
>        UpdateExpression: "SET stock = stock - 1",
>        ConditionExpression: "stock > 0"
>      )
>      ```
>    - If that specific shard is temporarily exhausted (`stock = 0`), the application catches the `ConditionalCheckFailedException` and immediately tries another random shard.
>
> 3. **The Architectural Result**:
>    - 50 shards $\times$ 1,000 WCU = **50,000 writes per second sustained throughput** with zero hot partition throttling.
>    - Stock is decremented with absolute ACID transactional consistency, guaranteeing zero inventory over-selling."
