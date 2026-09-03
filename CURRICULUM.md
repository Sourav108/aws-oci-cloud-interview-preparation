# Comprehensive Cloud Engineering Interview Curriculum (AWS + OCI)

> **Target Audience**: *Experienced Backend Engineers, Systems Engineers, DevOps/SREs, and Platform Engineers preparing for SDE2, Senior, and Staff-level cloud architecture and infrastructure engineering interviews.*
>
> **Pedagogical Arc**: `UNDERSTAND → DESIGN → IMPLEMENT → DEPLOY → SECURE → OBSERVE → SCALE → DEBUG → OPTIMIZE → DEFEND TRADE-OFFS`

---

## 🏛️ Curriculum Philosophy & Standards

1. **Not a Certification Course**: This curriculum does not test memorization of vendor buzzwords. It teaches rigorous architectural reasoning from first principles:
   ```text
   Business Requirement → Architecture → Cloud Services → Networking → Security →
   Compute → Storage → Database → Messaging → Observability → Reliability →
   Scaling → Cost → Implementation → Production Operations
   ```
2. **Dual-Cloud Bilingualism**: Every domain is taught side-by-side on **AWS** and **Oracle Cloud Infrastructure (OCI)**, comparing structural semantics, network topology, pricing models, and failure modes.
3. **Calibrated Lesson Depth**:
   - **Major Topics** (e.g., VPC/VCN, EKS/OKE, IAM, HA/DR, Managed Databases): 20-section comprehensive breakdown (1,500 – 3,000 words).
   - **Supporting Topics** (e.g., specific storage classes, specialized gateways): Targeted 6–8 section analysis (300 – 800 words).

---

## 📚 Master 30-Module Syllabus

### Module 01: Cloud Foundations & Shared Responsibility
- Cloud computing service models: IaaS vs. PaaS vs. SaaS trade-offs.
- Public vs. Private vs. Hybrid Cloud architectures.
- The Cloud Shared Responsibility Model: AWS vs. OCI customer boundaries.
- Core cloud attributes: Elasticity, horizontal scalability, high availability, durability.
- On-premise infrastructure vs. Cloud-native migration economics and trade-offs.

### Module 02: Regions, AZs, ADs & Global Infrastructure
- AWS Global Infrastructure: Regions, Availability Zones (AZs), Local Zones, and Edge Locations.
- OCI Global Infrastructure: Regions, Availability Domains (ADs), and Fault Domains (FDs).
- Hierarchical topology: $\text{Region} \to \text{AZ / AD} \to \text{Fault Domain} \to \text{Compute Rack}$.
- Latency implications, blast-radius containment, data sovereignty, and compliance.

### Module 03: Networking Fundamentals & Packet Flow
- IPv4 / IPv6 addressing, Classless Inter-Domain Routing (CIDR) math.
- Layer 4 (TCP/UDP) vs. Layer 7 (HTTP/HTTPS/gRPC) networking.
- TCP 3-way handshake, TLS 1.3 handshake, MTU, packet fragmentation, socket buffers.
- DNS resolution packet lifecycle, NAT packet translation, proxying, and firewalls.

### Module 04: VPC & Virtual Cloud Network (VCN)
- AWS Virtual Private Cloud (VPC) vs. OCI Virtual Cloud Network (VCN).
- Subnet models: AWS zonal subnets vs. OCI regional subnets.
- Network firewalls: AWS Security Groups (ENI-level) vs. OCI Network Security Groups (NSGs) vs. OCI Security Lists (subnet-level).
- Network Access Control Lists (NACLs) stateless evaluation rules.

### Module 05: Subnets, Routing & Cloud Gateways
- End-to-end traffic flow: $\text{Client} \to \text{Internet Gateway} \to \text{Load Balancer} \to \text{App Subnet} \to \text{DB Subnet}$.
- Internet Gateways (IGW), NAT Gateways, and egress traffic filtering.
- Cloud backbone routing: AWS VPC Endpoints (Gateway vs. Interface) vs. OCI Service Gateway.
- Peering topologies: VPC Peering, AWS Transit Gateway (TGW), and OCI Dynamic Routing Gateway (DRG v2).

