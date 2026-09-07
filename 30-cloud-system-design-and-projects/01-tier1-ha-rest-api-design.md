# System Design 01: Tier-1 High-Availability REST API Platform

---

## 1. Requirements & Constraints (R)

### 1.1 Business Context & Problem Statement
The enterprise requires a mission-critical, Tier-1 customer-facing REST API platform powering authentication, user profile management, product catalog metadata queries, and core transactional updates. Because this platform serves as the ingress backbone for web, mobile, and B2B partner integrations, any unplanned degradation directly impacts enterprise revenue and customer trust.

### 1.2 Functional Requirements
1. **CRUD API Operations**:
   - `POST /api/v1/resources`: Create a new transactional resource entity (idempotent submission).
   - `GET /api/v1/resources/{id}`: Retrieve a specific resource entity by unique identifier.
   - `PUT /api/v1/resources/{id}`: Full update of an existing entity with optimistic concurrency locking.
   - `GET /api/v1/resources?cursor={token}&limit={n}`: Paginated search and filtering over catalog items.
2. **Idempotency Enforcement**: Ensure that repeated or retried `POST` requests with the same `Idempotency-Key` header do not produce duplicate backend records or secondary side-effects.
3. **Authentication & Rate Limiting**: Validate JSON Web Tokens (JWT) at the network edge, enforcing per-tenant and per-client token bucket rate limiting.

### 1.3 Non-Functional Requirements & Quantitative SLAs
- **Availability Target**: **99.999% ("Five Nines")** annual availability across the region.
  - Allowed unplanned downtime: $\le 5.26\text{ minutes/year}$ ($\approx 26.3\text{ seconds/month}$).
- **Throughput & Traffic Volume**:
  - Baseline Read Traffic: **50,000 queries per second (QPS)**.
  - Peak Read Traffic (5x flash burst): **250,000 QPS**.
  - Baseline Write Traffic: **5,000 QPS**.
  - Peak Write Traffic (4x flash burst): **20,000 QPS**.
  - Read-to-Write Ratio: **10:1**.
- **Latency SLAs**:
  - Read Operations: $P95 < 15\text{ms}$, $P99 < 35\text{ms}$ at the API gateway layer.
  - Write Operations: $P95 < 40\text{ms}$, $P99 < 80\text{ms}$ inclusive of synchronous persistence.
- **Recovery Objectives (Intra-Region Blast Radius)**:
  - **Recovery Point Objective (RPO)**: $\mathbf{0}$ (Zero committed data loss during any single Availability Zone or Fault Domain loss).
  - **Recovery Time Objective (RTO)**: $\mathbf{< 10\text{ seconds}}$ automated failover for compute, $\mathbf{< 30\text{ seconds}}$ for managed database instances.
- **Data Retention & Storage Projections**:
  - Average payload size: $2\text{ KB}$ for read queries, $4\text{ KB}$ for write payloads.
  - Daily new data ingestion: $5,000\text{ writes/sec} \times 4\text{ KB} \times 86,400\text{ sec} \approx 1.728\text{ TB/day}$.
  - Annual relational data volume: $\approx 630\text{ TB/year}$.
  - Hot tier data (last 30 days): $\approx 52\text{ TB}$ cached and indexed.

---

## 2. High-Level Architecture (A)

### 2.1 Dual-Cloud Architectural Topology

The system is deployed in a strictly isolated 3-tier VPC/VCN topology spanning three distinct failure domains (3 Availability Zones in AWS; 3 Availability Domains or 3 Fault Domains in OCI).

