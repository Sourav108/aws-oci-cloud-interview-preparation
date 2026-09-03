# Cloud Architecture Design Patterns Catalog

This document details 16 fundamental production cloud design patterns for senior and staff engineering interviews. Every pattern is analyzed across both **AWS** and **OCI**, dissecting structural trade-offs, failure modes, and interview defense strategies.

---

## 📑 Catalog Index

1. [Classic 3-Tier Web Application](#1-classic-3-tier-web-application)
2. [Load-Balanced Microservices Tier](#2-load-balanced-microservices-tier)
3. [Private Application Tier with Egress Filtering](#3-private-application-tier-with-egress-filtering)
4. [Isolated Database Tier in Dedicated Subnets](#4-isolated-database-tier-in-dedicated-subnets)
5. [Event-Driven Architecture (EDA)](#5-event-driven-architecture-eda)
6. [Serverless Web & API Architecture](#6-serverless-web--api-architecture)
7. [Asynchronous Queue-Worker Pattern](#7-asynchronous-queue-worker-pattern)
8. [Distributed Cache-Aside Pattern](#8-distributed-cache-aside-pattern)
9. [Edge CDN-Backed Static & Dynamic Acceleration](#9-edge-cdn-backed-static--dynamic-acceleration)
10. [Blue/Green Immutable Deployments](#10-bluegreen-immutable-deployments)
11. [Canary Deployments with Automated Rollback](#11-canary-deployments-with-automated-rollback)
12. [Multi-Availability Zone Active-Active Tier](#12-multi-availability-zone-active-active-tier)
13. [Multi-Region Disaster Recovery (Active-Passive / Warm Standby)](#13-multi-region-disaster-recovery-active-passive--warm-standby)
14. [Multi-Region Active-Active with Global Routing](#14-multi-region-active-active-with-global-routing)
15. [Zero-Trust Cloud Network Architecture](#15-zero-trust-cloud-network-architecture)
16. [Centralized Observability & Telemetry Mesh](#16-centralized-observability--telemetry-mesh)

---

## 1. Classic 3-Tier Web Application

### Problem
A monolithic or decoupled web application requires strict network boundary isolation between presentation (web), business logic (app), and persistent state (database) while ensuring high availability.

### Architecture
```text
Internet ──> [Edge / Public Subnet: ALB / OCI LB]
                    │
                    ▼
         [Private Subnet: App Servers / Autoscaling Group]
                    │
                    ▼
         [Isolated Subnet: Primary DB + Standby Replica]
```

### Why It Works
Provides defense-in-depth: only load balancers face the public internet; application nodes reside in private subnets with no public IPs; databases are isolated in private subnets with zero internet routing.

### AWS Implementation
- Public subnets across 3 AZs containing Internet Gateway (IGW) and Application Load Balancer (ALB).
- Private subnets containing EC2 Auto Scaling Group (ASG) with NAT Gateway for outbound traffic.
- Isolated DB subnets hosting Amazon Aurora PostgreSQL with Multi-AZ cluster.

### OCI Implementation
- Public regional subnet hosting OCI Load Balancer (flexible shape) attached to an Internet Gateway.
- Private regional subnet hosting Compute Instance Pool with an Autoscaling configuration and NAT Gateway.
- Private DB subnet hosting OCI Base Database System or Autonomous DB with cross-AD/FD Data Guard.

### Trade-offs
- *Pros*: Clear security boundaries, decoupled scaling, simplified compliance auditing.
- *Cons*: Network hop latency between tiers; cross-AZ/AD data transfer costs.

### Failure Modes & Mitigation
- **NAT Gateway failure / bottleneck**: In AWS, deploy 1 NAT Gateway per AZ. In OCI, the native NAT Gateway is regional and automatically resilient.
- **Subnet IP exhaustion**: Size subnets with at least `/24` or `/23` blocks to prevent blocking autoscaling spikes.

### Interview Question & Defense
*Q: How do you prevent an attacker who compromises an app server from accessing other tenants' data in the database?*
*Defense*: Apply least-privilege database user credentials per microservice, enforce TLS with client certificates (mTLS), and implement Network Security Groups restricting egress strictly to port 5432/1521 of the database private IP.

---

## 2. Load-Balanced Microservices Tier

### Problem
Hundreds of concurrent containerized microservices need dynamic traffic distribution, health checking, path-based routing, and zero-downtime rolling upgrades.

### Architecture
```text
Client Request ──> [L7 Load Balancer]
                         │── /orders ──> [Orders Service Pods]
                         └── /users  ──> [Users Service Pods]
```

### AWS Implementation
- Amazon EKS cluster with AWS Load Balancer Controller provisioning an ALB that routes directly to Pod IPs via AWS VPC CNI.

### OCI Implementation
- OCI OKE cluster with OCI Cloud Controller Manager provisioning an OCI Load Balancer with path-based routing sets targeting worker nodes or Pods via VCN-native CNI.

### Trade-offs
- L7 routing introduces slightly higher latency than L4 pass-through but enables header inspection, path routing, and centralized TLS termination.

### Failure Modes
- **Health check flapping**: Misconfigured health check timeouts cause healthy pods under load to be marked dead, triggering a cascade. Mitigate with lenient check thresholds (e.g., 3 consecutive failures, 5s timeout).

---

## 3. Private Application Tier with Egress Filtering

### Problem
Internal backend compute must pull updates or call third-party APIs (e.g., Stripe, Twilio) without being directly addressable or reachable from the internet.

### AWS vs. OCI Implementation
- **AWS**: Route tables in private subnets direct `0.0.0.0/0` to a NAT Gateway located in a public subnet. Outbound traffic is filtered using AWS Network Firewall or Squid proxy.
- **OCI**: Private route tables direct `0.0.0.0/0` to an OCI NAT Gateway. To access Oracle internal services, route through an **OCI Service Gateway** (`all-services-in-region`) to eliminate NAT traversal.

### Trade-offs
- NAT Gateways charge per hour plus per-GB data processing fees. Routing internal cloud traffic to a Service Gateway or VPC Endpoint eliminates these fees.

---

## 4. Isolated Database Tier in Dedicated Subnets

### Problem
Databases must be protected from direct internet egress, preventing exfiltration even if the database software is compromised.

### AWS vs. OCI Implementation
- **AWS**: DB Subnet Group attached to a Route Table with **no default route** (`0.0.0.0/0`). Security Groups allow ingress on DB port strictly from the App Security Group ID.
- **OCI**: Private Subnet with no route rules to Internet Gateway or NAT Gateway. Security List / NSG allows ingress only from the Application Tier NSG.

### Trade-offs
- Secure against outbound exfiltration, but database patching requires downloading patches through private VPC endpoints or bastion proxies.

---

## 5. Event-Driven Architecture (EDA)

### Problem
Producers need to broadcast domain events to multiple downstream consumers asynchronously without coupling producer code to consumer availability.

### Architecture
```text
[Order Service] ──> [Event Router / Topic] ──┬──> [Inventory Service Queue]
                                             ├──> [Notification Service Queue]
                                             └──> [Analytics Stream]
```

### AWS Implementation
- Amazon EventBridge or SNS Topic with fan-out to multiple Amazon SQS Queues and Kinesis Data Streams.

### OCI Implementation
- OCI Events Service (capturing CloudEvents) and OCI Notifications (topic) fanning out to OCI Streaming (Kafka-compatible) and OCI Functions.

### Trade-offs
- *Pros*: Extreme decoupling, independent consumer scaling, fault containment.
- *Cons*: Eventual consistency, out-of-order event delivery risks, complex distributed tracing.

---

## 6. Serverless Web & API Architecture

### Problem
Unpredictable or spiky traffic patterns where provisioning continuous 24/7 VM infrastructure is cost-inefficient and operationally burdensome.

### AWS vs. OCI Implementation
- **AWS**: Amazon API Gateway (HTTP/REST) ──> AWS Lambda (ARM64 Graviton) ──> Amazon DynamoDB.
- **OCI**: OCI API Gateway ──> OCI Functions (Docker container runtime) ──> OCI NoSQL Database.

### Trade-offs
- Near-zero idle cost and automated scaling, but susceptible to cold starts and execution duration limits (AWS 15 min, OCI 5 min).

---

## 7. Asynchronous Queue-Worker Pattern

### Problem
Long-running background tasks (video transcoding, PDF generation, batch billing) must not block synchronous HTTP request-response cycles.

### AWS vs. OCI Implementation
- **AWS**: API sends message to Amazon SQS. Worker EC2 ASG / ECS tasks scale based on `ApproximateNumberOfMessagesVisible`. Failed messages move to a Dead-Letter Queue (DLQ).
- **OCI**: API writes to OCI Queue. Worker instance pool scales based on queue depth metrics via OCI Monitoring alarms. Poison messages route to DLQ.

### Trade-offs
- Provides backpressure buffering during load spikes. Requires idempotent worker processing because standard queues guarantee at-least-once delivery.

---

## 8. Distributed Cache-Aside Pattern

### Problem
Read-heavy relational databases become bottlenecked by redundant queries, increasing p99 latency.

### Architecture
```text
App ──1. Read Key──> [Cache (Redis)] ──Hit: Return Data──> App
 │                        │
 └──2. Miss: Query DB <───┘
 │
 └──3. Populate Cache with TTL
```

### AWS vs. OCI Implementation
- **AWS**: Amazon ElastiCache for Redis in Multi-AZ cluster mode.
- **OCI**: OCI Cache with Redis cluster in a private VCN subnet.

### Failure Modes & Mitigation
- **Cache Stampede (Thundering Herd)**: Occurs when a hot key expires and thousands of requests hit the DB simultaneously. Mitigate using mutual exclusion (mutex locks) or probabilistic early expiration.
- **Cache Penetration**: Queries for non-existent keys bypass cache. Mitigate by caching `null` values with short TTLs or using Bloom filters.

---

## 9. Edge CDN-Backed Static & Dynamic Acceleration

### Problem
Global users experience high round-trip latency when accessing centralized cloud regions.

### AWS vs. OCI Implementation
- **AWS**: CloudFront edge locations terminate TLS close to users, caching static S3 assets and proxying dynamic API calls via AWS global backbone to ALB.
- **OCI**: OCI Web Application Firewall (WAF) & OCI Edge Services integrated with global partner CDNs or OCI DNS Steering to optimize ingress paths.

### Trade-offs
- Substantially reduces origin load and bandwidth costs, but introduces cache invalidation challenges.

---

## 10. Blue/Green Immutable Deployments

### Problem
Deploying application updates to live servers risks downtime and mid-flight errors if an artifact is corrupted.

### Architecture
```text
           [Load Balancer]
                 │ (100% Traffic)
                 ▼
       [Blue Environment: v1.0 (Live)]

       [Green Environment: v2.0 (Staged / Validated)]
```

### AWS vs. OCI Implementation
- **AWS**: AWS CodeDeploy switches ALB Target Group traffic from Blue to Green target group once health checks pass.
- **OCI**: OCI DevOps deployment pipelines swap backend set routing in OCI Load Balancer from Blue instance pool to Green instance pool.

### Trade-offs
- Instant zero-downtime rollback by switching traffic back, but temporarily doubles compute infrastructure cost during deployment.

---

## 11. Canary Deployments with Automated Rollback

### Problem
Validating new releases against real production traffic without risking an outage for 100% of users.

### AWS vs. OCI Implementation
- **AWS**: Route 53 weighted records (e.g., 90% v1, 10% v2) or AWS ALB weighted target groups combined with CloudWatch Alarms triggering automated rollback.
- **OCI**: OCI Load Balancer backend set traffic weighting combined with OCI Monitoring alarms triggering automated pipeline rollback.

### Trade-offs
- Early detection of edge-case bugs and memory leaks with minimal user blast radius; requires backward-compatible database schemas.

---

## 12. Multi-Availability Zone Active-Active Tier

### Problem
Data center level hardware failure (power loss, flood, network cut) must not take down the application.

### AWS vs. OCI Implementation
- **AWS**: Workloads distributed equally across 3 AZs. ALB automatically routes away from degraded AZs. Aurora handles cross-AZ storage replication.
- **OCI**: Workloads distributed across all 3 Availability Domains (or 3 Fault Domains in single-AD regions). OCI Load Balancer health checks reroute traffic automatically.

### Trade-offs
- Guarantees high availability (99.99%), but requires cross-AZ network latency consideration (typically 1–2ms).

---

## 13. Multi-Region Disaster Recovery (Active-Passive / Warm Standby)

### Problem
Catastrophic failure of an entire cloud region or strict compliance requiring survival of regional disaster.

### Architecture
```text
[Global DNS: Route 53 / OCI DNS Steering]
      │
      ├──> Primary Region (Active: Serves 100% Traffic)
      │         │
      │    Async Data Replication
      │         ▼
      └──> Secondary Region (Standby: Scaled Down or Cold)
```

### AWS vs. OCI Implementation
- **AWS**: Primary in `us-east-1`, Secondary in `us-west-2`. Aurora Global Database (async replication < 1s). Route 53 health check flips DNS to secondary region if primary fails.
- **OCI**: Primary in Ashburn, Secondary in Phoenix. Autonomous DB / Base DB with cross-region OCI Data Guard. OCI DNS Traffic Management Failover steering policy.

### Trade-offs
- Balanced cost and recovery: RPO in seconds, RTO in minutes. Standby infrastructure runs at minimal footprint until failover.

---

## 14. Multi-Region Active-Active with Global Routing

### Problem
Zero-downtime tolerance and low latency requirements for worldwide users requiring writes in multiple continents simultaneously.

### AWS vs. OCI Implementation
- **AWS**: DynamoDB Global Tables (multi-master replication) + Route 53 Latency-Based Routing + Multi-Region ECS/EKS clusters.
- **OCI**: OCI DNS Geolocation Steering + Multi-region OKE clusters + distributed transactional data mesh or external CRDT-based storage.

### Trade-offs
- Near-zero RTO and global low latency, but high architectural complexity: requires conflict resolution logic (e.g., Last-Writer-Wins) and split-brain risk management.

---

## 15. Zero-Trust Cloud Network Architecture

### Problem
Perimeter defense alone is insufficient; internal network traffic must be treated as untrusted.

### Implementation
- **Identity-Driven Access**: Compute nodes communicate using short-lived tokens via AWS IAM Roles / OCI Instance Principals, not static network trust.
- **Microsegmentation**: Security Groups / NSGs applied per-workload, allowing only specific ports from authorized peer security groups.
- **End-to-End Encryption**: mTLS enforced across all pod-to-pod and service-to-service communication via service mesh (Istio / Linkerd).

---

## 16. Centralized Observability & Telemetry Mesh

### Problem
Scattered logs and metrics across hundreds of distributed cloud resources prevent rapid root-cause analysis during production incidents.

### Architecture
```text
Compute Nodes / Pods ──> [OTel Collector Agent]
                               │
               ┌───────────────┴───────────────┐
               ▼                               ▼
       [Metrics & Alarms]              [Structured Logs]
    (CloudWatch / OCI Monitor)     (CloudWatch Logs / OCI Logging)
```

### AWS vs. OCI Implementation
- **AWS**: OpenTelemetry Collector DaemonSet ──> Amazon CloudWatch Logs + Prometheus + AWS X-Ray.
- **OCI**: Unified Monitoring Agent ──> OCI Monitoring + OCI Logging + OCI Application Performance Monitoring (APM).

### Trade-offs
- Essential for operating at scale; telemetry ingestion and retention costs must be actively managed via log sampling and metric filtering.
