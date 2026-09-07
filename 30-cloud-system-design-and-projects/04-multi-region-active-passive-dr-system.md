# System Design 04: Multi-Region Active-Passive Disaster Recovery System

---

## 1. Requirements & Constraints (R)

### 1.1 Business Context & Problem Statement
An enterprise core banking and regulatory clearing system operates in a primary cloud region (`us-east-1` on AWS / `us-ashburn-1` on OCI). Given the existential business impact of a total regional cloud outage (such as a regional power failure, fiber cut, or catastrophic control-plane degradation), executive leadership mandates an automated, highly disciplined multi-region Disaster Recovery (DR) architecture into a secondary pairing region (`us-west-2` on AWS / `us-phoenix-1` on OCI).

The architecture must support **Warm Standby** operation: the secondary region maintains real-time replicated data and a warm compute baseline capable of rapidly scaling out to assume 100% of global production traffic while enforcing strict RPO/RTO compliance and preventing split-brain corruption.

### 1.2 Functional Requirements
1. **Automated Cross-Region Data Synchronization**:
   - Continuous replication of relational databases, object storage, block volumes, and configuration secrets from Primary to Secondary.
2. **Deterministic Failover Orchestration**:
   - Ability to execute both **Planned Switchover** (maintenance, drills) with zero data loss, and **Unplanned Failover** (catastrophic disaster) with minimal data loss.
3. **Split-Brain Fencing**:
   - Mechanically prevent both regions from simultaneously accepting write traffic if network communication between regions is severed.
4. **Global Health Probing & Traffic Redirection**:
   - Continuously evaluate end-to-end health from external vantage points and shift global DNS ingress without operator guesswork.

### 1.3 Non-Functional Requirements & Quantitative SLAs
- **Recovery Point Objective (RPO)**:
  - Planned Switchover: $\mathbf{RPO = 0}$ (Zero committed data loss).
  - Unplanned Regional Disaster: $\mathbf{RPO < 1\text{ minute}}$ (Typical asynchronous replication lag is $< 1\text{ second}$).
- **Recovery Time Objective (RTO)**:
  - Total time from disaster declaration to full traffic serving in Secondary Region: $\mathbf{RTO < 15\text{ minutes}}$.
- **Traffic Profile**:
  - Baseline Global Traffic: **25,000 QPS**.
  - Secondary Region Warm Standby Compute Footprint: **20% baseline capacity** (allows immediate health validation and pre-warmed connection pools).
- **Compliance & Immutable Retention**:
  - Daily database snapshots replicated to an isolated third regulatory compliance account/tenancy with immutable WORM locks (Write Once Read Many) for 7 years.

---

## 2. High-Level Architecture (A)

### 2.1 Dual-Cloud Architectural Topology

```
========================================================================================================================
                          MULTI-REGION ACTIVE-PASSIVE DISASTER RECOVERY TOPOLOGY
========================================================================================================================

                                  [ Global User Ingress (Web / Mobile / B2B) ]
                                                       │
                                                       ▼
                            [ Global Traffic Steering & Disaster Orchestration ]
                            - AWS: Route 53 Application Recovery Controller (ARC)
                            - OCI: Traffic Management Steering Policies + Full Stack DR (FSDR)
                                                       │
                           ┌───────────────────────────┴───────────────────────────┐
                           │ (100% Active Production Traffic)                      │ (0% Standby - Fails Over to 100%)
                           ▼                                                       ▼
  ═══════════════════════════════════════════════════════  ═══════════════════════════════════════════════════════
  PRIMARY REGION (AWS us-east-1 / OCI Ashburn)             SECONDARY REGION (AWS us-west-2 / OCI Phoenix)
  ───────────────────────────────────────────────────────  ───────────────────────────────────────────────────────
   [ Public ALB / OCI Flexible Load Balancer ]              [ Public ALB / OCI Flexible Load Balancer ]
                           │                                                       │
                           ▼                                                       ▼
   [ Compute Tier: EKS / OKE Pod Fleet (100% Load) ]        [ Compute Tier: EKS / OKE Pod Fleet (20% Warm Load) ]
                           │                                                       │
                           ▼                                                       ▼
   [ Caching Tier: ElastiCache Redis / OCI Cache ]          [ Caching Tier: Standby Redis (Pre-warmed) ]
                           │                                                       │
                           ▼                                                       ▼
   [ Primary Writer Database ]                              [ Standby Replica Database ]
   - AWS: Aurora PostgreSQL (Primary Writer)               - AWS: Aurora Global DB Replica (Read-Only)
   - OCI: Autonomous DB / Base DB (Primary)                - OCI: Active Data Guard (ADG) Standby
                           │                                                       ▲
                           │                                                       │
                           └────────────── (Asynchronous Replication) ─────────────┘
                                           - AWS: Aurora Storage-Level Physical Replication (< 1s lag)
                                           - OCI: Data Guard Redo Transport over Remote Peering (< 1s lag)
  ═══════════════════════════════════════════════════════  ═══════════════════════════════════════════════════════
                           │                                                       │
                           ▼                                                       ▼
   [ Object Storage: S3 Standard Primary ]                 [ Object Storage: S3 Standard Standby (CRR + RTC) ]
   [ OCI Object Storage Primary Bucket ]                   [ OCI Object Storage Cross-Region Replication ]
```