```
========================================================================================================================
                                TIER-1 HA REST API ARCHITECTURE TOPOLOGY
========================================================================================================================

                                  [ Global Clients (Web / Mobile / B2B) ]
                                                     │
                                                     ▼
                            [ Edge Anycast DNS: Route 53 / OCI DNS Steering ]
                                                     │
                                                     ▼
                             [ Edge CDN + WAF: CloudFront / OCI CDN + WAF ]
                                                     │ (TLS 1.3 / mTLS / Anycast Edge)
                                                     ▼
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  VPC / VCN BOUNDARY: 10.100.0.0/16
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   PUBLIC SUBNETS (Ingress Tier) - 3 AZs / 3 ADs
   [ AWS ALB / OCI Flexible Load Balancer (Public IPs, TLS Termination, Health Checks) ]
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
                                                     │ (Private IP Forwarding)
                                                     ▼
   PRIVATE SUBNETS (Application Compute Tier) - Non-Routable 10.100.16.0/20, 10.100.32.0/20, 10.100.48.0/20
   ┌───────────────────────────────────┬───────────────────────────────────┬───────────────────────────────────┐
   │ AZ-1 / AD-1 (FD-1)                │ AZ-2 / AD-2 (FD-2)                │ AZ-3 / AD-3 (FD-3)                │
   │                                   │                                   │                                   │
   │ [ EKS Pod / OKE Container Pod ]   │ [ EKS Pod / OKE Container Pod ]   │ [ EKS Pod / OKE Container Pod ]   │
   │  - Envoy Sidecar / OTel Agent     │  - Envoy Sidecar / OTel Agent     │  - Envoy Sidecar / OTel Agent     │
   │  - REST API Engine (Go / Rust)    │  - REST API Engine (Go / Rust)    │  - REST API Engine (Go / Rust)    │
   └─────────────────┬─────────────────┴─────────────────┬─────────────────┴─────────────────┬─────────────────┘
                     │                                   │                                   │
                     ▼                                   ▼                                   ▼
   PRIVATE ISOLATED SUBNETS (Cache & Connection Pool Tier) - 10.100.64.0/20, 10.100.80.0/20, 10.100.96.0/20
   ┌───────────────────────────────────────────────────────────────────────────────────────────────────────────┐
   │ [ In-Memory Cache: AWS ElastiCache Valkey/Redis Cluster (Multi-AZ) / OCI Cache with Redis ]               │
   │ [ Connection Pool Layer: AWS RDS Proxy / PgBouncer on OKE Compute ]                                       │
   └───────────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                     │
                                                     ▼
   ISOLATED DATA SUBNETS (Persistence Tier - Zero Internet Access) - 10.100.112.0/20, 10.100.128.0/20, 10.100.144.0/20
   ┌───────────────────────────────────┬───────────────────────────────────┬───────────────────────────────────┐
   │ AZ-1 / AD-1 (Primary Writer)      │ AZ-2 / AD-2 (Sync Standby)        │ AZ-3 / AD-3 (Read Replica)        │
   │ [ Aurora PostgreSQL /             │ [ Aurora Storage Quorum Node /    │ [ Aurora Read Replica /           │
   │   OCI Autonomous Transaction DB ] │   OCI Autonomous Local Standby ]  │   OCI Base DB Read Replica ]      │
   └───────────────────────────────────┴───────────────────────────────────┴───────────────────────────────────┘
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
```

### 2.2 Dual-Cloud Component Mapping

| Architectural Layer | AWS Native Implementation | OCI Native Implementation | Selection Rationale |
| :--- | :--- | :--- | :--- |
| **Global Edge & DNS** | Route 53 (Anycast, Latency-based Routing) `[Doc: Route 53, checked 2026]` | OCI Traffic Management Steering Policies (Load Balancing & Geolocation) `[Doc: OCI DNS, checked 2026]` | Anycast resolution routes client requests to nearest cloud edge point of presence (PoP) in $< 10\text{ms}$. |
| **Content Delivery & WAF**| Amazon CloudFront + AWS WAF (Managed Rulesets) | OCI CDN + OCI Web Application Firewall (WAF) | Blocks volumetric DDoS (SYN floods, UDP amplification) and OWASP Top 10 threats before entering VPC/VCN. |
| **Ingress Load Balancing** | Application Load Balancer (ALB) across 3 AZs | OCI Flexible Load Balancer (100 Mbps - 8000 Mbps shape) | Terminate TLS 1.3, execute HTTP/2 multiplexing, strip client headers, run Layer-7 health probing. |
| **Container Compute** | Amazon EKS (Managed Node Groups across 3 AZs) | OCI Kubernetes Engine (OKE) with Virtual Nodes | Kubernetes provides declarative container lifecycle, horizontal pod autoscaling (HPA), and zero-downtime rolling updates. |
| **In-Memory Caching Tier**| Amazon ElastiCache for Redis / Valkey (Cluster Mode) | OCI Cache with Redis (3-node cluster) | Caches read-heavy catalog data and stores short-lived idempotency tokens ($TTL = 86,400\text{s}$) with sub-millisecond retrieval. |
| **Connection Pooling** | AWS RDS Proxy (Managed Serverless Pooling) | PgBouncer deployed as high-availability daemonset on OKE | Prevents application autoscaling from exhausting database connection limits (`max_connections`). |
| **Relational Persistence** | Amazon Aurora PostgreSQL (I/O-Optimized, Multi-AZ) | OCI Autonomous Transaction Processing (ATP) / Exadata | Delivers ACID transaction semantics, multi-AZ synchronous replication quorum, and sub-30-second automated failover. |
| **Secrets & Keys** | AWS Secrets Manager + AWS KMS (Customer Managed Key) | OCI Vault + KMS (Hardware Security Module backed) | Hardware-enforced envelope encryption and automated secret rotation without application downtime. |

