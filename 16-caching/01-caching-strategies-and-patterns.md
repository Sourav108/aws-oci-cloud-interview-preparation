# 01. Caching Strategies & Design Patterns

## 1. Problem
Relational and NoSQL databases store data on persistent block or flash storage. Even with high-performance NVMe SSDs, reading data requires filesystem traversals, buffer pool synchronization, and query parsing, resulting in response times between 2 and 50 milliseconds. Under massive concurrent load (e.g., 500,000 queries per second during a Black Friday sale), databases hit CPU, I/O, and connection limits, driving p99 latencies into multi-second timeouts. In-memory distributed caching—using engines like Redis and Memcached deployed on AWS ElastiCache or OCI Cache with Redis—resolves this by storing pre-computed data structures directly in RAM, slashing read latencies to **sub-millisecond speeds ($< 500\mu\text{s}$)**. However, selecting the wrong caching pattern leads to severe data inconsistency, memory bloat, and stale data bugs.

## 2. Cloud Concept
### In-Process vs. Distributed Remote Caching
- **In-Process Cache (Local Memory / Guava / Caffeine)**:
  - Cache lives directly inside the application process heap (RAM).
  - *Ultra-fast*: Nanosecond access latency ($< 1\mu\text{s}$); zero network serialization overhead.
  - *Downside*: Memory is duplicated across hundreds of microservice instances; each instance maintains a different cache state, leading to inconsistent user experiences.
- **Distributed Remote Cache (Redis / Memcached)**:
  - Cache runs on a dedicated, shared cluster across multiple compute instances.
  - *Single Source of Truth*: All application instances share the exact same cached data.
  - *Sub-millisecond*: Microsecond network hop ($< 500\mu\text{s}$) over high-speed private cloud networks.

### The 4 Core Caching Design Patterns

```text
1. CACHE-ASIDE (LAZY LOADING)
[App] ──► 1. Check Cache ──(Miss)──► 2. Read DB ──► 3. Write Cache ──► Return

2. READ-THROUGH
[App] ──► 1. Read Cache ───────────► (Cache library auto-loads from DB on miss)

3. WRITE-THROUGH
[App] ──► 1. Write Cache ──(Sync)──► 2. Synchronous Write to DB ──► Both updated!

4. WRITE-BEHIND (WRITE-BACK)
[App] ──► 1. Write Cache (< 1ms ACK!) ──► Returns success immediately
               │ (Asynchronous Queue)
               ▼
          2. Batch write flushed to DB 5 seconds later (Risk of data loss on crash!)
```

1. **Cache-Aside (Lazy Loading)**:
   - The application orchestrates all caching logic.
   - On read: The app checks the cache. On a **Cache Hit**, it returns data immediately. On a **Cache Miss**, it queries the database, writes the result into the cache with a Time-to-Live (TTL), and returns.
   - *Advantages*: Only requested data is cached; resilient to cache node failure (app falls back to the database).
   - *Disadvantages*: Cache miss latency penalty (3 round trips: Cache Read $\to$ DB Read $\to$ Cache Write).
2. **Read-Through**:
   - The application treats the cache as the main data store. The caching layer or client plugin automatically fetches missing records from the underlying database and populates itself.
3. **Write-Through**:
   - The application writes data to the cache, and the caching layer synchronously writes the data to the database before confirming success.
   - *Advantages*: Guarantees data consistency between cache and database; fresh reads never miss.
   - *Disadvantages*: Higher write latency (incurs both cache and DB write time).
4. **Write-Behind (Write-Back)**:
   - The application writes strictly to the in-memory cache, which acknowledges the write in $< 1\text{ms}$.
   - The cache engine asynchronously queues the modifications and writes them in batches to the database seconds or minutes later.
   - *Advantages*: Extreme write performance and absorption of massive write spikes.
   - *Disadvantages*: If the cache node crashes before flushing its queue to the database, **committed data is permanently lost**!

### Cache Eviction Policies
When an in-memory cache reaches its maximum memory ceiling (`maxmemory`), it must evict existing keys to accommodate new writes:

| Eviction Policy | Algorithm Mechanics | Production Best Practice |
| :--- | :--- | :--- |
| **`volatile-lru`** | Evicts the Least Recently Used keys that have an explicit **TTL** set. | Standard choice when caching ephemeral session/query data. |
| **`allkeys-lru`** | Evicts the Least Recently Used keys across the **entire keyspace**, even without TTL. | **The Industry Standard** for pure caching tiers. |
| **`volatile-lfu`** | Evicts the Least Frequently Used keys (counts hit frequency) with TTL. | Best when certain historical keys are accessed repeatedly. |
| **`allkeys-lfu`** | Evicts Least Frequently Used keys across all keys. | Protects popular viral items from being evicted by new items. |
| **`noeviction`** | Never evicts data. Rejects all new writes with `OOM command not allowed`! | Default in raw Redis; disastrous for caches! |

