# 03. Cache Invalidation & Stampede Mitigation

## 1. Problem
In high-throughput distributed systems, the moment an in-memory cache experiences a perturbation, the backend database is exposed to catastrophic failure. If a hot cached key accessed by 20,000 concurrent requests per second expires, all 20,000 requests experience a cache miss simultaneously. Every request races to query the database and recompute the cache value, slamming the database with a sudden 20,000-query spike (**The Cache Stampede / Thundering Herd**). The database CPU spikes to 100%, connection pools exhaust, and the entire platform collapses. Understanding cache failure dynamics and mitigation algorithms is essential for senior cloud infrastructure design.

## 2. Cloud Concept: The Three Cache Disasters
```text
1. CACHE STAMPEDE (THUNDERING HERD)
Hot Key 'homepage_deals' expires at 12:00:00.
10,000 concurrent users hit Cache Miss at 12:00:01 ──► 10,000 heavy SQL queries crush DB!

2. CACHE AVALANCHE
1,000,000 keys cached with exact same TTL = 3600 seconds.
At second 3600, ALL 1,000,000 keys expire simultaneously ──► Database collapses under mass miss!

3. CACHE PENETRATION
Malicious actor requests non-existent IDs: GET /users/-999999
ID does not exist in Cache ──► Hits DB ──► DB returns NULL ──► Cache never stores NULL.
Every single request punches straight through to the database!
```

- **Cache Stampede (Thundering Herd)**: Occurs when a single, heavily requested hot key expires, causing thousands of concurrent application threads to query the database simultaneously to regenerate the key.
- **Cache Avalanche**: Occurs when a massive batch of cached items share the exact same TTL (Time-to-Live) and expire at the exact same second, causing system-wide cache hit ratios to drop from 95% to 0%.
- **Cache Penetration**: Occurs when queries target data that does not exist in either the cache or the database. Because the database returns empty results, the application never populates the cache, allowing repetitive queries to bypass the cache entirely and hammer the database.

## 3. Cache Stampede Mitigation: Distributed Mutex Locking (`SETNX`)
To prevent thousands of threads from regenerating the same expired key in parallel, enforce a **Distributed Mutex Lock**:
1. When an application thread discovers a Cache Miss, it attempts to acquire a temporary distributed lock in Redis using `SETNX` (Set if Not Exists) with a 5-second TTL:
   ```bash
   SET lock:product:42 "uuid_token" NX EX 5
   ```
2. **If Lock Acquired**: That single thread queries the database, writes the refreshed result to Redis with a standard TTL, and releases the lock.
3. **If Lock Fails (Other threads)**: The remaining 9,999 threads sleep for 50 milliseconds and retry reading from the cache, where the freshly populated data is now waiting!

## 4. Probabilistic Early Expiration: The XFetch Algorithm
Instead of waiting for a key to expire and suffering a lock race, the **XFetch algorithm** probabilistically triggers a background refresh before the key actually expires based on read frequency and computation time:

$$\text{Compute Delta} = - \beta \times \delta \times \ln(\text{random}(0, 1))$$

- $\delta$: Time taken to compute the database query (seconds).
- $\beta$: Eagerness parameter ($> 0$, typically 1.0).
- If $(\text{Time-to-Live Remaining}) \le \text{Compute Delta}$, the current reading thread asynchronously refreshes the key in the database while immediately returning the current valid cached value to the user! The key **never expires**, completely eliminating the cache stampede.

## 5. Cache Avalanche & Penetration Defenses
- **Defeating Cache Avalanche with Randomized TTL Jitter**:
  - Never use hardcoded static TTLs (e.g., `TTL = 3600`).
  - Add randomized **Jitter** to the expiration window:
    $$\text{TTL} = \text{Base TTL} + \text{random}(0, 300\text{ seconds})$$
  - Keys expire smoothly across a 5-minute distribution curve, preventing cliff-edge mass expirations.
- **Defeating Cache Penetration with Bloom Filters & Null Caching**:
  1. *Null Object Caching*: When the database confirms an ID does not exist, cache a sentinel value (`"null"`) with a short 60-second TTL so subsequent queries hit the cache.
  2. *Bloom Filter*: Place a memory-efficient **Bloom Filter** (supported in Redis via RedisBloom) in front of the cache. If the Bloom Filter reports the ID does not exist, reject the request immediately without touching the cache or database!

## 6. Troubleshooting & Redis Inspection Commands
1. **Detect High Key Eviction or Sudden Expiration**:
   ```bash
   redis-cli -h <cache-host> info stats | grep -E "evicted_keys|expired_keys"
   ```
2. **Inspect Memory Fragmentation**:
   ```bash
   redis-cli -h <cache-host> info memory | grep "mem_fragmentation_ratio"
   ```
   A ratio $> 1.5$ indicates heavy memory fragmentation; run `MEMORY PURGE` or enable active defragmentation.

## 7. Senior Interview Question & Defense
**Question**: *A high-traffic news portal features a breaking news article accessed by 50,000 users per second. The article cache key has a 60-second TTL. At second 60, the database CPU surges from 15% to 100%, causing a total outage. Explain what occurred, compare Mutex Locking vs. Probabilistic Early Expiration, and defend your choice.*

**Staff-Level Defense**:
> "This outage is a classic **Cache Stampede (Thundering Herd)** catastrophe:
>
> 1. **The Forensic Failure Sequence**:
>    - For 60 seconds, all 50,000 RPS are absorbed by in-memory Redis in $< 1\text{ms}$.
>    - At second 60.001, the cache key expires.
>    - Within a 100-millisecond window, 5,000 incoming requests experience a Cache Miss.
>    - Because the application follows naive Cache-Aside logic, all 5,000 threads simultaneously dispatch heavy SQL queries to the relational database to fetch the article and comments.
>    - The database connection pool exhausts in 200ms, query queues explode, and database CPU locks at 100%, killing the platform.
>
> 2. **Evaluating Solutions: Mutex Locking vs. Probabilistic Early Expiration (XFetch)**:
>    - *Approach A: Distributed Mutex Locking (`SETNX`)*:
>      The first thread acquires a lock to query the DB; other threads wait. While this protects the database, the 4,999 waiting threads experience artificial latency spikes ($50–200\text{ms}$ delay while waiting for the lock).
>    - *Approach B: Probabilistic Early Expiration (The XFetch Algorithm)*:
>      As the key approaches expiration, incoming reads evaluate a probabilistic formula factoring in query computation duration ($\delta$) and a random float. As TTL drops, the probability that a reader triggers an asynchronous background database refresh approaches 100%.
>
> 3. **The Staff-Level Architecture**:
>    - I implement **Probabilistic Early Expiration (XFetch)** for this breaking news workload.
>    - The key is refreshed in the background *while it is still warm in cache*.
>    - **Result**: Zero cache misses, zero user-facing latency spikes, and 100% of all 50,000 RPS continue receiving sub-millisecond cached responses uninterrupted."