### 2.2 Dual-Cloud Component Mapping

| Architectural Function | AWS Native Primitive | OCI Native Primitive | Architectural Rationale |
| :--- | :--- | :--- | :--- |
| **Disaster Recovery Orchestrator**| Route 53 Application Recovery Controller (ARC) `[Doc: Route 53 ARC, checked 2026]` | OCI Full Stack Disaster Recovery (FSDR) `[Doc: OCI FSDR, checked 2026]` | Provides deterministic, audited failover execution plans across compute, database, storage, and networking with a single click/API call. |
| **Global DNS Traffic Steering** | Route 53 Failover Routing with Health Checks | OCI DNS Traffic Management Steering Policies (Failover) | Rapidly shifts DNS records to the secondary regional load balancer VIP upon regional disaster declaration. |
| **Relational DB Replication** | Amazon Aurora Global Database `[Doc: Aurora Global DB, checked 2026]` | OCI Active Data Guard (ADG) / Autonomous Data Guard `[Doc: OCI Data Guard, checked 2026]` | Delivers physical block/redo log replication across regions with typical replication latency $< 1\text{ second}$. |
| **Object Storage Replication** | Amazon S3 Cross-Region Replication (CRR) with S3 RTC | OCI Object Storage Cross-Region Replication | Replicates objects across regions asynchronously; S3 RTC enforces an SLA of 99.99% objects replicated within 15 minutes. |
| **Secrets & Keys Replication** | AWS KMS Multi-Region Keys (MRK) + Secrets Manager Multi-Region Replica | OCI Vault Cross-Region Key Replication + Secret Replication | Guarantees identical cryptographic key IDs and database credentials exist in both regions before failover. |
| **Immutable Compliance Vault** | AWS Backup Vault Lock (Compliance Mode) | OCI Immutable Retention Rules (Locked Buckets) | Prevents ransomware or rogue administrative actors from purging disaster recovery backups. |

---

## 3. Traffic Flow & Ingress Path (T)

### 3.1 Steady-State Traffic Routing
1. Global clients query `api.enterprise.com`.
2. **AWS Route 53 ARC**: Evaluates Routing Control states. The primary routing control rule is set to `ENABLED` (routing to `us-east-1`), while the secondary routing control rule is set to `DISABLED`.
3. **OCI DNS Steering**: Evaluates the Failover Steering Policy. The primary pool (`us-ashburn-1` LB VIP) is assigned Priority 1; the secondary pool (`us-phoenix-1` LB VIP) is assigned Priority 2.
4. Client requests are routed 100% to the Primary Region.

### 3.2 Automated vs. Operator-Invoked Failover Sequence