## 3. Mental Model
Think of caching patterns as personal note-taking:
- **Cache-Aside** is keeping a sticky note on your computer screen. When someone asks you for the office Wi-Fi password, you look at the sticky note. If it's not there, you walk over to the IT handbook in the file cabinet, write it on a new sticky note, and stick it to your monitor.
- **Write-Through** is updating the employee phone directory by typing the new phone number into your notebook and simultaneously updating the master company spreadsheet before hanging up the phone.
- **Write-Behind** is scribbling a customer's order on a napkin during a chaotic restaurant rush and planning to enter it into the accounting register after your shift ends. If the napkin falls into the sink, the order is lost forever.

## 4. Architecture Diagram
```text
┌────────────────────────────────────────────────────────────────────────┐
│                   CACHE-ASIDE READ & INVALIDATION FLOW                 │
│                                                                        │
│   READ WORKFLOW (Cache-Aside):                                         │
│   [Client] ──► 1. GET /products/42                                     │
│                     │                                                  │
│                     ▼                                                  │
│   [Application Server: Node.js / Go / Java]                            │
│         │                                                              │
│         ├──► 2. redisClient.get("prod:42")                             │
│         │         │                                                    │
│         │         ├─► CACHE HIT (< 1ms): Return payload to client!     │
│         │         │                                                    │
│         │         └─► CACHE MISS:                                      │
│         │               │                                              │
│         │               ▼ 3. Query DB (SELECT * FROM products WHERE id=42)
│         │             [Primary SQL / NoSQL Database]                   │
│         │               │ Returns row (Takes 25ms)                     │
│         │               ▼                                              │
│         ├──► 4. redisClient.setex("prod:42", 3600, jsonPayload)        │
│         │       (Cached for 1 hour with allkeys-lru eviction)          │
│         │                                                              │
│         ▼ 5. Return response to Client                                 │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS Amazon ElastiCache:
- **Engine Options**:
  - *ElastiCache for Redis*: Supports complex data structures (Strings, Hashes, Lists, Sets, Sorted Sets, Bitmaps, HyperLogLogs, Geospatial indexes), transactions, Pub/Sub, and replication.
  - *ElastiCache for Memcached*: Multithreaded, simple key-value store. Best for multi-threaded memory caching where data structures are not required.
- **ElastiCache Serverless**:
  - Automatically scales memory and compute capacity in fractions of a second based on active application traffic.
  - Eliminates the need to plan cluster node sizes or manage manual failover testing.
- **Parameter Groups**:
  - Configures `maxmemory-policy` (e.g., `allkeys-lru`).
  - Sets `reserved-memory-percent` (typically 25%) to ensure Redis background snapshotting (`BGSAVE`) has sufficient unallocated RAM to fork without triggering out-of-memory kernel panics.

## 6. OCI Implementation
In Oracle Cloud Infrastructure Cache with Redis:
- **OCI Cache with Redis Service**:
  - A fully managed, in-memory caching service built directly on open-source Redis `[Doc: OCI Cache with Redis Overview, checked 2026-09-04]`.
  - Deployed directly into private regional subnets of your VCN.
  - Provides a single dedicated private IP endpoint per node, completely shielded from internet routing.
- **Fault Domain High Availability**:
  - Automatically distributes primary and replica cache nodes across **OCI Fault Domains** within an Availability Domain.
  - If Fault Domain 1 suffers a power or top-of-rack switch outage, OCI Cache executes automated failover to the replica in Fault Domain 2 in **under 10 seconds** with zero data loss.
- **OCI Flexible Compute Sizing**:
  - Allows provisioning cache memory precisely tuned to application requirements without paying for fixed AWS node family tiers.
  - Native integration with OCI Monitoring and OCI Logging.

## 7. Configuration
Comparing cache provisioning in Terraform across AWS and OCI:

### AWS ElastiCache Redis Replication Group (Terraform)
```hcl
# AWS ElastiCache Redis Cluster with Multi-AZ Failover
resource "aws_elasticache_replication_group" "prod_cache" {
  replication_group_id       = "prod-redis-cache"
  description                = "Production Redis Replication Group"
  node_type                  = "cache.r6g.large"
  port                       = 6379
  parameter_group_name       = aws_elasticache_parameter_group.redis_params.name
  subnet_group_name          = var.cache_subnet_group_name
  security_group_ids         = [var.cache_security_group_id]

  # High Availability: 1 Primary + 1 Replica across 2 AZs
  num_cache_clusters         = 2
  automatic_failover_enabled = true
  multi_az_enabled           = true

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token
}

