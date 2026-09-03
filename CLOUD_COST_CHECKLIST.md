# Cloud Cost Optimization, Unit Economics & FinOps Checklist

> **The FinOps Engineering Mandate**: *Cost is not a finance afterthought; it is an architectural constraint. Every architectural decision incurs continuous recurring capital. In Senior and Staff interviews, you must quantify cost implications and analyze trade-offs.*
>
> **The Non-Negotiable Rule**: *Never recommend a cost optimization that silently weakens security perimeters or drops multi-AZ reliability guarantees without an explicit architectural defense.*

---

## 🏷️ Grounding & Pricing Baseline Notice
All estimated costs in this document carry explicit grounding tags per repository policy:
- `[Doc: <service>, checked <date>]`
- `[Approximation — verify before relying on this]`
Actual pricing varies across AWS and OCI geographical regions and commitment contracts.

---

## 💰 The 8 Cloud Cost Dimensions

### 1. Compute Right-Sizing & Flexible Provisioning
- [ ] **Continuous Metric Right-Sizing**:
  - Analyze CPU and memory utilization percentiles ($p95$). If an instance fleet consistently operates below 20% CPU over 14 days, downsize the instance family.
  - AWS: Use **AWS Compute Optimizer** recommendations.
  - OCI: Use **Flexible Shapes** (`VM.Standard.E5.Flex`) to dial in exact OCPU and RAM specifications (e.g., allocate 2 OCPUs and 12 GB RAM rather than jumping to a fixed rigid shape).
- [ ] **ARM Architecture Adoption**:
  - Migrate stateless microservices, Go, Java, and Python workloads to ARM64 processors (AWS Graviton3 / OCI Ampere A1).
  - *Financial Benefit*: Delivers up to 40% better price-performance compared to comparable x86 architectures `[Approximation — verify before relying on this]`.
- [ ] **Shutdown Idle Non-Production Environments**:
  - Schedule automated shutdown of dev/QA/staging VM fleets and Kubernetes worker nodes outside business hours (7 PM to 7 AM + weekends).
  - *Financial Impact*: Eliminates ~65% of non-production compute spend `[Inference]`.

### 2. Capacity Commitment Models (Baseline vs. Dynamic)
- [ ] **Establish Baseline vs. Peak Ratios**:
  - Baseline load (running 24/7/365) must be committed under discounted billing constructs.
  - Variable/spiky traffic must remain on on-demand or autoscaling groups.
- [ ] **AWS Commitment Models**:
  - *Compute Savings Plans*: Provide maximum flexibility across EC2, Fargate, and Lambda with discounts up to ~66% for 3-year commitments `[Approximation — verify before relying on this]`.
  - *Standard Reserved Instances*: Offer highest discount for static RDS/Aurora database instances.
- [ ] **OCI Commitment Models**:
  - *OCI Universal Credits (Annual Commit)*: Provides tiered volume discounts across all OCI services with monthly drawdown.
  - *Reserved Capacity*: Guarantees compute capacity in designated Availability Domains without upfront instance lock-in fees.

### 3. Spot & Preemptible Compute for Fault-Tolerant Workloads
- [ ] **Spot / Preemptible Adoption**:
  - Run stateless worker nodes, batch queues, ETL pipelines, and CI/CD runners on AWS Spot Instances / OCI Preemptible Instances.
  - *Discount*: Up to 70–90% savings compared to on-demand pricing `[Approximation — verify before relying on this]`.
- [ ] **Interruption Handling**:
  - Handle termination warnings (AWS 2-minute warning via EventBridge; OCI 2-minute preemption notification via metadata service).
  - Drain node gracefully, checkpoint progress to Object Storage, and requeue in-flight tasks.
- [ ] *Trade-off Notice*: **Never run stateful database primaries or single-node mission-critical APIs on Spot/Preemptible instances.**

### 4. Network Data Transfer & Egress Optimization
- [ ] **Eliminate Unnecessary NAT Gateway Egress**:
  - NAT Gateways charge both an hourly rate (~$0.045/hr) plus data processing fees (~$0.045/GB) in AWS `[Approximation — verify before relying on this]`.
  - Deploy **VPC Gateway Endpoints** for S3 and DynamoDB (100% free) to eliminate NAT traversal for storage APIs.
  - In OCI, route internal traffic through the **OCI Service Gateway** (free data transfer to all Oracle services in the region).