```text
====================================================================================================
                        DISASTER FAILOVER TIMELINE (< 15 MINUTES RTO)
====================================================================================================

[T = 00:00] Catastrophic disaster strikes Primary Region (Power / Subsea Fiber Severed)
[T = 01:00] Multi-region synthetic canary probes detect persistent 5xx errors & health check drop
[T = 02:00] Incident Commander / Automated ARC Trigger initiates Failover Sequence
[T = 02:30] STEP 1: FENCE PRIMARY REGION (Disable Primary Routing Control, revoke write credentials)
[T = 03:00] STEP 2: PROMOTE DATABASE
            - AWS: 'aws rds failover-global-cluster --global-cluster-identifier ...'
            - OCI: FSDR executes Data Guard Failover to Standby DB
[T = 04:30] Secondary Database promoted to Primary Writer; accepts read/write transactions
[T = 05:00] STEP 3: SCALE OUT COMPUTE FLEET
            - Karpenter / OCI Instance Pools scale standby pods from 20% to 100% capacity
[T = 08:00] Secondary compute fleet passes readiness probes; warm connection pool established
[T = 08:30] STEP 4: FLIP GLOBAL TRAFFIC STEERING
            - ARC enables Secondary Routing Control; OCI DNS Steering shifts VIP to Phoenix
[T = 10:00] Global DNS TTL dissipates; 95% of client traffic now terminating in Secondary Region
[T = 14:30] All asynchronous queues drained and fully reconciled. System operating normally.
====================================================================================================
```

---

## 4. Data Flow & Storage Engine (D)

### 4.1 Database Replication Mechanics (Aurora Global vs OCI Data Guard)

```text
+---------------------------------------------------------------------------------------------------+
| AWS AURORA GLOBAL DATABASE REPLICATION FLOW                                                       |
+---------------------------------------------------------------------------------------------------+
| Primary DB Compute (us-east-1)                                                                    |
|    │ Writes log records to 6-way storage nodes across 3 AZs                                       |
|    ▼                                                                                              |
| Aurora Storage Node (AZ-1) ──[ Dedicated AWS Backbone ]──► Aurora Secondary Storage (us-west-2)   |
| (Replication is physical storage-to-storage; zero impact on primary compute engine CPU)           |
|                                                            │                                      |
|                                                            ▼                                      |
|                                          Aurora Read Replica Compute Node (us-west-2)             |
+---------------------------------------------------------------------------------------------------+

+---------------------------------------------------------------------------------------------------+
| OCI ACTIVE DATA GUARD (ADG) REPLICATION FLOW                                                      |
+---------------------------------------------------------------------------------------------------+
| Primary Base DB / Autonomous (Ashburn)                                                            |
|    │ LGWR process sends redo vectors over FastConnect / Remote VCN Peering                        |
|    ▼                                                                                              |
| Standby Database (Phoenix)                                                                        |
|    │ RFS (Remote File Server) process writes to Standby Redo Logs                                 |
|    │ MRP (Managed Recovery Process) applies redo in real-time                                     |
|    ▼                                                                                              |
| Standby is open in READ-ONLY mode while continuous real-time apply runs in background             |
+---------------------------------------------------------------------------------------------------+
```

### 4.2 Handling Replication Lag and Unplanned Delta
During steady state, replication lag between regions is typically between $200\text{ms}$ and $800\text{ms}$.
- **Monitoring Lag**:
  - AWS CloudWatch metric: `AuroraGlobalDBReplicationLag`.
  - OCI Monitoring metric: `DataGuardLagSeconds`.
  - SRE Alert: If lag exceeds $10\text{ seconds}$ for more than 2 consecutive minutes, trigger P2 alert to investigate cross-region link saturation.
- **RPO Reconciliation Post-Failover**:
  - If failover was forced during an active primary outage, any transactions committed in the primary region during the final 500ms before disconnection were not yet replicated.
  - When the primary region eventually recovers, it must **never** be brought online as a writer (split-brain).
  - The recovered primary is booted as a replica and synchronized from the new primary, or delta transaction logs are extracted to reconcile missing records via an audited business ledger.

---

## 5. Security Architecture (S)

### 5.1 Split-Brain Fencing Architecture
The most dangerous failure mode in multi-region active-passive systems is **Split-Brain**: both regions believing they are the primary writer, causing conflicting transactions to be written concurrently to divergent databases.
- **Fencing Action 1: Network Route Severing**:
  - Revoke Ingress Load Balancer Security Group rules / NSG rules in the failed primary region via automation script, instantly dropping all external client ingress.
- **Fencing Action 2: IAM Workload Revocation**:
  - Update the IAM Policy / OCI Dynamic Group policy in the old primary region to deny `rds:ExecuteStatement` or `oci:db:write`.
