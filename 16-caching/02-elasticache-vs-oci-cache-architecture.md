# 02. ElastiCache vs. OCI Cache with Redis Architecture

## 1. Problem
A single Redis server is bottlenecked by two physical constraints: (1) it executes client commands on a single thread, and (2) its memory capacity is capped by the physical RAM of a single virtual machine (e.g., 512 GB). When an application's keyspace exceeds 1 TB or requires more than 150,000 write operations per second, a standalone Redis node saturates CPU cores and runs out of memory. Furthermore, if a single cache node crashes in an active-passive setup, the sudden cache drop exposes the backend database to a devastating flood of traffic. Hyperscale architectures solve this using distributed clustering and automated multi-node failover.

## 2. Cloud Concept
### Redis vs. Memcached: The Architectural Divide
- **Memcached**:
  - *Multithreaded Architecture*: Scales vertically across dozens of CPU cores on a single instance.
  - *Simple Key-Value*: Pure binary strings; no data structures, no disk persistence, no Pub/Sub, no replication.
  - *Best Use Case*: Simple, high-throughput caching of read-heavy HTML chunks or static serialized objects.
- **Redis (Remote Dictionary Server)**:
  - *Single-Threaded Execution Core*: Prevents race conditions and complex mutex locking; all data structure operations (sets, hashes, sorted sets) execute atomically.
  - *Rich Data Structures*: Supports hyperloglogs, geospatial radius queries, streams, bitfields, and Pub/Sub.
  - *Replication & Failover*: Built-in master-replica asynchronous replication with automated failover.

### ElastiCache Cluster Modes: Disabled vs. Enabled
1. **Cluster Mode Disabled (Single Shard)**:
   - Contains **1 Primary Node** and up to **5 Read Replicas**.
   - All writes target the Primary. Replicas serve read traffic.
   - *Limitation*: Total keyspace capacity is capped at the RAM of a single node (e.g., 500 GB). All nodes hold the exact same duplicate copy of the data.
2. **Cluster Mode Enabled (Distributed Sharding - The 16,384 Hash Slot Engine)**:
   - Partitions the keyspace horizontally across **up to 500 Shards** (each shard having 1 Primary and up to 5 Replicas) `[Doc: Amazon ElastiCache Cluster Limits, checked 2026-09-04]`.
   - **The Hash Slot Algorithm**:
     - The keyspace is divided into exactly **16,384 logical Hash Slots** (numbered 0 to 16383).
     - To find which shard holds a key, the client or node calculates:
       $$\text{Slot} = \text{CRC16}(\text{Key}) \pmod{16384}$$
     - *Shard Allocation Example*:
       - Shard 1 holds Slots $0 \longrightarrow 5460$.
       - Shard 2 holds Slots $5461 \longrightarrow 10922$.
       - Shard 3 holds Slots $10923 \longrightarrow 16383$.
   - **Hash Tags (`{...}`)**:
     - When executing multi-key transactions (`MGET`, `MSET`, Lua scripts), all target keys must reside on the **exact same shard**.
     - By wrapping a common substring in curly braces (e.g., `{user100}:profile` and `{user100}:orders`), Redis hashes **only the content inside the braces**, mathematically guaranteeing that both keys hash to the exact same slot!

## 3. Mental Model
Think of Redis cluster sharding as postal mail sorting:
- **Cluster Mode Disabled** is a small post office with 1 master clerk and 5 reading assistants. The master clerk handles all incoming mail and hands photocopies to the assistants. If the room fills with mail, you must move to a bigger room.
- **Cluster Mode Enabled (16,384 Hash Slots)** is a massive regional distribution center with 16,384 postal boxes distributed evenly across 10 specialized sorting conveyor belts (shards). The zip code (CRC16 hash) dictates which conveyor belt processes the letter. If mail volume doubles, you simply add 10 more conveyor belts and reassign the postal boxes.