### Module 06: DNS, Route 53 & Service Discovery
- Domain Name System (DNS) fundamentals: Hosted zones, record types (A, AAAA, CNAME, ALIAS, TXT, SRV).
- AWS Route 53 routing policies: Simple, Weighted, Latency-based, Geolocation, Failover, Multi-value.
- OCI DNS and Traffic Management Steering Policies.
- Internal service discovery: AWS Cloud Map vs. Kubernetes CoreDNS vs. Consul.

### Module 07: Load Balancing & Traffic Ingress
- Layer 7 Application Load Balancing: AWS ALB vs. OCI Load Balancer (flexible shapes).
- Layer 4 Network Load Balancing: AWS NLB vs. OCI Network Load Balancer (pass-through).
- Reverse proxy mechanics: Path/host routing, TLS termination, HTTP/2 & gRPC support, sticky sessions, connection draining.
- Health check architectures: Liveness vs. Readiness checks, flapping dampening.

### Module 08: Compute & Virtual Machines
- AWS EC2: Nitro hypervisor architecture, instance families (compute, memory, storage, burstable), EBS optimization.
- OCI Compute: Bare Metal vs. Virtual Machines, Flexible Shapes (`VM.Standard.E5.Flex`), independent OCPU/RAM allocation.
- Lifecycle management: On-Demand, Spot / Preemptible, Reserved Instances, Savings Plans.
- Host placement strategies: Placement Groups (Cluster, Spread, Partition) vs. OCI Fault Domains.

### Module 09: Containers & Serverless Container Runtimes
- Container runtime mechanics: Linux namespaces, cgroups, container security posture.
- Container registries: Amazon ECR vs. OCI Container Registry (OCIR), vulnerability scanning.
- Serverless container execution: AWS ECS / Fargate vs. OCI Container Instances.
- When to choose containers over virtual machines; container networking and logging.

### Module 10: Kubernetes (Cloud-Managed Lens: EKS vs. OKE)
- Cloud-managed Kubernetes control plane: Amazon EKS vs. OCI Container Engine for Kubernetes (OKE).
- Worker node management: EKS Managed Node Groups / Karpenter vs. OKE Node Pools.
- Cloud networking integration: AWS VPC CNI vs. OCI VCN-Native Pod Networking.
- Cloud IAM integration: EKS Pod Identity / IRSA vs. OKE Workload Identity.
- Cloud load balancer integration, ingress controllers, and storage provisioners (CSI).

### Module 11: Serverless Functions & Event-Driven Compute
- Event-driven compute runtimes: AWS Lambda vs. OCI Functions (powered by Fn Project).
- Execution lifecycle: Cold start mitigation, execution limits, memory/CPU scaling, concurrency controls.
- API Gateways: Amazon API Gateway vs. OCI API Gateway (routing, rate limiting, authentication).
- Serverless architectural patterns: Webhook ingestion, event filtering, async stream processors.

### Module 12: Object Storage
- Cloud object storage architecture: Amazon S3 vs. OCI Object Storage.
- Storage tiers: Standard, Infrequent Access, Archive, Deep Archive, Auto-Tiering.
- Data protection: Strong consistency models, bucket policies, object locking, pre-signed URLs, cross-region replication.
- High-throughput optimization: Partition prefixes, multipart uploads, byte-range reads.

### Module 13: Block & File Storage
- Persistent VM block storage: Amazon EBS (`gp3`, `io2`) vs. OCI Block Volume.
- Performance models: IOPS, throughput, latency, OCI Volume Performance Units (VPUs).
- Shared network filesystems: Amazon EFS vs. OCI File Storage Service (FSS) (NFSv3/NFSv4).
- Snapshots, point-in-time recovery, volume cloning, and disaster recovery replication.

