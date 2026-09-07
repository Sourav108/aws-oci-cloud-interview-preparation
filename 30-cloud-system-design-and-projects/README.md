# Module 30: Cloud System Design & Reference Architectures

---

## 1. Module Overview & Learning Objectives

At the Senior, Staff, and Principal Cloud Architect level, system design interviews are not trivia contests about cloud console menus. They are rigorous, high-stakes evaluations of your ability to transform ambiguous business constraints into resilient, scalable, secure, and cost-effective distributed systems.

System design interviews at tier-1 technology firms, hyperscalers, and global enterprises evaluate:
- **Architectural Decomposition**: Can you untangle monoliths and sprawling workflows into clean, decoupled, bounded contexts?
- **Deterministic Traffic and Data Flows**: Can you trace a byte from Anycast edge DNS down through TLS termination, layer 7 reverse proxies, VPC/VCN peering, microservice pods, and distributed databases with ACID/BASE semantics?
- **Zero-Trust Hardening**: Do you default to least-privilege workload identities (IRSA / OCI Dynamic Groups), customer-managed KMS envelope encryption, and non-routable private topologies?
- **Failure Engineering & Blast Radius Isolation**: Do you engineer for the inevitability of AZ/AD failure, primary database failover, downstream circuit-tripping, and split-brain fencing?
- **FinOps & Unit Economics**: Can you justify compute, storage, and egress choices with rigorous cost calculations, spot/preemptible utilization, and auto-tiering policies?
- **Dual-Cloud Mastery**: Can you fluently articulate and defend production-grade architectures on both **Amazon Web Services (AWS)** and **Oracle Cloud Infrastructure (OCI)** without hand-waving?

This module synthesizes all 29 preceding modules into actionable, production-grade system designs structured strictly according to the battle-tested **11-Step Cloud Engineering Interview Framework (R-A-T-D-S-R-S-O-C-F-T)**.

---

## 2. The 11-Step System Design Framework (R-A-T-D-S-R-S-O-C-F-T)

Every system design lesson in this module adheres to this 11-step chronological delivery structure:

```text
====================================================================================================
                        THE 11-STEP CLOUD SYSTEM DESIGN FRAMEWORK
====================================================================================================

 [1. Requirements (R)]   --> Clarify Scope, Functional/Non-Functional SLAs, QPS, RPO/RTO
           │
 [2. Architecture (A)]   --> Dual-Cloud Component Topology, Boundary Segmentation, VPC/VCN Design
           │
 [3. Traffic Flow (T)]   --> Trace Client Ingress: Edge DNS, Anycast, CDN, WAF, ALB/OCI LB, Pods
           │
 [4. Data Flow (D)]      --> Relational vs NoSQL, Read/Write Paths, Replication Lag, Sharding
           │
 [5. Security (S)]       --> Zero-Trust, IRSA/Dynamic Groups, KMS Envelope Encryption, Private Endpoints
           │
 [6. Reliability (R)]    --> Multi-AZ/AD Isolation, Liveness/Readiness Probes, Circuit Breakers, FSFO
           │
 [7. Scaling (S)]        --> Horizontal Autoscaling (KEDA/HPA), Connection Pooling, Cache Invalidation
           │
 [8. Observability (O)]  --> Golden Signals, W3C TraceContext, OTel Exporters, Multi-Burn-Rate Alarms
           │
 [9. Cost (C)]           --> FinOps BOM, Egress Elimination, Spot/Preemptible Fleets, Auto-Tiering
           │
 [10. Failure Modes (F)] --> Cascading Failure Injections, Partition Drills, DB Crash Failovers
           │
 [11. Trade-offs (T)]    --> Defense of Design Compromises, Alternative Rejections, Bar-Raiser Defense
====================================================================================================
```

### Framework Step Breakdown

