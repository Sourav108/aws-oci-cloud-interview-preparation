# 03. High Availability, Latency & Data Residency Architecture

## 1. Problem
Senior and staff engineers are frequently asked to architect globally resilient systems that simultaneously achieve **zero data loss (RPO = 0)**, **instantaneous failover (RTO < 30s)**, **sub-50ms user latency**, and **strict compliance with national data residency regulations (GDPR, HIPAA, DORA)**. In reality, these requirements directly conflict. You cannot achieve zero data loss across continents without paying a heavy latency tax on every synchronous write, and you cannot replicate data freely across global regions without violating data residency laws.

## 2. Cloud Concept
Every distributed cloud architecture must reconcile physical geography with distributed systems theory:

### The Physical Topology Hierarchy
$$\text{Global Cloud (Realm)} \longrightarrow \text{Geographic Region} \longrightarrow \text{Availability Zone / Domain} \longrightarrow \text{Fault Domain} \longrightarrow \text{Physical Rack}$$

1. **Latency Budgets & The Speed of Light**:
   - Signal propagation in optical glass is roughly $200\text{ km/ms}$ ($\approx 5\mu\text{s/km}$) `[Inference]`.
   - Intra-Fault Domain (same rack): $< 0.1\text{ms}$ RTT.
   - Intra-Region Cross-AZ/AD (10–50 km): $1.0\text{ms} - 2.0\text{ms}$ RTT.
   - Cross-Region Continental (e.g., Virginia to Ohio $\approx 600\text{ km}$): $12\text{ms} - 20\text{ms}$ RTT.
   - Cross-Region Transoceanic (e.g., US East to Europe $\approx 6,500\text{ km}$): $70\text{ms} - 100\text{ms}$ RTT.
   - Cross-Region Transpacific (e.g., US West to Tokyo $\approx 8,500\text{ km}$): $110\text{ms} - 150\text{ms}$ RTT.

2. **Synchronous vs. Asynchronous Replication Boundary**:
   - **Synchronous Replication**: The primary node does not acknowledge a write to the client until a quorum of replicas acknowledge persistence to disk.
     - Feasible **only within a region** (across AZs/ADs), where 1–2ms RTT does not cripple database transaction throughput.
     - Guarantees **RPO = 0** (Zero data loss).
   - **Asynchronous Replication**: The primary acknowledges the write immediately, streaming the write-ahead log (WAL) to secondary regions asynchronously.
     - Mandatory **across regions**, preserving low write latency at the cost of non-zero RPO (typically seconds of potential data loss during a disaster).

3. **Data Residency & Sovereign Clouds**:
   - Legal frameworks (e.g., European Union GDPR, Germany BDSG, US FedRAMP) prohibit personal identifiable information (PII) or sovereign financial data from physically crossing jurisdictional borders.
   - Multi-region architectures must isolate customer data within legal boundaries while federating application binaries and non-PII metrics globally.

## 3. Mental Model
Think of replication latency as physical mail delivery:
- **Intra-AZ/Fault Domain**: Speaking to a colleague sitting at the next desk (instantaneous, zero overhead).
- **Cross-AZ**: Sending a courier on a bicycle across town (takes 2 minutes; completely fine for confirming critical legal agreements before signing).
- **Cross-Region**: Sending a package on an international cargo flight (takes days; you cannot wait for the cargo plane to land before telling the customer their credit card transaction was approved).