### Module 14: Managed Relational Databases
- Cloud relational database architecture: Amazon RDS & Amazon Aurora vs. OCI Base Database System & Autonomous Database.
- High availability: Multi-AZ synchronous replication vs. OCI Active Data Guard.
- Read scaling: Asynchronous read replicas, replication lag dynamics, read-write splitting.
- Maintenance, automated backups, point-in-time recovery (PITR), and connection pooling (RDS Proxy / PgBouncer).

### Module 15: NoSQL & Distributed Data Systems
- Distributed key-value and document databases: Amazon DynamoDB vs. OCI NoSQL Database.
- Partitioning mechanics: Partition keys, sort keys, partition splitting, hot partition mitigation.
- Consistency models: Strongly consistent reads vs. Eventually consistent reads.
- Global tables: Multi-region active-active replication, conflict resolution strategies.

### Module 16: In-Memory Caching & Distributed Caches
- In-memory data acceleration: Amazon ElastiCache (Redis) vs. OCI Cache with Redis.
- Caching strategies: Cache-aside, write-through, write-behind, refresh-ahead.
- Cache failure modes: Cache stampede (thundering herd), cache penetration, cache breakdown.
- Redis clustering: Master-replica failover, shard partitioning, memory eviction policies.

### Module 17: Messaging & Eventing Architectures
- Decoupled message queues: Amazon SQS vs. OCI Queue (at-least-once delivery, visibility timeouts, DLQs).
- Publish/subscribe topic broadcasts: Amazon SNS vs. OCI Notifications.
- Event buses & event streaming: Amazon EventBridge & Kinesis vs. OCI Events & OCI Streaming (Kafka-compatible).
- Message ordering, idempotency, deduplication, and backpressure management.

### Module 18: IAM, Cloud Security & Workload Identity
- Identity architecture: AWS IAM (Users, Groups, Roles, Policies, STS) vs. OCI IAM (Compartments, Groups, Dynamic Groups, Policies).
- Policy evaluation logic: AWS explicit allow/deny evaluation vs. OCI compartment policy inheritance.
- Workload identity: IAM Roles for EC2/EKS vs. OCI Dynamic Groups & Instance Principals.
- Centralized governance: AWS Organizations & SCPs vs. OCI Security Zones.

### Module 19: Secrets, Encryption & Key Management
- Cryptographic key management: AWS Key Management Service (KMS) vs. OCI Vault (KMS).
- Hardware security modules (HSM), customer-managed keys (CMK), envelope encryption mechanics.
- Secret lifecycle management: AWS Secrets Manager vs. OCI Secrets in Vault.
- Automated secret rotation, access auditing, and application integration.

### Module 20: Observability, Monitoring & Logging
- The Three Pillars of Observability: Metrics, Structured Logs, Distributed Traces.
- AWS Observability: Amazon CloudWatch (Metrics, Logs, Alarms), Container Insights, AWS X-Ray.
- OCI Observability: OCI Monitoring, OCI Logging, Service Connector Hub, OCI Application Performance Monitoring (APM).
- SLO/SLI tracking, error budget multi-burn rate alerting, and OpenTelemetry integration.

### Module 21: Autoscaling, Capacity Planning & Throttling
- Elastic scaling mechanisms: AWS Auto Scaling Groups vs. OCI Instance Pools.
- Scaling policies: Target tracking, step scaling, scheduled scaling, queue-depth scaling.
- Throttling mechanics: Token bucket algorithms, leaky bucket algorithms, API rate limiting.
- System bottleneck identification across compute, memory, database connection, and network boundaries.

### Module 22: High Availability & Disaster Recovery (HA/DR)
- Disaster recovery planning: Recovery Point Objective (RPO) and Recovery Time Objective (RTO).
- DR architecture tiers: Backup & Restore, Pilot Light, Warm Standby, Multi-Region Active-Active.
- Cross-region replication: S3 Cross-Region Replication, Aurora Global Database, OCI Remote Peering & Data Guard.
- Region failure game days, automated failover steering, and failback procedures.

