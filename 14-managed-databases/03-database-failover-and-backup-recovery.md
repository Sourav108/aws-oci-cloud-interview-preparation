# 03. Database Failover Mechanics & Point-in-Time Recovery

## 1. Problem
When a primary database server experiences hardware termination, power loss, or operating system kernel panics, enterprise applications must maintain high availability without data loss. If failover mechanics are poorly engineered, application connection pools hang in socket read timeouts, DNS TTL caching prevents clients from discovering the newly promoted primary for 15 minutes, or split-brain partitions allow conflicting writes on both primary and standby nodes. Furthermore, accidental administrative mistakes (`DROP TABLE users`) require resilient Point-in-Time Recovery (PITR) mechanisms to restore databases to the exact second prior to corruption.

## 2. Cloud Concept: Failover Protocols & Continuous Log Archiving
```text
DATABASE HIGH AVAILABILITY & CONTINUOUS RECOVERY ARCHITECTURE:

[Active Database Primary Node] ──► Synchronous / Asynchronous Log Stream ──► [Standby Database Node]
            │                                                                        ▲
            ▼ Continuous WAL / Redo Flush                                            │ Promoted during
┌────────────────────────────────────────────────────────┐                           │ failure!
│ OBJECT STORAGE ARCHIVE (Amazon S3 / OCI Object Storage)│                           │
│ * Baseline Daily Storage Snapshots                     │                           │
│ * Continuous Transaction Log Archiving (WAL / Redo)    │                           │
│ * Point-in-Time Recovery (PITR) restored to any second │                           │
└────────────────────────────────────────────────────────┘                           │
                                                                                     │
[Independent Failover Observer / Cloud Monitor] ─────────────────────────────────────┘
(Detects primary outage via heartbeat; promotes standby in < 30s)
```

- **Failover Routing Mechanisms**:
  1. *DNS CNAME Repointing (AWS RDS)*: The database service updates the DNS record of the database endpoint (e.g., `db.xyz.rds.amazonaws.com`) to point to the standby instance's IP address.
  2. *Virtual IP (VIP) Swapping (OCI RAC / Multi-AD)*: A floating IP address is programmatically reassigned across network interfaces in under 2 seconds.
  3. *Client-Side Multi-Host Reconnection*: Cloud database drivers (JDBC/ODBC) are configured with connection strings containing both primary and standby endpoints, enabling the driver to detect failovers instantly without waiting for DNS propagation.
- **Point-in-Time Recovery (PITR)**:
  - Combines periodic **full storage volume snapshots** with **continuous transaction log archiving** (PostgreSQL WAL, MySQL binlog, Oracle Redo logs) streamed directly to durable object storage.
  - Enables restoring a fresh database instance to any arbitrary second within the backup retention window (e.g., 1 to 35 days).

## 3. AWS Failover & Recovery Implementation
In AWS:
- **RDS Multi-AZ Failover**:
  - Automatically initiates failover if the primary instance loses network connectivity, crashes, or during routine maintenance.
  - AWS updates the DNS CNAME record of the endpoint. Total failover duration: **60 to 120 seconds**.
- **Aurora Cluster Promotion**:
  - If a primary writer node fails, Aurora evaluates Read Replicas based on promotion tiers (`tier 0` to `tier 15`).
  - The highest-priority replica is promoted to primary writer in **$< 15\text{ seconds}$**.
- **Automated Continuous Backups**:
  - AWS RDS continuously backs up data and transaction logs to Amazon S3.
  - PITR allows restoring to any point up to the last 5 minutes.

## 4. OCI Failover & Recovery Implementation
In OCI:
- **OCI Active Data Guard & Fast-Start Failover (FSFO)**:
  - An independent **Observer** process continuously pings the primary and standby databases.
  - If the primary fails to respond within a threshold (e.g., 30 seconds), FSFO verifies network health and automatically executes a failover, promoting the standby with zero data loss ($RPO = 0$).
