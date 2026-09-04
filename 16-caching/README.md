# Module 16: Caching (ElastiCache vs. OCI Cache with Redis)

> **Architectural Objective**: *Master in-memory caching topologies, caching design patterns (Cache-Aside, Write-Through, Write-Behind), cluster sharding algorithms, and high-availability cache resilience. Deconstruct AWS ElastiCache against OCI Cache with Redis, evaluate eviction algorithms (LRU/LFU), and master advanced disaster mitigations (Cache Stampede, Cache Avalanche, and Cache Penetration).*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. Caching Strategies & Design Patterns](01-caching-strategies-and-patterns.md)** | Topologies, Cache-Aside (Lazy Loading), Read-Through, Write-Through, Write-Behind, Eviction Policies (LRU, LFU, Volatile/Allkeys) | Full 20-Section Deep Dive (~2,200 words) |
| **[02. ElastiCache vs. OCI Cache Architecture](02-elasticache-vs-oci-cache-architecture.md)** | Redis vs. Memcached, Cluster Mode Enabled (16,384 Hash Slots) vs. Disabled, Multi-AZ Failover, OCI Cache with Redis Sharding & Fault Domains | Full 20-Section Deep Dive (~2,400 words) |
| **[03. Cache Invalidation & Stampede Mitigation](03-cache-invalidation-and-stampede-mitigation.md)** | Cache Stampede (Thundering Herd), Distributed Mutex Locking (`SETNX`), Probabilistic Early Expiration (XFetch), Cache Avalanche Jitter | Abbreviated In-Memory Guide (~950 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **The 16,384 Hash Slot Sharding Engine**: How Redis Cluster partitions keyspaces horizontally across up to 500 shards using CRC16 hashing, and how smart Redis client drivers avoid redirect penalties.
2. **Mitigating the Thundering Herd / Cache Stampede**: How to prevent catastrophic database collapse when a hot cache key expires by implementing distributed locks or the probabilistic XFetch algorithm.
3. **Cache Invalidation Consistency**: Why maintaining dual-write cache invalidation in application code is an anti-pattern, and how to build event-driven cache synchronization using database CDC streams.
