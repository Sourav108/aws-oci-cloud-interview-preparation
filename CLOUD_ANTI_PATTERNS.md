# Cloud Architectural Anti-Patterns Catalog

This document catalogues 15 critical cloud architecture, infrastructure, and operational anti-patterns frequently tested in Senior and Staff Cloud Engineering interviews.

Every anti-pattern is structured to analyze:
1. **Why It Happens**
2. **Real-World Production Risk**
3. **Symptoms & Observability Signals**
4. **Detection & Audit Mechanisms**
5. **Better Architectural Design (AWS & OCI)**
6. **When It May Be Acceptable**

---

## 📑 Anti-Patterns Index

1. [The "Everything Public" Subnet Strategy](#1-the-everything-public-subnet-strategy)
2. [Public Database Endpoints](#2-public-database-endpoints)
3. [Wildcard IAM Policies (`Action: "*"`)](#3-wildcard-iam-policies-action-)
4. [Hardcoded Credentials & Long-Lived API Keys](#4-hardcoded-credentials--long-lived-api-keys)
5. [Single-AZ / Single-AD Deployment for Production Workloads](#5-single-az--single-ad-deployment-for-production-workloads)
6. [Missing Health Checks & Naive Ping Checks](#6-missing-health-checks--naive-ping-checks)
7. [Unmitigated Retry Storms Without Jitter](#7-unmitigated-retry-storms-without-jitter)
8. [Missing Socket, Connect, and Request Timeouts](#8-missing-socket-connect-and-request-timeouts)
9. [Static Overprovisioning Instead of Autoscaling](#9-static-overprovisioning-instead of-autoscaling)
10. [Unnecessary Cross-AZ & Unrouted Internal Traffic Traversal](#10-unnecessary-cross-az--unrouted-internal-traffic-traversal)
11. [Unmanaged Terraform State Drift & Manual Console Tweaks](#11-unmanaged-terraform-state-drift--manual-console-tweaks)
12. [Monolithic "Giant" Terraform Root Modules](#12-monolithic-giant-terraform-root-modules)
13. [Untested Backups & Theoretical Disaster Recovery Plans](#13-untested-backups--theoretical-disaster-recovery-plans)
14. [The "Managed Service Means Zero Operational Responsibility" Fallacy](#14-the-managed-service-means-zero-operational-responsibility-fallacy)
15. [Unbounded Telemetry & High-Cardinality Metric Ingestion](#15-unbounded-telemetry--high-cardinality-metric-ingestion)

---

## 1. The "Everything Public" Subnet Strategy

### Why It Happens
Teams deploy all EC2 instances, containers, or VMs into public subnets with public IP addresses attached to avoid setting up NAT Gateways or bastion hosts.

### Production Risk
Direct exposure of application ports to automated internet scanners, brute-force SSH attacks, and zero-day vulnerabilities in host operating systems.

### Symptoms & Observability Signals
- AWS GuardDuty or OCI Cloud Guard alerting on unauthorized port scanning or unexpected inbound SSH/RDP attempts from foreign IPs.
- VPC Flow Logs / VCN Flow Logs showing traffic from unknown internet IP ranges hitting non-web ports.

### Detection & Audit
- AWS Config rule `ec2-instance-no-public-ip`.
- OCI Cloud Guard detector `INSTANCE_HAS_PUBLIC_IP`.

### Better Design
Deploy workloads in private subnets with no public IPs. Ingress is permitted only via an Application Load Balancer or OCI Load Balancer. Egress is routed through a regional NAT Gateway or Service Gateway.

### When It May Be Acceptable
Ephemeral developer proof-of-concept sandboxes with automatic teardown after 24 hours.

---

## 2. Public Database Endpoints

### Why It Happens
Developers want to connect directly to RDS, Aurora, or OCI Base Database from their local laptops via DBeaver or PgAdmin without configuring VPNs or SSH tunnels.

### Production Risk
Catastrophic data breach, credential brute-forcing, and DDoS attacks on database connection listeners.

### Symptoms
- Massive spikes in connection authorization failures in database audit logs.
- Unexpected outbound data transfer spikes indicating data exfiltration.

### Detection & Audit
- AWS Config rule `rds-instance-public-access-check`.
- OCI Security Zone policy preventing public IP assignment to database systems.

### Better Design
Place databases in isolated private subnets with no route to an Internet Gateway. Administrative access is achieved exclusively via AWS Systems Manager Session Manager / OCI Bastion Service with IAM-authenticated ephemeral port forwarding.

### When It May Be Acceptable
Never in production or staging environments containing real or sanitized data.

---

## 3. Wildcard IAM Policies (`Action: "*"`)

### Why It Happens
Engineers encounter `AccessDeniedException` during development and replace fine-grained permissions with `Resource: "*"` and `Action: "*"` to quickly unblock testing.

### Production Risk
Privilege escalation, accidental resource destruction, and massive blast radius upon credential compromise.

### Symptoms
- AWS CloudTrail / OCI Audit showing identity entities executing administrative APIs far outside their functional scope.

### Detection & Audit
- AWS IAM Access Analyzer identifying overly permissive policies.
- OCI Identity policy review flagging `manage all-resources in tenancy`.

### Better Design
Implement least privilege: grant specific API actions (`s3:GetObject`, `s3:PutObject`) scoped strictly to explicit resource ARNs/OCIDs. In OCI, use restrictive verbs (`read` or `use` instead of `manage`) scoped to child compartments.

### When It May Be Acceptable
Bootstrap tenancy admin identity used solely for setting up organizational guardrails.

---

## 4. Hardcoded Credentials & Long-Lived API Keys

### Why It Happens
Developers embed AWS Access Keys (`AKIA...`) or OCI API signing keys directly into application configuration files, environment variables, or Git repositories.

### Production Risk
Credentials committed to version control, leaked through build artifacts, or scraped by botnets leading to unauthorized crypto-mining and data theft.

### Symptoms
- CloudTrail / OCI Audit showing API calls originating from anomalous residential or VPS IP addresses.

### Detection & Audit
- Secret scanning tools (git-secrets, TruffleHog) in CI/CD.
- AWS Secrets Manager and OCI Vault automatic secret rotation checks.

### Better Design
Adopt **Workload Identity**:
- AWS: Attach an IAM Role to EC2 instance profile or use EKS Pod Identity (IRSA).
- OCI: Add compute instances to a Dynamic Group and write IAM policies granting the dynamic group access to needed services. SDKs authenticate automatically via metadata service tokens.

### When It May Be Acceptable
Legacy external third-party tools that do not support OpenID Connect (OIDC) federation, accompanied by 30-day automated secret rotation.

---

## 5. Single-AZ / Single-AD Deployment for Production Workloads

### Why It Happens
Cost optimization taken to an extreme; avoiding cross-AZ data transfer fees or multiple VM provisioning costs.

### Production Risk
Zero tolerance for data center outages. A physical fiber cut, cooling failure, or power disruption in one AZ causes a total application outage.

### Symptoms
- Total service degradation when a single cloud availability zone reports impairment on the cloud service health dashboard.

### Detection & Audit
- Architectural review verifying that ASGs, Target Groups, and RDS clusters span at least 2 (preferably 3) Availability Zones or Fault Domains.

### Better Design
Multi-AZ active-active deployment:
- AWS: Auto Scaling Groups distributed across 3 AZs; Multi-AZ RDS / Aurora.
- OCI: Instance Pools distributed across all 3 Availability Domains (or 3 Fault Domains in single-AD regions).

### When It May Be Acceptable
Non-critical development, QA, or staging environments where downtime during off-hours is acceptable.

---

## 6. Missing Health Checks & Naive Ping Checks

### Why It Happens
Teams configure the load balancer health check to ping a static HTTP `/health` endpoint that unconditionally returns `200 OK` without validating database or backend connectivity.

### Production Risk
Load balancer continues forwarding client requests to application nodes whose internal connection pools are exhausted or whose background threads have deadlocked.

### Symptoms
- Load balancer reports 100% healthy backend targets, while client error rates (HTTP 500 / 504) soar to 50%+.

### Detection & Audit
- Correlate load balancer target health status with downstream error response rates during load tests.

### Better Design
Differentiate **Shallow vs. Deep Health Checks**:
- *Liveness check*: Simple ping verifying process is running and event loop is not blocked.
- *Readiness check*: Validates downstream dependencies (database ping, cache ping). If a dependency fails temporarily, shed traffic gracefully without crashing the process.

### When It May Be Acceptable
Never. Every production service must have calibrated liveness and readiness probes.

---

## 7. Unmitigated Retry Storms Without Jitter

### Why It Happens
When a downstream microservice degrades, client applications retry failed requests immediately and continuously in tight loops.

### Production Risk
The downstream service, attempting to recover from a minor hiccup, is overwhelmed by a wave of exponential retries and crashes completely (thundering herd).

### Symptoms
- Downstream service CPU spikes to 100%; incoming request rates jump 10x–50x above normal traffic; p99 latency flatlines at the timeout limit.

### Detection & Audit
- Inspect client-side retry configurations in SDKs and HTTP clients.

### Better Design
Implement **Exponential Backoff with Full Jitter** and **Circuit Breakers**:
$$t_{sleep} = \text{random}(0, \min(t_{max}, t_{base} \cdot 2^{attempt}))$$
Trip circuit breakers when error rates exceed 50% over a 10-second rolling window.

### When It May Be Acceptable
Idempotent local batch file processing with a maximum retry count of 2.

---

## 8. Missing Socket, Connect, and Request Timeouts

### Why It Happens
Engineers rely on default HTTP client configurations, which in many languages (e.g., standard Go `http.Client`, Python `requests`, Java `HttpURLConnection`) have **infinite** connect or read timeouts.

### Production Risk
A single hanging downstream dependency causes upstream connection pools and worker threads to block indefinitely, exhausting memory and crashing the entire caller fleet.

### Symptoms
- Thread pool exhaustion (`java.lang.OutOfMemoryError: unable to create new native thread`); socket connection counts reaching OS limits (`ulimit`).

### Detection & Audit
- Static code analysis flagging unconfigured HTTP client timeouts.

### Better Design
Explicitly configure 3 distinct timeout thresholds on every client:
1. *Connect Timeout*: 500ms – 1s.
2. *Socket / Read Timeout*: Calibrated to service p99 SLA + buffer (e.g., 2s).
3. *Overall Request Deadline*: Propagated via context headers (e.g., gRPC deadlines).

### When It May Be Acceptable
Never. Defaults must always be overridden.

---

## 9. Static Overprovisioning Instead of Autoscaling

### Why It Happens
Engineers size VM fleets for peak seasonal traffic (e.g., Black Friday) and leave them running 24/7/365 to avoid dealing with autoscaling lag or configuration.

### Production Risk
Enormous cloud bill waste; 70–80% of compute capacity sits idle during off-peak hours.

### Symptoms
- Average fleet CPU utilization hovering below 10–15% in CloudWatch / OCI Monitoring.

### Detection & Audit
- AWS Compute Optimizer / OCI Cloud Advisor flagging over-provisioned compute resources.

### Better Design
Dynamic Autoscaling:
- Base layer: Reserved Instances / Savings Plans sized for minimum baseline load.
- Dynamic layer: Auto Scaling Group / Instance Pool driven by target tracking (e.g., maintain 60% CPU) or queue depth.
- Spike handling: Scheduled scaling for anticipated events or containerized microservices on Kubernetes (HPA).

### When It May Be Acceptable
Ultra-low-latency real-time trading engines where VM warm-up and scaling latencies cannot be tolerated.

---

## 10. Unnecessary Cross-AZ & Unrouted Internal Traffic Traversal

### Why It Happens
Services in AZ-a communicate with databases in AZ-b without AZ-affinity; internal S3 or Object Storage traffic is routed out to the public internet via NAT Gateways.

### Production Risk
Unnecessary network latency added to every RPC call (1–2ms per hop) and massive cross-AZ data transfer fees on monthly cloud bills.

### Symptoms
- High line-item charges for `NatGateway-Bytes` and `InterZone-DataTransfer-Out` on cloud billing reports.

### Detection & Audit
- Analyze AWS Cost Explorer / OCI Cost Analysis by Usage Type.

### Better Design
- Keep latency-sensitive inter-service communication within the same AZ using AZ-aware routing.
- Provision **VPC Endpoints (Gateway for S3/DynamoDB)** in AWS and **Service Gateways** in OCI to route internal cloud API traffic over the provider backbone with zero NAT or egress charges.

### When It May Be Acceptable
Strict active-active load balancing where equal traffic distribution across AZs outweighs marginal data transfer costs.

---

## 11. Unmanaged Terraform State Drift & Manual Console Tweaks

### Why It Happens
Engineers log into the AWS or OCI web console during a production incident to manually change a security group rule or VM size, never committing the change back to IaC.

### Production Risk
Next automated CI/CD Terraform pipeline run either fails, or worse, silently destroys/reverts the emergency hotfix, causing a regression outage.

### Symptoms
- `terraform plan` reports hundreds of unexpected changes, replaces, or conflicts with remote state.

### Detection & Audit
- Run scheduled automated `terraform plan -detailed-exitcode` in CI to detect drift daily.

### Better Design
Enforce **Immutable Infrastructure & GitOps**:
- Revoke manual write permissions to production consoles; all production changes must pass through Terraform CI/CD pipelines.
- If emergency hotfixes are necessary, an incident ticket must require a corresponding Terraform commit within 24 hours.

### When It May Be Acceptable
P0 emergency triage when the CI/CD pipeline itself is offline, followed immediately by state import/reconciliation.

---

## 12. Monolithic "Giant" Terraform Root Modules

### Why It Happens
A single repository houses the entire corporate infrastructure—VPCs, databases, Kubernetes clusters, IAM roles, DNS records—in one massive Terraform state file.

### Production Risk
- Blast radius is catastrophic: a typo or state lock failure can destroy the entire corporate environment.
- `terraform plan` takes 30+ minutes due to thousands of cloud API read calls.
- High risk of state locking contention across engineering teams.

### Symptoms
- `terraform refresh` times out; team members blocked waiting for state locks.

### Detection & Audit
- Repository inspection showing thousands of lines in a single `main.tf` or state files exceeding 10MB.

### Better Design
Decompose infrastructure into layered, independently deployed state boundaries:
1. *Foundation Layer*: VPC/VCN, Subnets, Gateways, Core Routing.
2. *Platform Layer*: Kubernetes clusters, Load Balancers, Shared Registries.
3. *Data Layer*: Databases, Queues, Caches.
4. *Application Layer*: Microservices, IAM roles, DNS records.

### When It May Be Acceptable
Single-purpose, small disposable sandbox environments.

---

## 13. Untested Backups & Theoretical Disaster Recovery Plans

### Why It Happens
Organizations configure automated database snapshots or storage replication and assume disaster recovery is solved, without ever running a live recovery rehearsal.

### Production Risk
During a real ransomware incident or region failure, backups are found to be corrupt, encryption keys are missing from the recovery region, or restoration takes 36 hours instead of the target 1-hour RTO.

### Symptoms
- No documentation or automated tests verifying restoration validity in the past 90 days.

### Detection & Audit
- Check AWS Backup / OCI Backup compliance reports; demand recovery test audit evidence.

### Better Design
**Continuous Automated Recovery Testing (CART)**:
- Schedule an automated weekly pipeline that restores the latest snapshot into an isolated sandbox, spins up a temporary database instance, executes a synthetic data validation query, and tears the instance down.
- Maintain a proven, documented RPO and RTO backed by real drill metrics.

### When It May Be Acceptable
Never for tier-1 or tier-2 production systems.

---

## 14. The "Managed Service Means Zero Operational Responsibility" Fallacy

### Why It Happens
Teams assume that using Amazon RDS, DynamoDB, or OCI Autonomous Database means the cloud provider handles all scaling, connection pooling, indexing, and query optimization automatically.

### Production Risk
Sudden database freeze due to storage auto-growth limits, connection pool exhaustion during traffic surges, or runaway costs from unindexed full-table scans.

### Symptoms
- High database CPU utilization; `TooManyConnectionsException`; throttled read/write capacity units.

### Detection & Audit
- Database performance insights metrics showing unoptimized queries and connection spikes.

### Better Design
Adhere strictly to the **Shared Responsibility Model for Managed Services**:
- Cloud provider manages hardware, hypervisor, OS patching, and storage replication.
- Customer remains 100% responsible for query optimization, schema indexing, connection management (PgBouncer / RDS Proxy), capacity sizing, and application-level retry policies.

### When It May Be Acceptable
Never. Managed services eliminate undifferentiated heavy lifting, not engineering discipline.

---

## 15. Unbounded Telemetry & High-Cardinality Metric Ingestion

### Why It Happens
Developers emit custom metrics containing high-cardinality dimensions (e.g., User ID, UUID, Order ID) into CloudWatch Metrics or Datadog.

### Production Risk
Exponential explosion in custom metric charges, often resulting in cloud observability bills exceeding the cost of the underlying compute infrastructure.

### Symptoms
- CloudWatch custom metric bill surges by thousands of dollars; monitoring dashboards lag and freeze.

### Detection & Audit
- Audit metric dimensions in CloudWatch / OCI Monitoring for unique IDs or timestamps.

### Better Design
- Reserve **Metrics** for low-cardinality aggregate dimensions (e.g., `Region`, `StatusCode`, `Service`).
- Pass high-cardinality metadata (User ID, Order ID) inside **Structured Logs** or **Distributed Traces** (OpenTelemetry attributes), which are far cheaper to ingest and query.

### When It May Be Acceptable
Small-scale testing with strict short-term retention limits.