## 4. Architecture Diagram
```text
REDIS CLUSTER SHARDING (16,384 HASH SLOTS ACROSS 3 SHARDS):

[Smart Redis Client (Jedis / Lettuce / ioredis)]
       │
       ├──► Calculates: Slot = CRC16("{tenant_42}:orders") % 16384 = 7210
       │
       ▼ Directly routes TCP packet to Shard 2 Primary (Zero proxy hops!)
┌────────────────────────────────────────────────────────────────────────┐
│ CLOUD IN-MEMORY CACHE FABRIC (ElastiCache / OCI Cache with Redis)      │
│                                                                        │
│   SHARD 1 (Slots 0 - 5460)         SHARD 2 (Slots 5461 - 10922)       │
│   ┌────────────────────────────┐   ┌────────────────────────────┐      │
│   │ Primary (AZ-1 / FD-1)      │   │ Primary (AZ-2 / FD-2) ◄────┼──┐   │
│   │   │ Asynchronous Repl      │   │   │ Asynchronous Repl      │  │   │
│   │   ▼                        │   │   ▼                        │  │   │
│   │ Replica (AZ-2 / FD-2)      │   │ Replica (AZ-3 / FD-3)      │  │   │
│   └────────────────────────────┘   └────────────────────────────┘  │   │
│                                                                    │   │
│   SHARD 3 (Slots 10923 - 16383)                                    │   │
│   ┌────────────────────────────┐                                   │   │
│   │ Primary (AZ-3 / FD-3)      │                                   │   │
│   │   │ Asynchronous Repl      │                                   │   │
│   │   ▼                        │                                   │   │
│   │ Replica (AZ-1 / FD-1)      │                                   │   │
│   └────────────────────────────┘                                   │   │
└────────────────────────────────────────────────────────────────────┼───┘
                                                                     │
  * Total Cluster Capacity: 3x RAM & 3x Write Throughput!            │
  * Direct Client Routing eliminates intermediate proxy latency! ────┘
```

## 5. AWS Implementation
In AWS ElastiCache:
- **Configuration & Smart Endpoints**:
  - *Configuration Endpoint*: Used by cluster-aware clients to automatically discover cluster topology, shard IP addresses, and hash slot distributions.
  - *Primary Endpoint / Reader Endpoint*: Provided in Cluster Mode Disabled configurations for explicit write/read splitting.
- **ElastiCache Data Tiering (R6gd Shapes)**:
  - Integrates NVMe SSDs alongside physical RAM on each node.
  - When memory reaches capacity, least frequently used items are swapped automatically to local NVMe flash storage with sub-millisecond read latency.
  - Slashes storage costs by up to **60%** for massive multi-terabyte caches `[Doc: Amazon ElastiCache Data Tiering, checked 2026-09-04]`.
- **Auto-Scaling Clusters**:
  - Scales shards dynamically (e.g., from 5 to 20 shards) or scales read replicas per shard based on CloudWatch CPU and memory alarms.

## 6. OCI Implementation
In Oracle Cloud Infrastructure Cache with Redis:
- **OCI Architecture & Topology**:
  - OCI Cache with Redis provides a fully managed, enterprise Redis 7.0+ caching service `[Doc: OCI Cache with Redis Architecture, checked 2026-09-04]`.
  - Built directly on OCI's ultra-low-latency physical Clos network fabric, delivering sub-millisecond p99 latency between compute VMs and cache clusters.
- **Fault Domain Native Distribution**:
  - Automatically isolates primary and replica nodes across **OCI Fault Domains** within an Availability Domain.
  - Guarantees protection against physical chassis failures, power rack outages, and hypervisor maintenance events.
- **Zero Control Plane Premium**:
  - OCI bills strictly for the memory and compute capacity allocated to the cache nodes, without additional proprietary software licensing fees.
- **VCN Private Network Security**:
  - Integrates directly with OCI Network Security Groups (NSGs).
  - Employs dedicated private IP interfaces inside customer regional subnets. Public IP assignment is prohibited by design, preventing accidental internet exposure.

## 7. Configuration
Comparing sharded Redis cluster provisioning in Terraform across AWS and OCI:

### AWS ElastiCache Cluster Mode Enabled (Terraform)
```hcl
# AWS ElastiCache Redis Cluster (Cluster Mode Enabled)
resource "aws_elasticache_replication_group" "sharded_redis" {
  replication_group_id = "prod-sharded-redis"
  description          = "Sharded Redis Cluster across 3 AZs"
  node_type            = "cache.r6g.xlarge"
  port                 = 6379
  parameter_group_name = "default.redis7.cluster.on"
  subnet_group_name    = var.cache_subnet_group_name
  security_group_ids   = [var.cache_security_group_id]

  # Sharding Configuration: 3 Shards, 1 Replica per Shard (6 nodes total)
  num_node_groups         = 3
  replicas_per_node_group = 1
  automatic_failover_enabled = true
  multi_az_enabled           = true

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token
}
```

### OCI Cache with Redis Sharded Cluster (Terraform)
```hcl
# OCI Cache with Redis (Sharded Multi-Node Configuration)
resource "oci_redis_redis_cluster" "sharded_cache" {
  compartment_id     = var.compartment_id
  display_name       = "prod-sharded-cache"
  node_count         = 6 # 3 Primaries + 3 Replicas
  node_memory_in_gbs = 32
  software_version   = "V7_0_5"
  subnet_id          = var.private_cache_subnet_id
  nsg_ids            = [var.cache_nsg_id]

  cluster_mode = "SHARDED"
}
```