- [ ] **Cross-AZ & Inter-Region Traffic Management**:
  - Data transfer within the same AZ is generally free. Cross-AZ traffic incurs inter-zone charges (~$0.01/GB in/out in AWS) `[Approximation — verify before relying on this]`.
  - Keep chatty microservice RPCs inside the same AZ where feasible using AZ-aware routing, while maintaining multi-AZ resilience for the overall service.
- [ ] **Internet Egress Cost Differences**:
  - OCI provides 10 TB of internet data egress per month free across all tenancies, with subsequent egress priced significantly lower than AWS `[Doc: OCI Cloud Economics, checked 2026-09-03]`. AWS provides 100 GB/month free egress `[Doc: AWS Data Transfer Pricing, checked 2026-09-03]`.

### 5. Storage Tiering & Automated Lifecycle Management
- [ ] **Object Storage Lifecycle Policies**:
  - AWS S3: Configure S3 Lifecycle rules transitioning objects from S3 Standard to S3 Infrequent Access (IA) after 30 days, Glacier Flexible Retrieval after 90 days, and deletion/Glacier Deep Archive after 365 days.
  - OCI Object Storage: Enable **Auto-Tiering** on buckets, automatically shifting objects between Standard and Infrequent Access based on access patterns without transition penalties.
- [ ] **Orphaned EBS & Block Volume Cleanup**:
  - Audit and delete unattached EBS volumes (`state == available`) left behind after instance termination.
  - Enforce automated snapshot cleanup after 30 days using AWS Data Lifecycle Manager (DLM) / OCI Scheduled Backups.
- [ ] **Dynamic Performance Tuning in OCI**:
  - Dial down OCI Block Volume Volume Performance Units (VPUs) from High Performance (20 VPUs) to Lower Cost (0 VPUs) during off-peak hours for batch processing volumes.

### 6. Managed Database Cost Engineering
- [ ] **Connection Pooling to Prevent Over-Sizing**:
  - Provisioning a larger DB instance simply to handle idle client connections is extremely wasteful.
  - Deploy **AWS RDS Proxy** or **PgBouncer** to pool database connections, allowing a smaller database instance to handle thousands of client connections.
- [ ] **Aurora Serverless v2 vs. Provisioned Clusters**:
  - Use Aurora Serverless v2 for workloads with unpredictable or intermittent traffic, scaling in fine-grained 0.5 ACU (Aurora Capacity Unit) increments.
  - Use provisioned Multi-AZ instances with Reserved Instances for predictable, high-concurrency baseline workloads.
- [ ] **Autonomous Database Auto-scaling in OCI**:
  - Enable auto-scaling on OCI Autonomous Database to allow compute to expand up to 3x base OCPU during peak queries, scaling down instantly when queries subside.

### 7. Observability, Telemetry & Logging Cost Control
- [ ] **Log Retention & Ingestion Filtering**:
  - Default log retention in CloudWatch Logs is "Never Expire". Set explicit retention limits (e.g., 7 days for dev, 30 days for production) on all log groups.
  - Filter noisy HTTP 200 health check logs at the agent level (OTel / Fluentd) to avoid paying ingestion fees for useless data.
- [ ] **Metric Cardinality Management**:
  - Never include user IDs, transaction UUIDs, or timestamps as metric dimensions in CloudWatch or OCI Monitoring.
  - Keep high-cardinality metadata in structured logs or distributed traces.

### 8. Architectural Cost Defense in Interviews
When asked to optimize cost in a system design interview, structure your answer using the following defense hierarchy:

1. **Quick-Win Waste Elimination**: Purge unattached block storage, terminate abandoned test VMs, set S3/Object Storage lifecycle rules.
2. **Architectural Decoupling**: Introduce caching (Redis) to reduce database read replicas; introduce queues (SQS/OCI Queue) to smooth traffic spikes and downsize compute fleets.
3. **Purchasing Optimization**: Apply Compute Savings Plans and Reserved Instances to known steady-state baseline compute and databases.
4. **Provider-Specific Advantages**: Highlight OCI Flexible Shapes (no paying for unused RAM) and OCI's aggressive data transfer free tiers compared to AWS.
5. **State Trade-offs Explicitly**: *"We could cut costs by 50% by running a single-AZ database, but we accept the higher cost of Multi-AZ because business requirements mandate 99.99% availability."*