| Step | Focus Area | Critical Deliverables & Artifacts |
| :--- | :--- | :--- |
| **1. Requirements & Constraints (R)** | Functional & Non-Functional Scoping | Write/Read QPS, Peak multipliers, P95/P99 latency SLA, Availability ($99.99\% = 4.38\text{ min/month}$), RPO/RTO targets, Storage growth (TB/PB). |
| **2. High-Level Architecture (A)** | Dual-Cloud Topology | Mermaid / ASCII topology diagram showing edge, public ingress, private compute, and isolated data subnets for both AWS and OCI. |
| **3. Traffic Flow & Ingress Path (T)** | Packet Ingress & Routing | Route 53 / OCI DNS Steering $\to$ CloudFront/OCI CDN $\to$ AWS WAF/OCI WAF $\to$ ALB/OCI LB $\to$ EKS/OKE pods over VPC CNI / OCI VCN-Native CNI. |
| **4. Data Flow & Storage Engine (D)** | Persistence & Consistency | Aurora Global DB / OCI Autonomous DB / DynamoDB / OCI NoSQL, write paths, read paths, CDC pipelines (Kinesis/Kafka/Streaming), cross-region sync. |
| **5. Security Architecture (S)** | Zero-Trust & Identity | Credential-less workload identity (EKS Pod Identity/IRSA vs OCI Dynamic Groups), KMS customer-managed envelope encryption, Security Groups & NSGs. |
| **6. Reliability & High Availability (R)**| Fault Isolation & Resilience | 3-AZ / 3-AD + 3-Fault Domain topologies, health check differentiation (liveness vs readiness), circuit breakers (Envoy/Resilience4j), Data Guard FSFO. |
| **7. Scaling & Capacity Planning (S)** | Elasticity & Bottleneck Mitigation | HPA + Karpenter / OCI Instance Pools, connection pooling (RDS Proxy / PgBouncer), hot partition mitigation, Redis cache invalidation. |
| **8. Observability & Telemetry (O)** | Monitoring & SRE Signals | The 4 Golden Signals, OpenTelemetry collector sidecars, CloudWatch / OCI APM distributed tracing, multi-window multi-burn-rate SLO alerting. |
| **9. Cloud Cost & Unit Economics (C)** | FinOps & Bill of Materials | Itemized BOM for AWS and OCI, Savings Plans vs Universal Credits, eliminating cross-AZ/VCN data transfer fees, S3 Intelligent-Tiering / OCI Auto-Tiering. |
| **10. Failure Modes & Cascades (F)** | Failure Simulation & Chaos | AZ drop, primary DB ungraceful crash, downstream microservice brownout, Redis cluster failover, poison-pill event isolation. |
| **11. Trade-offs & Defense (T)** | Staff-Level Architectural Defense| Justifying Aurora vs DynamoDB, defending synchronous vs asynchronous cross-region replication, answering tough interviewer pushback. |

---

## 3. Scenario Catalog & Architectural Index

This module provides exhaustive, end-to-end architectures for the 5 most critical system design interview scenarios asked in cloud engineering interviews:

```
30-cloud-system-design-and-projects/
├── README.md                                             # This module roadmap and system design guide
├── 01-tier1-ha-rest-api-design.md                        # Scenario 1: High-Availability Tier-1 REST API
├── 02-event-driven-order-processing-platform.md          # Scenario 2: Asynchronous Event-Driven Order Processing
├── 03-globally-distributed-media-upload-platform.md      # Scenario 3: Globally Distributed Media Upload Platform
├── 04-multi-region-active-passive-dr-system.md           # Scenario 4: Multi-Region Active-Passive DR System
└── 05-realtime-financial-transaction-engine.md           # Scenario 5: Real-Time Financial Transaction Engine
```

### Scenario Mapping Matrix