## 8. Data Flow
```text
The Smart Client MOVED Redirection Protocol:
1. Application initializes Redis client with Configuration Endpoint.
2. Client queries: CLUSTER SLOTS
   - Receives mapping: Shard 1 (0-5460), Shard 2 (5461-10922), Shard 3 (10923-16383).
   - Caches slot map in application memory.
3. App issues command: SET user:99 "data"
   - Client calculates CRC16("user:99") % 16384 = 8214.
   - Client sends TCP packet directly to Shard 2 Primary!
4. Zero-Latency Execution:
   - Shard 2 Primary executes write in RAM in 0.2ms.
   - Asynchronously streams write to Shard 2 Replica.
5. Cluster Rebalancing / Shard Migration Case:
   - If Slot 8214 migrated to Shard 3 and client map is stale:
   - Shard 2 returns: -MOVED 8214 10.0.3.50:6379
   - Client updates internal slot map and replays write to 10.0.3.50 transparently!
```

## 9. Security
- **Redis AUTH & RBAC**:
  - Enforce strong passwords via Redis `AUTH` token or configure fine-grained Redis Access Control Lists (ACLs) to restrict dangerous commands (`FLUSHALL`, `CONFIG`, `DEBUG`).
- **TLS In-Transit Encryption**:
  - Required in production to prevent man-in-the-middle packet sniffing of plaintext cached sessions traversing hypervisors.

## 10. Reliability
- **Multi-AZ Automatic Failover Mechanics**:
  - Primary nodes stream updates to replicas asynchronously.
  - If a primary node misses heartbeats for $> 15\text{ seconds}$, the cluster elects the healthiest replica and promotes it to primary.
  - DNS endpoints or smart clients update routing automatically in **under 30 seconds**.

## 11. Scaling
- **Online Resharding (Horizontal Scaling)**:
  - Both AWS ElastiCache and OCI Cache support **online resharding**: adding new shards to an active cluster while serving production traffic.
  - Hash slots are dynamically migrated from existing shards to the new shards in the background with zero downtime.

## 12. Observability
- **Primary Health Metrics**:
  - `EngineCPUUtilization`: Single-threaded Redis engine load.
  - `CurrConnections`: Active client connection count. If climbing into thousands, implement client-side connection pooling.
  - `ReplicationLag`: Time in seconds the replica is lagging behind the primary.

## 13. Cost
- **Sizing Economics**:
  - Running a 3-shard cluster on `cache.r6g.xlarge` (26.32 GB RAM per node, 6 nodes total) costs $\approx \$1,500/\text{month}$.
  - Optimize by evaluating **ElastiCache Serverless** for spiky traffic or **Data Tiering** for massive datasets where 80% of keys are cold.

## 14. Failure Modes
- **The Cross-Slot Transaction Deadlock (`CROSSSLOT Keys in request don't hash to the same slot`)**: An application executes a multi-key command: `MGET order:100 user:100`. Because `order:100` hashes to Shard 1 and `user:100` hashes to Shard 2, the Redis engine rejects the operation with a fatal `CROSSSLOT` error. *Remediation: Use Hash Tags: `MGET {tenant:100}:order {tenant:100}:user`.*
- **The Replication Lag Data Loss Window During Failover**: A primary node accepts 1,000 writes and commits them to memory. Before asynchronous replication can sync those writes to the replica, the physical host loses power. The replica is promoted to primary, but the 1,000 writes are permanently lost.

## 15. Troubleshooting
When Redis returns `CLUSTERDOWN The cluster is down`:
1. **Verify Quorum Across Shards**:
   - `CLUSTERDOWN` indicates that a majority of master nodes cannot reach consensus, or hash slots are unbound.
2. **Inspect Node States**:
   ```bash
   redis-cli -h <endpoint> cluster nodes
   ```
   Check for nodes marked `fail` or slots marked `unassigned`.
3. **Inspect Client Driver Configuration**:
   - Ensure the application is using a **cluster-aware driver** (e.g., `JedisCluster` in Java or `Redis.Cluster` in Node.js). Pointing a standard standalone Redis client at a sharded cluster endpoint will fail with unhandled `MOVED` redirect exceptions.