# Parameter group enforcing allkeys-lru eviction
resource "aws_elasticache_parameter_group" "redis_params" {
  family = "redis7"
  name   = "prod-redis7-params"

  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru" # Protects against OOM crashes!
  }
}
```

### OCI Cache with Redis Cluster (Terraform)
```hcl
# OCI Cache with Redis Cluster
resource "oci_redis_redis_cluster" "prod_cache" {
  compartment_id     = var.compartment_id
  display_name       = "prod-oci-redis"
  node_count         = 2 # 1 Primary, 1 Replica
  node_memory_in_gbs = 16
  software_version   = "V7_0_5"
  subnet_id          = var.private_subnet_id
  nsg_ids            = [var.cache_nsg_id]

  # Distributed across Fault Domains for HA
  cluster_mode = "NONSHARDED"
}
```

## 8. Data Flow
```text
Write-Through vs. Write-Behind Execution:
Write-Through Flow:
1. App executes: writeUserData(user_123, payload)
2. App writes to Redis: SET user:123 payload
3. App writes to PostgreSQL: UPDATE users SET ... WHERE id = 123
4. As soon as BOTH acknowledge, success returned to caller.
   Total latency: 1ms (Redis) + 15ms (SQL) = 16ms.

Write-Behind Flow:
1. App executes: writeUserData(user_123, payload)
2. App writes to Redis: SET user:123 payload
3. Cache returns SUCCESS immediately (< 1ms elapsed!).
4. Background worker polls Redis dirty list every 5 seconds.
5. Worker flushes 500 batched updates to PostgreSQL in a single bulk INSERT.
   Total client latency: 0.8ms!
