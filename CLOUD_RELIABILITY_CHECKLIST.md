# Cloud Reliability, Fault Tolerance & High Availability Checklist

> **Foundational Axiom**: *Systems fail; hardware breaks; networks partition. In Senior and Staff engineering interviews, reliability is not measured by the absence of failures, but by the elegance, speed, and automation with which a system detects, isolates, degrades, and self-heals under duress.*
>
> **The Golden Rule**: *"A backup that has never been restored is not a backup—it is merely a comforting hallucination."*

---

## 🏗️ The 12 Reliability Engineering Pillars

### 1. Multi-Availability Zone & Fault Domain Redundancy
- [ ] **Minimum 3-Way Redundancy**:
  - AWS: Compute ASGs and managed databases span at least **3 distinct Availability Zones** (AZs). Never deploy production single-AZ.
  - OCI: Compute Instance Pools span all **3 Availability Domains** (in multi-AD regions) or distribute across **3 Fault Domains** (in single-AD regions).
- [ ] **Anti-Affinity Rules**: Ensure clustered container pods or VM instances enforce pod anti-affinity / placement groups to prevent co-locating critical replicas on the same underlying physical host.
- [ ] **Capacity Headroom**: Maintain $N+1$ or $N+2$ capacity headroom per AZ so that the total loss of an entire AZ allows remaining zones to absorb 100% of peak load without immediate autoscaling delay.

### 2. Deep Health Checks & Traffic Steering
- [ ] **Calibrated Liveness vs. Readiness Probes**:
  - *Liveness*: Checks only if the process is responsive. If failing, restart container/process.
  - *Readiness*: Checks if the node can serve traffic (database connection pool healthy, cache warm). If failing, remove from load balancer pool; **do not restart**.
- [ ] **Health Check Dampening & Hysteresis**:
  - Require at least 2–3 consecutive successful checks before marking a node healthy.
  - Require at least 3 consecutive failed checks before evicting a node to avoid flapping during transient network blips.
- [ ] **Graceful Connection Draining**:
  - Configure Load Balancer Deregistration Delay (AWS ALB) / Connection Draining (OCI LB) (e.g., 30–60s) allowing in-flight HTTP requests to complete before terminating instances.

### 3. Automated Failover & State Recovery
- [ ] **Managed Database High Availability**:
  - AWS Aurora: Multi-AZ cluster with automated storage replication across 3 AZs; automated master failover in < 30 seconds with virtual IP / CNAME switch.
  - OCI Base DB / Autonomous DB: Active Data Guard configured with automated fast-start failover (FSFO) between primary and standby database instances.
- [ ] **Stateless Application Layer**: Ensure zero local state is stored on application VM/container disk. All session state is offloaded to distributed cache (Redis) or database.

### 4. Resilient Retries & Throttling Mitigation
- [ ] **Exponential Backoff with Full Jitter**:
  - Ban static retries. Calculate backoff delay with randomized jitter:
    $$t_{\text{sleep}} = \text{random}(0, \min(t_{\text{max}}, t_{\text{base}} \cdot 2^{\text{attempt}}))$$
- [ ] **Capped Retry Budgets**: Limit retries to a maximum of 3 attempts or allocate a 10% global retry budget in client connection pools to prevent retry storms.
- [ ] **Downstream Circuit Breakers**: Wrap external HTTP and database calls in circuit breakers (Envoy / Resilience4j). Trip open when failure rate exceeds 50% over a 10-second sliding window, failing fast without queuing.

### 5. Socket, Request & Propagation Timeouts
- [ ] **Defensive Timeout Hierarchies**: Every layer in the call stack must have a strictly shorter timeout than the layer above it:
  $$\text{Client Timeout} > \text{API Gateway Timeout} > \text{Load Balancer Timeout} > \text{App Server Timeout} > \text{Database Query Timeout}$$
- [ ] **Explicit Connect & Read Timeouts**: Override language defaults. Set connect timeout to 500ms–1000ms; set socket read timeout to $p99 + \text{buffer}$.
- [ ] **Deadline Propagation**: Propagate request deadlines (via gRPC context or HTTP headers) so downstream services abort execution if the upstream caller has already timed out.

### 6. Idempotent API & Message Processing
- [ ] **Idempotency Keys for Mutating Operations**: Require client-supplied `Idempotency-Key` headers on POST requests. Cache result in Redis for 24 hours to return identical responses on retries.
- [ ] **Deduplication in Message Queues**:
  - AWS SQS: Configure FIFO queues with message deduplication IDs or track processed message IDs in DynamoDB with conditional writes.
  - OCI Queue: Implement consumer-side deduplication tables in PostgreSQL or OCI NoSQL.

### 7. Capacity Planning & Load Shedding
- [ ] **Predictive & Metric-Based Autoscaling**:
  - Autoscale on leading indicators (e.g., SQS queue backlog, ALB request count per target) rather than lagging indicators (CPU utilization).
- [ ] **Graceful Load Shedding**: When CPU or queue depth reaches 85%, shed non-critical background traffic (e.g., analytics, email notifications) and return HTTP 503 (`Retry-After: 30`) to protect core transactional endpoints.

### 8. Backup Validation & Automated Restore Drills
- [ ] **Automated Snapshot Scheduling**:
  - Enforce automated daily point-in-time snapshots for RDS/Aurora and OCI Block Volumes with 35-day retention.
- [ ] **Automated Restore Verification (CART)**:
  - Run a weekly automated script that restores the latest production snapshot into an ephemeral sandbox, executes synthetic validation queries, and destroys the sandbox.
- [ ] **Documented & Tested RPO / RTO**:
  - Define explicit Recovery Point Objective (RPO) and Recovery Time Objective (RTO) per service tier and validate during game-day drills.

### 9. Disaster Recovery (DR) Strategy
- [ ] **Geographic Diversity**: Establish a secondary region at least 300+ miles away from the primary region to mitigate natural disasters or widespread grid blackouts.
- [ ] **Automated DNS Failover**:
  - Configure Route 53 Failover Routing or OCI DNS Traffic Management Failover steering policies tied to external health check endpoints.
- [ ] **Infrastructure as Code Parity**: Maintain identical Terraform configurations for both primary and secondary regions, validated through regular CI runs.

### 10. Observability, Golden Signals & Alerting
- [ ] **The Four Golden Signals**:
  - *Latency*: Track p50, p95, and p99 latency per route.
  - *Traffic*: Track total incoming requests per second.
  - *Errors*: Track HTTP 5xx error rates and unhandled exceptions.
  - *Saturation*: Track CPU, memory, connection pool, and disk IOPS saturation.
- [ ] **Multi-Window Multi-Burn Rate Alerts**: Alert on SLO error budget consumption rates rather than static thresholds to minimize alert fatigue.

### 11. Blast Radius Containment & Bulkheads
- [ ] **Cell-Based Architecture**: Segment massive workloads into independent, isolated "cells" (e.g., per region, tenant tier, or shard) so that a failure in one cell never cascades to another.
- [ ] **Thread & Connection Pool Bulkheads**: Allocate isolated connection pools for critical vs. non-critical downstream dependencies. A freeze in the reporting database must never exhaust connections needed by the checkout flow.

### 12. Chaos Engineering & Game-Day Drills
- [ ] **Automated Failure Injection**: Use AWS Fault Injection Simulator (FIS) or Chaos Mesh on Kubernetes to inject latency, terminate instances, and sever network paths in staging.
- [ ] **Game-Day Rehearsals**: Conduct scheduled quarterly failover drills simulating a full cloud region outage, testing team operational runbooks and runbook accuracy.