## 16. Common Mistakes
- **Using a Standalone Client on a Sharded Cluster**: Using non-cluster Redis client libraries. The client hits Node A, receives `-MOVED 8214 10.0.1.20`, treats it as an unhandled error, and crashes the web application.
- **Neglecting Hash Tags for Multi-Key Lua Scripts**: Writing Lua scripts that touch multiple keys without enforcing Hash Tags, causing script execution to fail when deployed to production sharded clusters.

## 17. Trade-offs
| Architecture | Max Keyspace | Write Scalability | Client Complexity | Cost |
| :--- | :--- | :--- | :--- | :--- |
| **Cluster Mode Disabled** | Single node RAM (~500 GB) | Single primary node limit | Low (Standard client) | Low |
| **Cluster Mode Enabled** | **Tens of Terabytes** | **Scales across 500 shards**| High (Cluster-aware driver & hash tags)| Moderate to High |
| **ElastiCache Data Tiering**| Massive NVMe expansion | High | Moderate | **Up to 60% cheaper per GB** |

## 18. Interview Questions
1. *Explain how Redis Cluster partitions keyspaces horizontally using the 16,384 Hash Slot algorithm. How do smart Redis clients interact with hash slots without introducing an extra proxy hop?*
2. *What is a `CROSSSLOT` error in Redis Cluster Mode Enabled, and how do you resolve it using Hash Tags (`{...}`)?*
3. *How does OCI Cache with Redis leverage OCI Fault Domains to deliver high availability, and how does its architecture compare to AWS ElastiCache Multi-AZ replication groups?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "Redis Cluster scales horizontally using **16,384 logical Hash Slots** and a direct-to-node routing protocol implemented inside **Smart Redis Client Libraries**:
>
> 1. **The Hash Slot Mathematical Model**:
>    - Rather than hashing keys directly to physical server IP addresses (which would require re-hashing the entire database whenever a node is added or removed), Redis partitions the keyspace into a fixed allocation of **16,384 Hash Slots** (numbered 0 to 16,383).
>    - Every key's slot is calculated deterministically using the **CRC16 checksum**:
>      $$\text{Slot} = \text{CRC16}(\text{Key}) \pmod{16384}$$
>    - These 16,384 slots are divided among the primary shards in the cluster. For example, in a 3-shard cluster:
>      - Shard 1 holds Slots $0 \longrightarrow 5460$.
>      - Shard 2 holds Slots $5461 \longrightarrow 10922$.
>      - Shard 3 holds Slots $10923 \longrightarrow 16383$.
>
> 2. **Smart Client Routing (Zero Proxy Overhead)**:
>    - In traditional architectures, traffic passes through a centralized load balancer or proxy, which adds an extra network hop and increases p99 latency.
>    - Redis Cluster eliminates this using **Smart Clients** (e.g., Lettuce in Java, `ioredis` in Node.js, `redis-py` in Python):
>      1. On startup, the client connects to any node and issues the `CLUSTER SLOTS` command.
>      2. It downloads the complete map of which IP address owns which range of hash slots and caches this map in client memory.
>      3. When the application issues `GET user:492`, the client runs `CRC16("user:492") % 16384`, finds the target slot, looks up the corresponding shard's IP, and **opens a direct TCP socket straight to that specific primary node**.
>
> 3. **Dynamic Topology Rebalancing via `MOVED` Redirects**:
>    - If the cloud platform adds a new shard or fails over to a replica, the client's cached map becomes temporarily stale.
>    - If the client queries the wrong node, the node responds with a redirection message:
>      `-MOVED <slot> <new-node-ip:port>`.
>    - The smart client catches this response, transparently updates its internal slot map, and re-executes the command against the new node in $< 1\text{ms}$, delivering linear horizontal scalability with zero proxy bottlenecks."

## 20. Hands-on Exercise
**Objective**: Demonstrate Redis Hash Slot calculation and Hash Tag enforcement using `redis-cli`.

### Verification Steps
1. Connect to an active Redis cluster.
2. Inspect cluster slot distribution:
   ```bash
   redis-cli -c -h <endpoint> -p 6379 cluster slots
   ```
3. Attempt a multi-key set without hash tags:
   ```bash
   redis-cli -c -h <endpoint> -p 6379 MSET keyA 1 keyB 2
   ```
   Observe: If `keyA` and `keyB` hash to different slots, the command fails with `(error) CROSSSLOT Keys in request don't hash to the same slot`.
4. Re-run using **Hash Tags**:
   ```bash
   redis-cli -c -h <endpoint> -p 6379 MSET {user100}:keyA 1 {user100}:keyB 2
   ```
   Observe: Command succeeds instantly (`OK`) because the `{user100}` tag forces both keys into the exact same hash slot!