---

## 3. Traffic Flow & Ingress Path (T)

### 3.1 Step-by-Step Packet Traversal

```text
Step 1: Client Query --> Anycast DNS Resolution (Route 53 / OCI DNS Steering)
Step 2: Client TCP + TLS 1.3 Handshake terminated at CDN Edge (CloudFront / OCI CDN)
Step 3: Edge WAF Token & Signature Inspection (AWS WAF / OCI WAF)
Step 4: CDN Origin Forwarding over Cloud Private Backbone to Public Subnets
Step 5: Layer-7 Ingress Load Balancer (ALB / OCI Flexible LB) Target Evaluation
Step 6: Ingress Gateway / Pod Ingress Traversal into Private Worker Subnet
Step 7: Envoy Reverse Proxy Routing & JWT Workload Authorization
Step 8: Application Handler Execution & Response Streaming
```

1. **Edge Resolution & Geoproximity**:
   - The client performs a DNS query for `api.enterprise.com`.
   - **AWS**: Route 53 utilizes Anycast name servers with Latency-Based Routing (LBR) records, resolving the user to the nearest CloudFront Edge Location.
   - **OCI**: OCI DNS Traffic Management Steering uses Geolocation Steering rules to map the client IP subnet to the geographically closest OCI CDN Edge POP.
2. **TLS 1.3 Termination & DDoS Mitigation**:
   - TLS 1.3 handshake terminates at the edge CDN. Modern cipher suites (`TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256`) enforce Perfect Forward Secrecy (PFS).
   - AWS Shield Advanced and OCI DDoS Protection scrub Layer-3/4 volumetric attacks (SYN floods, ACK reflection) without customer origin involvement.
   - AWS WAF / OCI WAF inspects HTTP headers, evaluating:
     - Rate-limiting rules: Max 2,000 requests per 5-minute rolling window per client IP.
     - Known Bad IP reputation lists and SQL injection / XSS signatures.
3. **Origin Ingress over Private Cloud Backbone**:
   - The CDN establishes persistent HTTP/2 keep-alive connections to the regional Load Balancer across dedicated backbone links (AWS Direct Connect / OCI FastConnect edge interconnects).
   - Ingress Load Balancers:
     - **AWS**: Internet-facing Application Load Balancer (ALB) distributed across 3 Public Subnets (`subnet-pub-az1`, `subnet-pub-az2`, `subnet-pub-az3`).
     - **OCI**: Public Flexible Load Balancer distributed across 3 Availability Domains or Fault Domains (`subnet-pub-ad1`, `subnet-pub-ad2`, `subnet-pub-ad3`).
4. **Target Routing & Kubernetes Ingress**:
   - ALB/OCI LB forwards HTTP traffic across the internal subnet boundary to worker nodes hosting an Ingress Controller (AWS Load Balancer Controller target-type `ip` routing directly to Pod ENIs; OCI Native Ingress Controller using OCI VCN-Native CNI).
   - Traversal bypasses intermediate `kube-proxy` NAT hops, cutting internal network latency by $2\text{--}4\text{ms}$.

---

## 4. Data Flow & Storage Engine (D)

### 4.1 Polyglot Data Strategy & Access Patterns