- **Fencing Action 3: Database Observer Quorum**:
  - In OCI Data Guard, an external FSFO Observer node running in a neutral third region (`us-chicago-1` or multi-cloud) holds the tie-breaker vote, preventing split-brain promotion if only cross-region network links are partitioned.

### 5.2 Cryptographic Continuity with Multi-Region Keys (MRK)
If an application encrypts data in the primary region using an AWS KMS key or OCI Vault key, the secondary region cannot decrypt that data unless identical cryptographic key material exists.
- **AWS Implementation**: Deploy **KMS Multi-Region Keys (MRK)** (`mrk-111122223333...`). The key shares the same key ID, key material, and key ARN across `us-east-1` and `us-west-2`.
- **OCI Implementation**: Utilize OCI Vault Cross-Region Replication to synchronize HSM encryption keys across Ashburn and Phoenix compartments.

---

## 6. Reliability & High Availability (R)

### 6.1 Route 53 ARC Routing Controls vs OCI Full Stack DR (FSDR)
- **AWS Route 53 ARC**:
  - Operates across 5 distinct AWS regions as an independent control plane, guaranteeing that ARC itself remains operational even if `us-east-1` experiences total failure.
  - SREs execute failover by toggling **Routing Controls** with safety rules (e.g., asserting that Secondary can only be turned on if Primary is turned off, preventing simultaneous active states).
- **OCI Full Stack Disaster Recovery (FSDR)**:
  - Automates the step-by-step transition of all infrastructure tiers:
    1. Promotes the standby database via Data Guard.
    2. Re-attaches replicated Block Volumes to compute instances.
    3. Boots standby VM shapes in the target fault domain.
    4. Re-assigns backend sets in the OCI Flexible Load Balancer.
    5. Updates OCI DNS Traffic Management Steering policies.

---

## 7. Scaling & Capacity Planning (S)

### 7.1 Warm Standby Compute Fleet Sizing Strategy
Why 20% warm capacity instead of 0% (Cold Pilot Light) or 100% (Hot Standby)?
1. **0% (Cold Pilot Light)**: Booting an entire EKS/OKE cluster, downloading container images, and provisioning nodes from scratch takes 12 to 20 minutes, violating our 15-minute RTO SLA.
2. **100% (Hot Standby)**: Running 100% idle compute in the secondary region doubles monthly compute expenditures ($\$40,000+/\text{month}$) for capacity that sits unused 99.9% of the year.
3. **20% (Warm Standby Compromise)**:
   - Maintains warm JVM/container processes with pre-warmed connection pools to the standby database.
   - Allows synthetic health checks to validate end-to-end functionality continuously.
   - When failover occurs, Karpenter / OCI Instance Pools scale the fleet from 20% to 100% in **under 4 minutes** using pre-cached container images.

---

## 8. Observability & Production Telemetry (O)

### 8.1 Critical Disaster Recovery SLIs and Dashboards
1. **Cross-Region Replication Lag**:
   - P99 target: $< 1.0\text{ second}$. Alert threshold: $> 5.0\text{ seconds}$.
2. **Standby Health Canary**:
   - Synthetic ping executing every 30 seconds against the Secondary Region internal VIP.
   - Verifies that standby pods can perform read-only transactions against the local replica.
3. **DR Readiness Index (SRE Scorecard)**:
   - Metric evaluating: DB lag OK + Secrets synchronized + Warm compute alive + Route 53 ARC safety rules valid.
   - Must be $100\%$ green at all times.

---

## 9. Cloud Cost & Unit Economics (C)

### 9.1 Monthly FinOps BOM for Active-Passive Warm Standby