| Scenario File | Architectural Archetype | Primary AWS Stack | Primary OCI Stack | Key Engineering Challenges |
| :--- | :--- | :--- | :--- | :--- |
| **01. Tier-1 HA REST API** | Synchronous, Low-Latency Web Service | Route 53 + CloudFront + ALB + EKS (Fargate/Node) + Aurora PG + ElastiCache Redis | OCI DNS + OCI CDN + OCI LB + OKE + OCI Autonomous Transaction Processing + OCI Cache | P99 $< 50\text{ms}$, connection pooling under burst, sub-second failover, zero downtime deployments. |
| **02. Event-Driven Order Processing** | Asynchronous, Distributed Event Bus & Worker Pool | API GW + SQS/SNS + Step Functions + Lambda/ECS + DynamoDB + S3 EventBridge | OCI API GW + OCI Streaming / OCI Queue + OCI Events + OCI Functions + OCI NoSQL | Exactly-once semantics, idempotency, dead-letter queue (DLQ) redrive, backpressure handling. |
| **03. Global Media Upload Platform** | Globally Distributed High-Throughput Ingestion | Route 53 Latency + CloudFront S3 Presigned URLs + S3 Transfer Acceleration + Lambda | OCI Traffic Steering + Pre-Authenticated Requests (PAR) + Object Storage + OCI Events | Multi-part upload chunking, resumable transfers, edge virus scanning, direct-to-storage bypass. |
| **04. Multi-Region Active-Passive DR** | High-Resilience Regional Failover System | Route 53 ARC + Aurora Global DB + S3 CRR + Backup Vault Lock + Cross-Region ASG | OCI Traffic Steering Failover + Full Stack DR (FSDR) + Remote Peering + Active Data Guard | RTO $< 15\text{ min}$, RPO $< 1\text{ min}$, split-brain prevention, automated health probes. |
| **05. Real-Time Financial Engine** | High-Throughput, ACID Compliant Transaction Core | MSK (Kafka) + EKS + Aurora I/O-Optimized Multi-AZ + DynamoDB Ledger + KMS HSM | OCI Streaming (Kafka API) + OKE + OCI Exadata Cloud Service + OCI Vault Dedicated HSM | Zero data loss (RPO = 0), distributed 2PC/Saga orchestration, PCI-DSS Level 1 compliance. |

---

## 4. Side-by-Side Dual-Cloud Primitives Catalog

| Architectural Domain | AWS Native Primitive | OCI Native Primitive | Key Comparative Trade-off |
| :--- | :--- | :--- | :--- |
| **Global Edge & DNS** | Route 53 (Anycast, Traffic Flow) | OCI DNS Steering Policies | Route 53 offers integrated health checks; OCI DNS offers native load balancing & geoproximity steering without extra policy licensing fees. |
| **Content Delivery & Edge Security** | CloudFront + AWS WAF + Shield | OCI CDN + OCI Web Application Firewall | CloudFront integrates natively with Lambda@Edge; OCI WAF provides edge-enforced OWASP Top 10 filtering with simple rule syntax. |
| **Layer 7 Load Balancing** | Application Load Balancer (ALB) | OCI Flexible Load Balancer | ALB charges per LCU (load balancer capacity unit); OCI LB provides dynamic bandwidth scaling (10 Mbps to 8000 Mbps) with predictable hourly pricing. |
| **Container Orchestration** | Elastic Kubernetes Service (EKS) | Oracle Cloud Infrastructure Kubernetes Engine (OKE) | EKS charges \$0.10/hour per cluster control plane; OKE provides free basic cluster control plane with self-healing master nodes. |
| **Serverless Compute** | AWS Lambda | OCI Functions (powered by Fn Project) | Lambda scales to thousands of concurrent executions in seconds; OCI Functions runs open-source Fn containers with native VCN ingress. |
| **Relational Database** | Aurora PostgreSQL / MySQL | OCI Autonomous Transaction Processing (ATP) / Base Database | Aurora utilizes a distributed 6-way quorum storage layer; OCI ATP runs on dedicated Exadata InfiniBand/RoCE storage delivering massive IOPS. |
| **Distributed NoSQL** | Amazon DynamoDB | OCI NoSQL Database | DynamoDB offers global tables and single-digit millisecond latency; OCI NoSQL offers flexible on-demand or provisioned read/write units. |
| **In-Memory Caching** | ElastiCache Redis / Valkey | OCI Cache with Redis | AWS ElastiCache supports multi-AZ auto-failover; OCI Cache provides fully managed Redis instances with zero-infrastructure overhead. |
| **Event Streaming & Queues** | Amazon SQS & SNS / Amazon MSK | OCI Queue & OCI Streaming (Kafka compatible) | SQS offers virtually unlimited scale and standard/FIFO queues; OCI Streaming provides high-throughput Kafka-compatible partitioned topics. |
| **Security & Workload Identity**| IAM Roles for Service Accounts (IRSA) / EKS Pod Identity | OCI Dynamic Groups + Matching Rules | IRSA leverages OIDC federation; OCI Dynamic Groups match instance/pod OCIDs directly within the tenancy identity domain. |
| **Disaster Recovery Automation**| Route 53 Application Recovery Controller (ARC) | OCI Full Stack Disaster Recovery (FSDR) | ARC provides zonal and regional routing control; OCI FSDR provides comprehensive end-to-end plan orchestration across compute, DB, and network. |