```text
+-----------------------------------------------------------------------------------------------+
|                                    APPLICATION WORKER POD                                     |
+-----------------------------------------------------------------------------------------------+
           │                                                                 │
   [Read Operation]                                                  [Write Operation]
           │                                                                 │
           ▼                                                                 ▼
+---------------------+    Cache Hit (92%)     +------------------+    1. Lock Idempotency Key
|   Check In-Memory   |----------------------->| Return Cached    |       in Redis (SET NX EX)
|   Cache (Redis)     |                        | Data (< 2ms)     |    2. Begin ACID Transaction
+---------------------+                        +------------------+    3. Write to Primary DB
           │ Cache Miss (8%)                                           4. Commit DB Transaction
           ▼                                                           5. Invalidate / Update
+---------------------+    Hydrate Cache                                  Cache Key in Redis
| Read Replica DB     |-----------------------> [ Async Redis Put ]
| (Aurora / OCI ATP)  |
+---------------------+
```

### 4.2 Read Path Architecture (Cache-Aside Pattern)
1. Application receives `GET /api/v1/resources/{id}`.
2. App queries the distributed in-memory cache cluster:
   - Key format: `entity:v1:{tenant_id}:{resource_id}`.
   - **Cache Hit**: Data returned to client in $< 1.5\text{ms}$.
   - **Cache Miss**: Application acquires a local mutex (preventing stampede), executes a read query against the database **Read Replica endpoint**, returns the record to the caller, and asynchronously populates the cache with a randomized jittered TTL ($3,600\text{s} \pm 300\text{s}$).

### 4.3 Write Path Architecture & Idempotent Commit Pipeline
1. Client issues `POST /api/v1/resources` with header `Idempotency-Key: 8b7d92c1-3f1a-4d92-9387-9b2f6b8c9d10`.
2. Application evaluates idempotency in Redis using atomic `SET resource:idem:{key} "PROCESSING" EX 120 NX`:
   - If Redis returns `nil` (key already exists), query the status:
     - If `"PROCESSING"`, return `HTTP 409 Conflict` (`"Request currently processing, please retry"`).
     - If `"COMPLETED:{response_id}"`, return the cached response payload immediately without hitting the database.
3. Open an ACID transaction on the Primary Database:
   - Insert entity record into `resources` table.
   - Insert idempotency record into `idempotency_audit` table.
   - Write commit log.
4. Database engine acknowledges commit:
   - **AWS Aurora**: Writes redo log records across **6 storage nodes distributed across 3 AZs** (4-of-6 write quorum). Acknowledgment is sent to the compute node as soon as 4 storage nodes persist the log record.
   - **OCI Autonomous DB / Exadata**: Commits redo entries to persistent memory (PMEM) / NVMe storage across 3 storage servers via Remote Direct Memory Access (RDMA) over Converged Ethernet (RoCE), achieving sub-millisecond commit latency.
5. Application updates Redis key to `"COMPLETED:{response_id}"` with a 24-hour expiration (`EX 86400`).

### 4.4 Data Model & Indexing Strategy (PostgreSQL / Autonomous DB)

```sql
-- Core Resource Table
CREATE TABLE resources (
    resource_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    resource_name   VARCHAR(255) NOT NULL,
    status          VARCHAR(32) NOT NULL DEFAULT 'ACTIVE',
    version         INTEGER NOT NULL DEFAULT 1,
    metadata        JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Idempotency Ledger Table for Failover Durability
CREATE TABLE idempotency_ledger (
    idempotency_key VARCHAR(128) NOT NULL,
    tenant_id       UUID NOT NULL,
    response_code   INTEGER NOT NULL,
    response_body   JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, idempotency_key)
);

-- Composite Index for Fast Multi-Tenant Filtering
CREATE INDEX idx_resources_tenant_status ON resources (tenant_id, status, updated_at DESC);
```

---

## 5. Security Architecture (S)

### 5.1 Zero-Trust Network Perimeter & Micro-Segmentation
- **No Public IPs on Compute**: Compute instances and container pods reside exclusively in private subnets with RFC 1918 addresses.
- **Strict Ingress Filtering**:
  - Load Balancer Security Group / NSG allows ingress only from CloudFront/OCI CDN IP prefix lists on TCP port 443.
  - Compute Pod Security Group / NSG allows ingress only from Load Balancer Security Group on application port 8080.
  - Database Security Group / NSG allows ingress only from Compute/PgBouncer Security Group on TCP port 5432 / 1521.
  - Zero egress to the public internet: Outbound traffic to cloud APIs (KMS, Secrets Manager, CloudWatch, OCI Object Storage) traverses **VPC Endpoints (AWS PrivateLink)** or **OCI Service Gateways**.