```

## 9. Security
- **In-Transit and At-Rest Encryption**:
  - Enforce TLS encryption on all Redis client-to-server connections.
  - Authenticate using `AUTH` tokens or IAM-based authentication (AWS IAM / OCI Identity).
- **Network Segmentation**:
  - Redis contains no internal user authorization or fine-grained table access control.
  - Deploy strictly into **isolated private subnets** with security groups restricting port 6379 access exclusively to backend application instances.

## 10. Reliability
- **Avoiding Fork-Induced OOM Crashes**:
  - When Redis executes background snapshots (`BGSAVE`) or replicates data to a replica, the Linux kernel invokes the `fork()` system call.
  - Linux uses Copy-on-Write (CoW). If the database experiences heavy write traffic during snapshotting, CoW memory usage doubles!
  - If free system RAM is exhausted, the Linux Out-Of-Memory (OOM) killer instantly terminates the Redis process.
  - *Reliability Rule*: Always configure `reserved-memory-percent = 25` in ElastiCache parameter groups.

## 11. Scaling
- **Vertical vs. Horizontal Scaling**:
  - Redis is primarily single-threaded for command execution. Sizing up to an instance with 64 vCPUs will **not** accelerate single-key execution!
  - To scale beyond a single node's CPU limits, you must scale horizontally using **Redis Cluster (Cluster Mode Enabled)**.

## 12. Observability
- **Key CloudWatch / OCI Cache Metrics**:
  - `EngineCPUUtilization`: Measures the CPU usage of the single-threaded Redis engine core. If $> 80\%$, the cluster is bottlenecked.
  - `CacheHits` / `CacheMisses`: Used to calculate **Cache Hit Ratio**:
    $$\text{Hit Ratio} = \frac{\text{CacheHits}}{\text{CacheHits} + \text{CacheMisses}}$$
    Production target: $\mathbf{> 90\%}$.
  - `Evictions`: Number of keys purged due to memory pressure. A sudden spike indicates insufficient RAM sizing.

## 13. Cost
- In-memory RAM is the most expensive storage tier in cloud computing (\$0.015 to \$0.05 per GB-hour).
- Optimize costs by:
  1. Storing compressed JSON strings (gzip/snappy) instead of verbose raw text.
  2. Setting explicit TTLs on all keys to prevent dead keys from lingering indefinitely.
  3. Using **ElastiCache Data Tiering** (utilizes NVMe SSDs to store infrequently accessed keys, reducing memory costs by up to 60%).

## 14. Failure Modes
- **The Stale Cache Invalidation Desynchronization**: A customer updates their profile address. Application Server A writes the update to the database, but fails to execute `redis.del("user:123")` because of a transient network timeout. The cache retains the old address for 24 hours until the TTL expires, displaying incorrect shipping information to the customer. *Remediation: Invalidate cache via database Change Data Capture (Streams).*
- **The Accidental `noeviction` Out-of-Memory Lockout**: Running Redis with the default `noeviction` policy on a memory-saturated cluster. When memory reaches 100%, Redis rejects all `SET`, `HSET`, and `LPUSH` commands with `OOM command not allowed`, taking down the entire web platform.

## 15. Troubleshooting
When an application experiences elevated database load despite having a cache:
1. **Calculate Cache Hit Ratio**:
   - If Hit Ratio is $< 70\%$, review application caching keys. Are keys expiring too quickly, or is the application caching keys that are never read again?
2. **Inspect Redis Slowlog**:
   ```bash
   redis-cli -h <cache-host> slowlog get 10
   ```
   Look for expensive commands like `KEYS *` (which blocks the single-threaded engine) or massive `HGETALL` queries on collections with 100,000 items.
3. **Verify Eviction Rate**: Check if `Evictions` metric is climbing continuously.

## 16. Common Mistakes
- **Running `KEYS *` in Production**: Executing `KEYS *` to search for matching patterns. Because Redis is single-threaded, `KEYS *` scans millions of keys in RAM, freezing all operations for 10 to 30 seconds. Always use **`SCAN`** in production.
- **Failing to Set TTLs on Dynamically Generated Keys**: Writing temporary verification tokens or user session records to Redis without a Time-to-Live (`EXPIRE`). Over months, millions of abandoned keys consume 100% of available RAM.

## 17. Trade-offs
| Caching Pattern | Read Latency | Write Latency | Data Consistency Risk | Complexity |
| :--- | :--- | :--- | :--- | :--- |
| **Cache-Aside** | Low on hit; High on miss | Low (Writes to DB directly) | Moderate (Stale window until TTL/Del) | Low |
| **Read/Write-Through**| Low | Moderate (Sync double write) | **Zero (Cache & DB synchronized)** | Moderate |
| **Write-Behind** | **Ultra-Low** | **Sub-millisecond** | **High (Data loss if cache dies)** | High |

## 18. Interview Questions
1. *Explain the architectural trade-offs between the Cache-Aside, Write-Through, and Write-Behind caching patterns. When would you strictly mandate Write-Through over Cache-Aside?*
2. *Why does running the command `KEYS *` on a production Redis instance trigger a catastrophic latency spike across all connected microservices? What command must be used instead?*
3. *What is the difference between `volatile-lru` and `allkeys-lru` in Redis cache eviction, and what disastrous failure mode occurs if eviction is left on `noeviction`?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "The choice between Cache-Aside, Write-Through, and Write-Behind represents a fundamental trade-off between **Read Latency, Write Latency, and Data Consistency**:
>
> 1. **Cache-Aside (Lazy Loading)**:
>    - The application reads from the cache; on a miss, it reads from the database and writes the data into the cache. Writes bypass the cache or explicitly delete the cached key.
>    - *Trade-off*: Highly resilient (if the cache crashes, the app falls back to the database), and only frequently accessed data is cached. However, it introduces a three-hop latency penalty on cache misses and risks serving stale data if database writes fail to invalidate the cache.
>
> 2. **Write-Through Caching**:
>    - The application writes directly to the caching layer, which synchronously writes the updated record to the database before returning success.
>    - *Trade-off*: Read latency is optimal because data in the cache is always fresh, and data consistency between cache and database is 100% guaranteed. However, write latency is elevated because every write must complete both an in-memory update and a persistent disk I/O transaction.
>    - *When to Mandate*: I strictly mandate **Write-Through** for **financial balances, inventory counts, and authorization entitlements** where serving stale data from a lazy-loaded cache could cause financial loss or security bypasses.
>
> 3. **Write-Behind (Write-Back) Caching**:
>    - The application writes strictly to the in-memory cache and returns immediately. The cache asynchronously queues and flushes updates to the database in background batches.
>    - *Trade-off*: Delivers the highest possible write throughput and sub-millisecond write latency. However, if the cache node crashes before flushing dirty memory to the database, **committed customer transactions are permanently lost**. I only allow Write-Behind for non-critical, high-volume telemetry, video viewing counts, and real-time gaming leaderboards."

## 20. Hands-on Exercise
**Objective**: Connect to an AWS ElastiCache or OCI Redis instance and benchmark eviction behavior using `redis-benchmark`.

### Verification Steps
1. Connect to an EC2 or OCI VM in the cache subnet.
2. Run `redis-benchmark` to simulate 100,000 requests across 50 concurrent clients:
   ```bash
   redis-benchmark -h <cache-endpoint> -p 6379 -c 50 -n 100000 -t set,get -q
   ```
3. Observe benchmark output:
   ```text
   SET: 98,231.83 requests per second, p50 = 0.42 ms, p99 = 0.89 ms
   GET: 104,166.67 requests per second, p50 = 0.38 ms, p99 = 0.76 ms
   ```
4. Confirm that in-memory cache operations deliver consistent sub-millisecond p99 latency under 100,000 operations per second.