- **Autonomous Database Automated Recovery**:
  - Oracle Autonomous Database takes automated daily full backups and continuous archivelog backups to OCI Object Storage.
  - Backups are stored across 3 Fault Domains / Availability Domains with 11 9s of durability.
  - Supports point-in-time recovery via the OCI console or API with a single click, restoring databases in minutes.

## 5. Production Failure Modes: DNS Caching & Split-Brain
- **The JVM DNS Caching Lockout**: In Java applications, the default JVM security setting caches DNS resolutions forever (`networkaddress.cache.ttl = -1`). When an RDS primary fails over and AWS flips the DNS CNAME, the Java application continues attempting to connect to the dead primary's old IP address indefinitely!
  - *Mandatory Fix*: Set JVM DNS TTL explicitly: `-Dsun.net.inetaddr.ttl=5` (cache DNS for no more than 5 seconds).
- **The Asynchronous Replication Data Loss Window (RPO > 0)**: Promoting an asynchronous read replica during an emergency failover when the replica was lagging behind by 12 seconds. All transactions committed during those 12 seconds are permanently lost.

## 6. Troubleshooting & Diagnostics
1. **Verify Failover Event in Cloud Audit Logs**:
   ```bash
   aws rds describe-events --source-identifier prod-db --source-type db-instance
   ```
   Look for: `Multi-AZ instance failover started` and `Multi-AZ instance failover completed`.
2. **Inspect Current Database Write Role**:
   ```sql
   -- PostgreSQL Primary Check
   SELECT pg_is_in_recovery(); -- Returns FALSE on primary writer, TRUE on read replica!
   ```

## 7. Senior Interview Question & Defense
**Question**: *When Amazon RDS Multi-AZ fails over to the standby instance, application servers continue throwing 'Connection Refused' and 'Read Timeout' errors for over 10 minutes, even though the AWS console reports the database is healthy. What is the root cause across the application stack, and how do you architect a zero-downtime failover?*

**Staff-Level Defense**:
> "This prolonged outage is caused by a failure across two distinct layers: **Application-Level DNS Caching** and **Connection Pool Stagnation**:
>
> 1. **The Root Causes**:
>    - **DNS Caching**: RDS Multi-AZ failover relies on **DNS CNAME repointing**. When the primary fails, AWS updates the DNS record to point to the standby instance's IP. If the application runtime (e.g., Java JVM or Node.js) caches DNS responses based on OS defaults rather than respecting the 5-second DNS TTL, the application continues bombarding the dead IP address.
>    - **Dead Connection Pool Sockets**: Application connection pools (such as HikariCP) maintain persistent TCP sockets to the database. During failover, the primary OS terminates abruptly without sending TCP FIN packets. Client sockets remain stuck in half-open TCP states, waiting for operating system keep-alive timeouts (which default to 2 hours in Linux!).
>
> 2. **The 3-Step Production Remediation**:
>    - **Step 1: Deploy Amazon RDS Proxy (or OCI CMAN)**:
>      We route all application traffic through **Amazon RDS Proxy**. When failover occurs, the proxy automatically absorbs the failover event, transparently queues incoming queries in memory for 3 to 5 seconds, and connects to the newly promoted primary. Client applications experience zero connection drops.
>    - **Step 2: Enforce Aggressive TCP Keep-Alives and Socket Timeouts**:
>      In HikariCP and the JDBC driver, configure:
>      - `maxLifetime = 600000` (10 minutes)
>      - `connectionTimeout = 5000` (5 seconds)
>      - Set Linux kernel `tcp_keepalive_time = 30` seconds to detect severed sockets immediately.
>    - **Step 3: Lock JVM DNS TTL to 5 Seconds**:
>      Configure `-Dsun.net.inetaddr.ttl=5` in application startup flags so that DNS resolutions expire and refresh within 5 seconds of an AWS DNS change."