### 5.2 Credential-less Workload Identity

```text
+---------------------------------------------------------------------------------------+
| AWS EKS: Pod Identity / IRSA                    OCI OKE: Workload Identity            |
+---------------------------------------------------------------------------------------+
| 1. Pod ServiceAccount annotated with            1. OKE Pod mounts Workload Identity   |
|    IAM Role ARN.                                   Token projected volume.            |
| 2. EKS OIDC identity provider generates         2. Tenancy IAM Dynamic Group matches  |
|    short-lived AWS STS credentials.                pod namespace and service account. |
| 3. Application SDK uses STS token to call       3. App calls OCI Vault API via local  |
|    KMS & Secrets Manager. Zero hardcoded keys.     principal; zero credentials on disk.|
+---------------------------------------------------------------------------------------+
```

#### AWS IAM Policy (Least-Privilege Pod Identity)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowKMSDecrypt",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:111122223333:key/mrk-tier1-api-key"
    },
    {
      "Sid": "AllowSecretsManagerRead",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:111122223333:secret:tier1/api/db-*"
    }
  ]
}
```

#### OCI Dynamic Group & Matching Rule
```text
# Dynamic Group: dg-tier1-api-workload
All {
    instance.compartment.id = 'ocid1.compartment.oc1..aaaaaaaaxxx',
    request.principal.type = 'workload',
    request.principal.service_account = 'tier1-api-sa',
    request.principal.namespace = 'production'
}

# IAM Policy:
Allow dynamic-group dg-tier1-api-workload to use vaults in compartment Production where target.vault.id = 'ocid1.vault.oc1..'
Allow dynamic-group dg-tier1-api-workload to read secret-bundles in compartment Production where target.secret.name = 'tier1-db-credentials'
```

---

## 6. Reliability & High Availability (R)

### 6.1 Multi-Domain Blast Radius Containment
- **AWS Deployment**: 3 Availability Zones (`us-east-1a`, `us-east-1b`, `us-east-1c`).
  - Pods scheduled using Kubernetes `topologySpreadConstraints` with `maxSkew: 1` across `topology.kubernetes.io/zone`.
  - Node groups span 3 AZs; if a physical facility experiences catastrophic power failure, Kubernetes scheduler automatically rebalances pods across remaining zones.
- **OCI Deployment**: 3 Availability Domains (in multi-AD regions like Ashburn/Phoenix) or 3 Fault Domains (FD-1, FD-2, FD-3 in single-AD regions).
  - Node pools configured with `fault_domains = ["FAULT-DOMAIN-1", "FAULT-DOMAIN-2", "FAULT-DOMAIN-3"]`.

### 6.2 Health Check Topology & Probing Differentiation
1. **Liveness Probe**:
   - Path: `/healthz/live`
   - Purpose: Verifies internal process thread pool responsiveness.
   - Behavior: Returns `HTTP 200` if Go/Rust runtime is responsive. If thread pool is deadlocked, returns timeout; kubelet terminates and restarts the container.
2. **Readiness Probe**:
   - Path: `/healthz/ready`
   - Purpose: Verifies that the pod can actively serve external traffic.
   - Behavior: Performs shallow checks against local database connection pool and Redis socket. If connection pool is saturated, returns `HTTP 503`. The Ingress Load Balancer stops routing requests to this pod without killing the process.

### 6.3 Automated Database Failover Mechanics

```text
====================================================================================================
                        DATABASE FAILOVER TIMELINE COMPARISON
====================================================================================================

AWS AURORA FAILOVER SEQUENCE:
[T=0s] Primary Aurora Node crashes (Hardware/Kernel failure)
[T=3s] Aurora distributed storage layer detects loss of compute heartbeat
[T=8s] Standby Replica in AZ-2 promoted to Primary Writer status
[T=12s] Cluster DNS CNAME updated to point to new writer instance IP
[T=15s] AWS RDS Proxy detects socket reset, buffers in-flight requests, switches to new Primary
[T=18s] Total application disruption window: ~15-20 seconds (Zero connection drops via RDS Proxy)