### Module 23: Resilience, Fault Tolerance & Chaos Engineering
- Fault containment patterns: Circuit breakers, bulkheads, rate limiters, fallback responses.
- Resilient client design: Exponential backoff with full jitter, retry budgets, deadline propagation.
- Chaos engineering: AWS Fault Injection Simulator (FIS) and Chaos Mesh.
- Designing for partial network partitions and grey failures.

### Module 24: Cloud Migrations & Modernization
- The 5/6 R's Migration Framework: Rehost, Replatform, Refactor, Repurchase, Retire, Retain.
- Migration tooling: AWS Application Migration Service (MGN), Database Migration Service (DMS) vs. OCI Database Migration.
- Cutover strategies: Big bang vs. Phased strangler-fig migration, dual-writing, and rollback planning.

### Module 25: Infrastructure as Code (Terraform)
- Modern IaC workflows: Providers, resources, data sources, variables, outputs, and local values.
- State management: Remote backends (S3 + DynamoDB vs. OCI Object Storage), state locking, workspace separation.
- Modularization: Reusable, composable Terraform modules; blast-radius containment.
- Drift detection, `terraform plan -detailed-exitcode`, and import strategies.

### Module 26: CI/CD & Production Deployment Strategies
- Deployment strategies: Rolling update, Blue/Green, Canary, and Feature Flags.
- AWS Developer Tools: CodePipeline, CodeBuild, CodeDeploy vs. OCI DevOps service.
- Vendor-neutral GitOps pipelines: GitHub Actions with OIDC cloud authentication.
- Automated testing gates, deployment health metrics, and automated rollback hooks.

### Module 27: Cloud Performance Optimization
- Latency profiling and bottleneck elimination across the network, compute, database, and cache.
- Connection pooling optimization: Thread models, socket re-use, HTTP/2 multiplexing.
- Database query tuning: Indexing strategies, execution plan analysis, read replica offloading.
- Measuring performance using p50, p95, and p99 percentiles.

### Module 28: Cloud Cost Optimization & FinOps
- Cloud unit economics: Cost per request, cost per active tenant.
- Compute optimization: Right-sizing, Graviton/Ampere ARM adoption, Spot/Preemptible instance pools.
- Commitment discounts: AWS Savings Plans / Reserved Instances vs. OCI Universal Credits & Reserved Capacity.
- Network data transfer cost reduction: NAT elimination, VPC endpoints, OCI free egress allowances.
- FinOps cultural practices and automated budget alerts.

### Module 29: Cloud Engineering Interview Questions (500 Questions)
- Sub-phased 500-question comprehensive bank (format: Question, Short Answer, Deep Answer, Architecture, AWS, OCI, Trap, Follow-up).
- Sub-phase 29.1: Fundamentals, AWS Core, OCI Core, Networking, Compute (125 Qs).
- Sub-phase 29.2: Storage, Databases, Caching, Messaging, Kubernetes (125 Qs / Total: 250).
- Sub-phase 29.3: Serverless, IAM, Security, Observability, Autoscaling (125 Qs / Total: 375).
- Sub-phase 29.4: HA, DR, Resilience, Terraform, CI/CD (100 Qs / Total: 475).
- Sub-phase 29.5: Performance, Cost, Troubleshooting, System Design (25 Qs / Total: 500).

### Module 30: Cloud System Design & Reference Architectures
- Complete dual-cloud reference designs for senior/staff interview scenarios:
  1. High-Availability Tier-1 REST API.
  2. Asynchronous Event-Driven Order Processing Platform.
  3. Globally Distributed Low-Latency Media Upload Platform.
  4. Multi-Region Active-Passive Disaster Recovery System.
  5. High-Throughput Real-Time Financial Transaction Engine.
- Deep architectural dissection: Requirements, Traffic Flow, Storage, Security, Scaling, Reliability, Observability, Cost, Failure Modes, and Trade-off Defense.