## 4. Architecture Diagram
```text
┌────────────────────────────────────────────────────────────────────────┐
│                        PRIMARY CLOUD REGION                            │
│                 (Active Traffic Tier: 100% Writes)                     │
│                                                                        │
│   [AZ-1 / AD-1]                 [AZ-2 / AD-2]                          │
│   ┌────────────────────┐        ┌────────────────────┐                 │
│   │ App Server Primary │        │ App Server Standby │                 │
│   └─────────┬──────────┘        └─────────┬──────────┘                 │
│             │                             │                            │
│             ▼ Synchronous Quorum (< 2ms)  ▼                            │
│   ┌──────────────────────────────────────────────────┐                 │
│   │ Primary Storage Engine (RPO = 0 across AZs)      │                 │
│   └─────────────────────────┬────────────────────────┘                 │
└─────────────────────────────┼──────────────────────────────────────────┘
                              │
                              │ Asynchronous WAL Stream (~80ms RTT)
                              │ RPO: ~1-5 seconds | RTO: < 2 minutes
                              ▼
┌────────────────────────────────────────────────────────────────────────┐
│                       SECONDARY CLOUD REGION                           │
│                 (Disaster Recovery Warm Standby)                       │
│                                                                        │
│   ┌──────────────────────────────────────────────────┐                 │
│   │ Standby Read Replica / Promotable Database Engine│                 │
│   └─────────────────────────┬────────────────────────┘                 │
│                             │                                          │
│                             ▼                                          │
│   ┌──────────────────────────────────────────────────┐                 │
│   │ Minimal Compute Fleet (Scales out upon failover) │                 │
│   └──────────────────────────────────────────────────┘                 │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS, high availability and disaster recovery are engineered through distinct regional primitives:

### Cross-AZ vs. Cross-Region Database Engines
- **Amazon Aurora Multi-AZ (Intra-Region)**: Writes are distributed across 6 storage nodes in 3 AZs. Quorum requires 4 of 6 nodes to acknowledge write persistence before returning success to the client `[Doc: Aurora Storage Architecture, checked 2026-09-03]`. This guarantees RPO=0 against any single AZ failure with sub-2ms latency overhead.
- **Amazon Aurora Global Database (Cross-Region)**: Uses dedicated storage-layer replication engines over the AWS global network backbone. Storage replication lag is typically under **1 second** `[Doc: Aurora Global Databases, checked 2026-09-03]`. In a regional disaster, secondary regions can be promoted to standalone read-write primaries in under **1 minute** (RTO < 1 min).
- **AWS European Sovereign Cloud & AWS Outposts**: Dedicated independent cloud partitions designed to satisfy European data residency requirements without external control plane dependencies.

## 6. OCI Implementation
OCI provides enterprise-grade data management designed specifically for high-throughput transactional continuity:

### Cross-AD vs. Cross-Region Database Engines
- **OCI Base Database & Autonomous Database with Active Data Guard**:
  - *Intra-Region (Synchronous)*: Configured with **Maximum Protection** or **Maximum Availability** mode. Writes are committed synchronously across Availability Domains or Fault Domains (RPO = 0) `[Doc: OCI Data Guard Overview, checked 2026-09-03]`.
  - *Cross-Region (Asynchronous)*: Configured with **Maximum Performance** mode. Data Guard streams redo logs asynchronously over OCI's high-bandwidth inter-region backbone with minimal impact on primary transaction latency.
- **Fast-Start Failover (FSFO)**: An automated observer process monitors primary database health. If the primary goes offline, the observer coordinates an automated, lossless failover to the standby database without requiring human intervention.
- **OCI Sovereign Cloud & Dedicated Regions**:
  - **OCI EU Sovereign Cloud**: Physically and logically separate sovereign cloud realm (`eu-sovereign-1`) located within the European Union, staffed exclusively by EU residents to guarantee strict GDPR and DORA compliance `[Doc: OCI EU Sovereign Cloud, checked 2026-09-03]`.
  - **Dedicated Region Cloud@Customer**: Delivers the entire OCI portfolio inside an enterprise customer's private data center, satisfying extreme data sovereignty mandates.

## 7. Configuration
Comparing cross-region disaster recovery replication setup:

### AWS Aurora Global Database (Terraform)
```hcl
# Primary Global Cluster in us-east-1
resource "aws_rds_global_cluster" "global_db" {
  global_cluster_identifier = "global-order-cluster"
  engine                    = "aurora-postgresql"
  engine_version            = "16.1"
  database_name             = "orders"
  storage_encrypted         = true
}