OCI DATA GUARD / AUTONOMOUS DB FAILOVER SEQUENCE:
[T=0s] Primary Database node loses heartbeat
[T=2s] Data Guard Fast-Start Failover (FSFO) Observer detects loss of primary
[T=6s] Standby database in AD-2 / FD-2 promoted to Primary role
[T=8s] Fast Application Notification (FAN) event broadcast to client connection pool (UCP/Hikari)
[T=10s] Client sockets re-routed instantly via VIP/SCAN listener; zero DNS propagation delay
[T=12s] Total application disruption window: < 12 seconds
====================================================================================================
```

---

## 7. Scaling & Capacity Planning (S)

### 7.1 Horizontal Autoscaling Architecture
- **Pod Layer**:
  - Horizontal Pod Autoscaler (HPA) configured to scale on composite metrics:
    1. Average CPU utilization target: **65%**.
    2. Custom Prometheus/CloudWatch metric: `http_requests_per_second_per_pod` target: **2,500 RPS**.
    3. P95 latency threshold: If P95 $> 25\text{ms}$, trigger immediate scale-out.
- **Node Layer**:
  - **AWS**: Karpenter autoscaler monitors unschedulable pod events and provisions right-sized EC2 instances (`c6i.2xlarge`, `c7g.2xlarge`) within $45\text{ seconds}$.
  - **OCI**: OKE Cluster Autoscaler provisions OCI Flexible Compute shapes (`VM.Standard3.Flex` with 8 OCPUs / 64 GB RAM) across fault domains.

### 7.2 Database Connection Pool Preservation

```text
50,000 Read QPS + 5,000 Write QPS
                 │
                 ▼
  [ 150 Kubernetes Pods ] (Each running 50 application threads = 7,500 connections)
                 │
                 ▼ (WITHOUT PROXY: DB crashes with "FATAL: sorry, too many clients already")
                 │
  [ RDS Proxy / PgBouncer Layer ] (Multiplexes 7,500 client connections down to 150 warm DB sockets)
                 │
                 ▼
  [ Primary DB max_connections = 300 ] (Stable memory, 0% connection exhaustion risk)
```

- **RDS Proxy / PgBouncer Configuration**:
  - Session pinning avoidance: Queries use transaction-level pooling (`pool_mode = transaction`).
  - Warm connection cache: Maintains 100 idle, pre-authenticated connections to Aurora/Base DB, eliminating TCP and TLS handshake overhead on burst traffic.

---

## 8. Observability & Production Telemetry (O)

### 8.1 The Four Golden Signals Implementation
1. **Latency**:
   - Monitored at ALB/OCI LB and Pod level: P50, P90, P95, P99, P99.9.
   - Alert threshold: P95 $> 25\text{ms}$ sustained for 3 consecutive minutes.
2. **Traffic**:
   - Ingress QPS per route (`/api/v1/resources`).
   - Monitored via CloudWatch Metric Math / OCI Monitoring MQL.
3. **Errors**:
   - HTTP status codes segmented by class: `4xx` (client errors), `5xx` (server faults).
   - Target: $5xx\text{ Error Rate} < 0.001\%$ ($1$ in $100,000$).
4. **Saturation**:
   - Connection pool utilization (Active Connections / Max Available).
   - CPU and memory utilization on worker nodes and database instances.

### 8.2 Multi-Window Multi-Burn-Rate SLO Alerting
For our **99.999% Availability SLO** over a 30-day rolling window, the error budget is $0.001\%$ ($10\text{ ppm}$).

```text
====================================================================================================
                             MULTI-BURN-RATE ALERT MATRIX
====================================================================================================