```text
====================================================================================================
               FINOPS COST ESTIMATE: ACTIVE-PASSIVE WARM STANDBY DISASTER RECOVERY
====================================================================================================

INFRASTRUCTURE TIER              PRIMARY REGION (100% LOAD)       SECONDARY REGION (WARM STANDBY)
----------------------------------------------------------------------------------------------------
Ingress CDN & WAF                $1,200                           $150 (Standby health checks)
Load Balancers                   $450                             $180 (Warm ALB/LB)
Compute Tier (K8s Pods)          $6,400 (50 Nodes)                $1,280 (10 Warm Nodes - 20%)
Database Tier (Aurora/DataGuard) $7,200 (Primary Writer 3-AZ)     $4,800 (Storage Replica Node)
Storage Replication (S3/Object)  $1,800 (Primary 50TB)            $1,800 (CRR Replica 50TB)
Cross-Region Data Egress Fee     $1,600 (Replication Bandwidth)   $0.00
Disaster Recovery Automation     $0.00                            $250 (Route 53 ARC / OCI FSDR)
----------------------------------------------------------------------------------------------------
REGIONAL MONTHLY SUBTOTAL        $18,650                          $8,460
COMBINED MULTI-REGION MONTHLY SPEND: ~$27,110 (A 45% premium over single-region for DR insurance)
====================================================================================================
```

---

## 10. Failure Modes & Cascades (F)

### 10.1 Scenario A: Split-Brain Network Partition Drill
- **Failure Condition**: Trans-continental communication between `us-east-1` and `us-west-2` drops, but both regions still have active internet connections.
- **System Defense**:
  - The secondary region's automated watchdog detects loss of replication heartbeat.
  - Automated promotion is **explicitly blocked** by Route 53 ARC quorum / OCI FSDR observer rules.
  - Secondary remains read-only unless human SRE on-call executes authenticated emergency override or external quorum confirms primary failure.

### 10.2 Scenario B: Unclean Primary Recovery Post-Disaster
- **Failure Condition**: 8 hours after failover to `us-west-2`, power is restored to `us-east-1` and the old primary database boots up automatically.
- **System Defense**:
  - Fencing rules implemented during failover revoked the old primary's IAM role and security group ingress.
  - Local database listener on old primary starts in restricted read-only mode.
  - Automated runbook executes reverse replication setup, converting old primary into a downstream replica of the new primary in `us-west-2`.

---

## 11. Trade-offs & Defense (T)

### 11.1 Key Architectural Compromises

```text
====================================================================================================
                                  ARCHITECTURAL TRADE-OFF MATRIX
====================================================================================================

DESIGN CHOICE                   CHOSEN OPTION              REJECTED ALTERNATIVE       TECHNICAL JUSTIFICATION
----------------------------------------------------------------------------------------------------
Multi-Region Topology           Active-Passive Warm        Active-Active Multi-Region Active-Active requires complex
                                Standby (20% Compute)      (Multi-Master Writes)      distributed conflict resolution
                                                                                      (CRDTs) and costs 2x more.
----------------------------------------------------------------------------------------------------
Replication Mode                Asynchronous Storage       Synchronous Cross-Region   Speed of light latency (~60ms
                                Replication (RPO < 1s)     Replication (RPO = 0)      RTT) makes synchronous cross-
                                                                                      region writes unacceptable.
----------------------------------------------------------------------------------------------------
Failover Initiation             Supervised Automated       Fully Autonomous Without   Fully autonomous failovers risk
                                (One-Click ARC Trigger)    Human Approval             false-positive split-brains
                                                                                      during transient network blips.
====================================================================================================
```

### 11.2 Bar-Raiser Defense Script

> **Interviewer**: *"Why accept an RPO of up to 1 minute and asynchronous replication instead of configuring synchronous replication to achieve absolute zero data loss (RPO = 0) during a regional failure?"*

**Candidate Defense**:
*"Enforcing synchronous cross-region replication is physically constrained by the speed of light in optical fiber. The round-trip time (RTT) between Virginia (us-east-1) and Oregon (us-west-2) is approximately 65 to 70 milliseconds. If we enforce synchronous replication on every database commit, every single write transaction in our application would incur an unavoidable 70ms latency penalty, degrading our P99 write latency SLA from 40ms to over 110ms and drastically reducing write throughput.*

*Furthermore, under synchronous replication, if the cross-region link experiences transient packet loss or jitter, all write operations in the primary region freeze completely, sacrificing local availability for remote consistency (CAP theorem).*

*By adopting asynchronous physical storage replication via Aurora Global Database or OCI Active Data Guard, primary commits remain sub-millisecond local operations, while replication lag remains under 1 second during normal operations. Accepting an RPO of under 1 minute for a catastrophic black swan regional failure allows us to maintain 99.999% intra-region availability and sub-40ms write performance 365 days a year."*