# Primary Regional Cluster
resource "aws_rds_cluster" "primary" {
  provider                  = aws.primary
  cluster_identifier        = "order-cluster-primary"
  global_cluster_identifier = aws_rds_global_cluster.global_db.id
  engine                    = aws_rds_global_cluster.global_db.engine
  engine_version            = aws_rds_global_cluster.global_db.engine_version
  availability_zones        = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# Secondary Regional Cluster in us-west-2 (DR)
resource "aws_rds_cluster" "secondary" {
  provider                  = aws.secondary
  cluster_identifier        = "order-cluster-dr"
  global_cluster_identifier = aws_rds_global_cluster.global_db.id
  engine                    = aws_rds_global_cluster.global_db.engine
  engine_version            = aws_rds_global_cluster.global_db.engine_version
  depends_on                = [aws_rds_cluster.primary]
}
```

### OCI Autonomous Database Cross-Region Data Guard (Terraform)
```hcl
# Primary Autonomous Database in Ashburn
resource "oci_database_autonomous_database" "primary_adb" {
  compartment_id           = var.compartment_id
  db_name                  = "ordersdb"
  display_name             = "orders-db-primary"
  cpu_core_count           = 2
  data_storage_size_in_tbs = 1
  is_auto_scaling_enabled  = true
}

# Standby Autonomous Database in Phoenix (DR)
resource "oci_database_autonomous_database" "standby_adb" {
  compartment_id      = var.compartment_id
  source              = "CROSS_REGION_DATAGUARD"
  source_id           = oci_database_autonomous_database.primary_adb.id
  remote_data_guard_type = "CROSS_REGION"
  db_name             = "ordersdb_dr"
  display_name        = "orders-db-standby"
}
```

## 8. Data Flow
```text
Write Request ──> [Active Regional Primary: us-east-1 / Ashburn]
                         │
         ┌───────────────┴───────────────┐
         │ (Synchronous Quorum < 2ms)    │ (Async WAL Shipping ~80ms)
         ▼                               ▼
 [Local Storage Nodes across AZs/FDs]   [Secondary Region: us-west-2 / Phoenix]
 (RPO = 0: Write Committed to Disk)     (Applied to Standby Read Replica)
         │                                       │
         ▼                                       ▼
  HTTP 200 OK to User                   Lag: Typically < 1 second
```

## 9. Security
- **Cross-Region Replication Encryption**: All replication traffic between AWS regions and OCI regions flows over dedicated private optical fiber backbone networks, encrypted in flight via TLS 1.3 or MACsec at the physical optical layer.
- **KMS Key Isolation in DR**: Cryptographic keys cannot be exported across regions. AWS KMS Customer Managed Keys (CMKs) and OCI Vault Master Encryption Keys are **Region-Bound**. Cross-region replication requires multi-region keys or re-encrypting data with the target region's KMS key upon arrival.

## 10. Reliability
- **The RPO / RTO Spectrum**:
  | DR Strategy | Cost Multiplier | RPO (Recovery Point) | RTO (Recovery Time) |
  | :--- | :---: | :---: | :---: |
  | **Backup & Restore (S3/Object Storage)** | $1\times$ | Hours (Last snapshot) | Hours to Days |
  | **Pilot Light (DB Replicating, Min Compute)** | $1.5\times$ | Seconds | 10 to 30 Minutes |
  | **Warm Standby (Scaled-Down Fleet)** | $2\times$ | Seconds | Under 5 Minutes |
  | **Active-Active Multi-Region** | $3\times - 5\times$ | Zero (or sub-second) | Near-Zero |
- **Split-Brain Prevention**: In multi-region active-active architectures, conflicting writes to the same record in two continents must be resolved via deterministic algorithms:
  - *Last-Writer-Wins (LWW)*: Relies on synchronized NTP/GPS wall clocks; prone to clock drift data loss.
  - *Conflict-Free Replicated Data Types (CRDTs)*: Mathematically proven convergent data structures for sets, counters, and registers.

## 11. Scaling
- **Read Scaling via Regional Replicas**: Cross-region read replicas (Aurora Read Replicas or OCI Active Data Guard Standby) allow local users in Europe or Asia to query read-only data with $< 10\text{ms}$ latency, routing only mutating writes back to the US primary region.

## 12. Observability
- **Replication Lag Telemetry**:
  - AWS Aurora: Monitor `AuroraGlobalDBReplicationLag` (milliseconds) in CloudWatch. Alarm if lag exceeds 5,000ms.
  - OCI Data Guard: Monitor `DataGuardLag` and `RedoApplyRate` in OCI Monitoring.
- **Synthetic Canary Probing**: Deploy synthetic canaries (AWS Synthetics / OCI Health Checks) from external locations worldwide to detect regional connectivity blackouts before customers report them.

## 13. Cost
- **Cross-Region Replication Egress**: Continuous streaming of database write-ahead logs (WAL) across regions generates steady egress bandwidth charges. Sizing database write volume is critical for predicting monthly DR networking costs.
- **Idle DR Compute Waste**: In Warm Standby architectures, run minimal compute nodes (e.g., 2 small instances) in the secondary region. Configure autoscaling to expand to 100% capacity only upon failover alarm trigger.

## 14. Failure Modes
- **The Synchronous Cross-Region Write Trap**: An engineering team configures synchronous 2-phase commits between US East and Frankfurt to guarantee zero data loss. The $85\text{ms}$ network latency turns a 5ms database write into a 180ms transaction. Database connection pools exhaust immediately, and the entire platform locks up under normal traffic.
- **The "Unpromotable" DR Replica**: A disaster strikes the primary region. The team attempts to promote the secondary region database, only to discover that the secondary database cannot decrypt its data because the IAM policy granting access to the secondary KMS key was never applied.

## 15. Troubleshooting
When investigating excessive cross-region replication lag:
1. **Inspect Network Path**: Check AWS Direct Connect / OCI FastConnect or inter-region fiber health status.
2. **Check Primary Write Saturation**: Did a massive batch ETL job generate gigabytes of WAL logs faster than the network pipe can stream?
3. **Inspect Standby Resource Utilization**: Is the standby database CPU maxed out at 100%, preventing the replication process from applying redo logs?

## 16. Common Mistakes
- **Assuming DNS Failover is Instantaneous**: Route 53 or OCI DNS health check failover depends on client-side DNS caching. Many ISPs and corporate resolvers ignore DNS TTLs (e.g., caching a 10s record for 15 minutes). RTO must account for DNS propagation tail latency.
- **Failing to Test Failback**: Teams spend months planning regional failover, but have zero documented procedure for **failing back** to the original primary region once it recovers, leading to accidental data overwrite.

## 17. Trade-offs
| Dimension | Active-Passive Warm Standby | Active-Active Multi-Region |
| :--- | :--- | :--- |
| **Data Consistency** | Strong (Single master, async replica) | Eventual (Multi-master, conflict resolution required) |
| **Failover RTO** | 1 to 5 minutes | Zero (Traffic shifts seamlessly) |
| **Cost** | Moderate ($1.5\times - 2\times$ base) | Extreme ($3\times - 5\times$ base + high egress) |
| **Operational Complexity** | Medium (Promote replica and swing DNS) | Extreme (Distributed consensus, split-brain risk) |

## 18. Interview Questions
1. *You are designing a core banking ledger that requires RPO = 0 and 99.999% availability. Can you achieve this with a multi-region active-active deployment? Defend your architectural choices.*
2. *How do you solve the challenge of cross-region encryption key management during an automated disaster recovery failover in AWS vs. OCI?*
3. *A financial regulator mandates that all German customer data must remain within the European Union. How do you design a high-availability disaster recovery architecture that complies with this regulation?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "Achieving both **RPO = 0** and **99.999% availability** simultaneously across multiple regions is fundamentally constrained by physics and the CAP theorem.
>
> 1. **The CAP / Physics Constraint**: To guarantee RPO = 0 across regions separated by hundreds of kilometers, every single financial write must be committed synchronously across both regions. A synchronous round-trip between US East and US West takes ~70ms; between US and Europe it takes ~85ms. This adds unacceptable latency to every debit/credit transaction and makes the primary region vulnerable to availability dips in the secondary region: if the cross-region link stutters, the primary must freeze writes to preserve consistency.
> 2. **The Correct Banking Architecture**:
>    - **Intra-Region Multi-AZ Active-Active (RPO = 0)**: In the primary region (e.g., Frankfurt or Ashburn), we deploy a multi-AZ cluster (Amazon Aurora or OCI Autonomous Database with Multi-AD Data Guard in Maximum Availability mode). Within the region, dark fiber latency is $< 2\text{ms}$. Writes commit synchronously across 3 Availability Zones or Fault Domains. If any single data center fails, RPO = 0 is mathematically preserved, and failover takes $< 30$ seconds.
>    - **Cross-Region Asynchronous DR (RPO $\approx 1$s, RTO $< 2$ mins)**: We stream the write-ahead log asynchronously to a secondary region (e.g., Ireland or Phoenix). In the catastrophic event of an entire regional failure, we accept an RPO of a few hundred milliseconds, or we pause automated promotion for 60 seconds to allow the replication queue to drain completely before manual promotion.
>    - This preserves ultra-low write latency in normal operations, guarantees 99.999% local availability, and provides bulletproof disaster recovery without the split-brain nightmares of multi-master banking systems."

## 20. Hands-on Exercise
**Objective**: Measure cross-region network latency and calculate the performance impact on synchronous vs. asynchronous database operations.

### Verification Steps
1. Deploy an EC2 instance in `us-east-1` and another in `us-west-2` (or OCI instances in Ashburn and Phoenix).
2. Measure round-trip ping latency:
   ```bash
   ping -c 50 <secondary-region-ip>
   ```
   *Expected Result*: Round-trip latency will average between $65\text{ms}$ and $80\text{ms}$.
3. Calculate: If a database transaction executes 10 sequential round-trip queries over this link, total transaction duration will exceed $700\text{ms}$—proving why cross-region data operations must be decoupled asynchronously.