SEVERITY    BURN RATE    BUDGET CONSUMED    LONG WINDOW    SHORT WINDOW    PAGER RESPONSE
----------------------------------------------------------------------------------------------------
Critical    14.4x        2.0% in 1 hour     1 Hour         5 Minutes       Immediate Page (P1)
Warning     6.0x         5.0% in 6 hours    6 Hours        30 Minutes      Ticket / Slack (P2)
Notice      1.0x         10% in 3 days      3 Days         3 Hours         Daily Triage
====================================================================================================
```

### 8.3 Distributed Tracing Configuration (OpenTelemetry)
- Each request injects W3C TraceContext headers (`traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`).
- OpenTelemetry Collector runs as a daemonset, exporting traces to **AWS X-Ray** or **OCI Application Performance Monitoring (APM)** with a 5% adaptive head-based sampling rate (100% sampling on HTTP $5xx$ responses).

---

## 9. Cloud Cost & Unit Economics (C)

### 9.1 Monthly Bill of Materials (BOM) for 50,000 Baseline QPS

```text
====================================================================================================
                      MONTHLY FINOPS ESTIMATE (AWS vs OCI at 50,000 QPS)
====================================================================================================

INFRASTRUCTURE COMPONENT         AWS ARCHITECTURE (MONTHLY)       OCI ARCHITECTURE (MONTHLY)
----------------------------------------------------------------------------------------------------
Ingress CDN & WAF                $1,250 (CloudFront + WAF)        $720 (OCI CDN + WAF)
Load Balancers                   $480 (3 ALBs + LCU usage)        $280 (Flexible LB 4Gbps)
Compute Tier (K8s Nodes)         $4,800 (40 c6i.2xlarge 1-yr SP)  $3,100 (E4.Flex 320 OCPUs)
Database Tier                    $6,200 (db.r6g.4xlarge 3-AZ)     $4,400 (Base DB Extreme Perf)
In-Memory Cache (Redis)          $1,150 (cache.r6g.2xlarge 3-AZ)  $780 (OCI Cache 3-node)
Data Transfer & PrivateLink      $1,400 (Cross-AZ + VPC Endpoints)$220 (10TB Free Egress + SGW)
Observability (Logs/Metrics)     $1,600 (CloudWatch + X-Ray)      $850 (OCI Logging + APM)
----------------------------------------------------------------------------------------------------
TOTAL ESTIMATED MONTHLY COST     $16,880                          $10,350
UNIT COST PER 1M REQUESTS        ~$0.130 per 1M req               ~$0.079 per 1M req
====================================================================================================
```

### 9.2 FinOps Optimization Strategies
1. **Compute Rightsizing**: Leverage 3-Year Compute Savings Plans (AWS) and Annual Universal Credits Commitments (OCI) to slash baseline compute costs by $42\%$.
2. **Data Transfer Minimization**:
   - Keep chatty microservice communications within the same subnet/AZ where possible.
   - Utilize VPC Endpoints (AWS PrivateLink) and OCI Service Gateways to eliminate NAT Gateway data processing fees ($\$0.045/\text{GB}$ on AWS).
3. **Database I/O Optimization**: Utilize **Aurora I/O-Optimized** storage to eliminate unpredictable per-request I/O billing charges during high-volume read bursts.

---

## 10. Failure Modes & Cascades (F)

### 10.1 Scenario A: Catastrophic Availability Zone / AD Outage
- **Failure Condition**: A fiber cut or power grid failure takes down AWS `us-east-1a` or OCI `AD-1`.
- **System Impact**: 33% of compute pods, one ALB/LB IP, and the primary database writer node are abruptly terminated.
- **Automated Mitigation Flow**:
  1. ALB/OCI LB health checks fail within 5 seconds; traffic is redistributed to healthy listeners in AZ-2 and AZ-3.
  2. Aurora/Data Guard detects heartbeat loss and initiates automated failover to the replica in AZ-2 (failover time $< 20\text{s}$).
  3. Karpenter / OKE Cluster Autoscaler detects unassigned pods and spins up replacement nodes in AZ-2 and AZ-3.
  4. Client impact: P99 latency spikes to $180\text{ms}$ for 20 seconds; zero data loss (RPO = 0).

### 10.2 Scenario B: Redis Cache Tier Catastrophic Crash (Stampede)
- **Failure Condition**: Memory fragmentation causes all Redis primary nodes to evict simultaneously or restart.
- **System Impact**: 100% of read traffic (50,000 QPS) bypasses the cache and floods the relational database, threatening total connection pool saturation and CPU collapse.
- **Automated Mitigation Flow**:
  1. Application implements **Probabilistic Early Expiration (XFetch algorithm)** and an in-process local Mutex lock (`singleflight` in Go).
  2. Only 1 worker pod is permitted to query the database per cache key miss; all other concurrent requests wait on a local channel for the single query to resolve.
  3. RDS Proxy / PgBouncer queues excess database connections, throttling execution to match database CPU saturation limits.
  4. Database CPU rises from 35% to 75%, but never exceeds capacity; P99 latency degrades gracefully to $45\text{ms}$ while cache re-hydrates.

### 10.3 Scenario C: Downstream Third-Party Dependency Timeout Storm
- **Failure Condition**: An external payment verification or credit check API begins timing out with a 30-second latency tail.
- **System Impact**: Application worker threads block waiting for HTTP responses, exhausting the container thread pool and triggering widespread HTTP 504 gateway timeouts.
- **Automated Mitigation Flow**:
  1. Envoy sidecar / Resilience4j **Circuit Breaker** evaluates error rate over a sliding window of 100 requests.
  2. If timeout rate exceeds 20%, the circuit trips to `OPEN` state.
  3. Subsequent requests fail fast within $< 2\text{ms}$ returning a graceful fallback response or cached default without consuming backend threads.
  4. After 30 seconds, circuit enters `HALF-OPEN` sending 5% of traffic to test downstream recovery.

---

## 11. Trade-offs & Defense (T)

### 11.1 Staff-Level Architectural Trade-Off Decisions

```text
====================================================================================================
                                  ARCHITECTURAL TRADE-OFF MATRIX