---

## 5. System Design Interview Rubric & Scoring Guide

When evaluating candidates for Senior (L5/IC4), Staff (L6/IC5), and Principal (L7/IC6) positions, interviewers look for specific competency signals:

```text
====================================================================================================
                             INTERVIEW SCORING RUBRIC
====================================================================================================

CRITERIA              MID-LEVEL (L4)           SENIOR (L5)              STAFF / PRINCIPAL (L6+)
----------------------------------------------------------------------------------------------------
Scoping & Limits      Waits for interviewer     Asks for QPS, latency,   Calculates throughput,
                      to give numbers.          data sizes, SLA.         bandwidth, storage curves;
                                                                         challenges assumptions.
----------------------------------------------------------------------------------------------------
Architectural Flow    Monolithic or standard    Decoupled tiers,         Zero-trust cell-based
                      3-tier template.          multi-AZ boundaries.     or event-driven architecture;
                                                                         bilingual cloud fluency.
----------------------------------------------------------------------------------------------------
Data Tier Design      Selects default RDBMS     Justifies NoSQL vs SQL   Analyzes consistency models,
                      without access patterns.  with access patterns.    replication lag, sharding,
                                                                         and CDC pipelines.
----------------------------------------------------------------------------------------------------
Reliability & Faults  Mentions "add more VMs".  Configures multi-AZ,     Injects split-brain, cascade
                                                health checks, backups.  failures, poison pills, and
                                                                         details mitigation runbooks.
----------------------------------------------------------------------------------------------------
FinOps & Economics    Ignores cost completely.  Mentions reserved        Calculates BOM, unit cost
                                                instances and spot.      per request, egress fees,
                                                                         and rightsizing economics.
----------------------------------------------------------------------------------------------------
Trade-off Defense     Defensive when poked;     Explains 1-2 pros/cons.  Proactively volunteers
                      cannot offer options.                              weaknesses and defends
                                                                         compromises mathematically.
====================================================================================================
```

---

## 6. How to Use This Module for Interview Preparation

1. **Simulate Real Time**: Give yourself 45 minutes per scenario. Start with a blank sheet of paper or whiteboard and follow the 11-step framework without looking at the lesson.
2. **Practice Bilingual Conversion**: If you are comfortable on AWS, force yourself to draw and describe the exact OCI equivalent, including specific networking primitives (LPGs, DRGs, SGWs) and identity constructs (Dynamic Groups).
3. **Master the Failure Injections**: Pay extreme attention to **Step 10 (Failure Modes & Cascades)** in each scenario. Senior and Staff bar-raisers spend up to 40% of the interview probing how your system breaks and recovers.
4. **Quantify Everything**: Never say "we scale horizontally when traffic spikes." Say: "We trigger an HPA scale-out when target request count exceeds 2,500 RPS per pod or P95 latency crosses 45ms, supported by Karpenter provisioning warm EC2/OCI nodes within 45 seconds."