====================================================================================================

DESIGN CHOICE                   CHOSEN OPTION              REJECTED ALTERNATIVE       TECHNICAL JUSTIFICATION
----------------------------------------------------------------------------------------------------
Relational vs NoSQL             Aurora PostgreSQL /        DynamoDB / OCI NoSQL       Complex multi-table joins,
                                OCI Autonomous DB                                     ACID financial integrity,
                                                                                      and strict schema validation
                                                                                      outweigh pure key-value scale.
----------------------------------------------------------------------------------------------------
Compute Engine                  EKS / OKE (Containers)     AWS Lambda / OCI Functions Cold starts (200ms-1.5s)
                                                                                      violate the P99 < 35ms SLA;
                                                                                      high sustained 50k QPS makes
                                                                                      containers 65% more economical.
----------------------------------------------------------------------------------------------------
Caching Topology                Distributed Redis          Local In-Memory Cache      Local cache causes data
                                (ElastiCache / OCI Cache)  (Guava / Go sync.Map)      inconsistency across 150 pods;
                                                                                      distributed cache ensures
                                                                                      immediate invalidation.
----------------------------------------------------------------------------------------------------
Connection Management           RDS Proxy / PgBouncer      Direct Pod DB Connections Direct connections crash DB
                                                                                      during horizontal pod scale-outs
                                                                                      (7,500 connections > max DB limit).
====================================================================================================
```

### 11.2 Defending the Architecture Under Bar-Raiser Scrutiny

> **Interviewer**: *"Why not use AWS DynamoDB or OCI NoSQL to easily achieve 250,000 QPS with single-digit millisecond latency without worrying about connection pooling or relational failover?"*

**Candidate Defense**:
*"While DynamoDB and OCI NoSQL excel at flat key-value queries and horizontal partition scaling, our Tier-1 REST API platform must support complex catalog search filtering across multiple dynamic attributes, relational integrity between tenants and sub-resources, and strict ACID transaction rollbacks. Modeling these relational access patterns in NoSQL requires either complex single-table design with denormalization—which creates severe write amplification and consistency anomalies during partial updates—or building custom distributed transaction logic in application code.*

*By selecting Amazon Aurora PostgreSQL and OCI Autonomous Transaction Processing, we achieve ACID durability with sub-millisecond commit latency via cloud-native storage engines (Aurora's 6-way quorum storage and OCI's Exadata RoCE PMEM). We achieve the required 250,000 read QPS by offloading 92% of queries to an ElastiCache/OCI Cache tier and scaling database read replicas across 3 Availability Zones. Connection pooling via RDS Proxy and PgBouncer guarantees that container scale-outs never exhaust database sockets."*
