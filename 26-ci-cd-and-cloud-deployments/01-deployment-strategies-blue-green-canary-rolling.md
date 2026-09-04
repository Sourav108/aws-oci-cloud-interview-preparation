# Production Deployment Strategies: Blue/Green, Canary, Rolling & Database Evolution (AWS vs. OCI)

---

## 1. Executive Summary & Core Definitions

In modern cloud engineering, deploying new software to production must be a routine, low-stress, zero-downtime event occurring multiple times per day. The traditional model of shutting down servers at midnight, running a 4-hour deployment script, and praying that the application restarts cleanly is completely incompatible with high-availability cloud architecture.

A modern deployment strategy manages the transition of user traffic between the existing software version (v1) and the candidate release (v2). It must guarantee zero service interruption, bound the blast radius of latent bugs, enable instantaneous automated rollbacks, and seamlessly coordinate database schema evolutions across asynchronous application fleets.

```
+---------------------------------------------------------------------------------------------------+
|                            THE THREE PILLARS OF ZERO-DOWNTIME DEPLOYMENT                          |
+---------------------------------------------------------------------------------------------------+
| 1. ROLLING UPDATE (Surge / Drain)     2. BLUE / GREEN (Instant Cutover)   3. CANARY (Progressive) |
| Pod 1 (v1) -> Pod 1 (v2)              [ Blue Fleet (v1) ] 100% Traffic    [ Stable Fleet (v1) ] 95%|
| Pod 2 (v1) -> Pod 2 (v2)                             \                    [ Canary Fleet (v2) ] 5% |
| Pod 3 (v1) -> Pod 3 (v2)                              v (Switch LB)                                |
| Zero Downtime; Moderate Skew          [ Green Fleet (v2) ] 100% Traffic   Automated Metric Audit   |
+---------------------------------------------------------------------------------------------------+
```

### Core Terminology
* **Rolling Update**: A strategy where instances or pods running the old application version are incrementally replaced by instances of the new version over time. Controlled via `maxSurge` (temporary excess capacity) and `maxUnavailable` (maximum allowable offline capacity).
* **Blue/Green Deployment (Red/Black)**: A strategy that provisions two identical, independent physical environments: "Blue" (active production processing live user traffic) and "Green" (staged release candidate). Traffic is shifted instantaneously at the load balancer or DNS layer once Green passes synthetic testing.
* **Canary Deployment**: A progressive release pattern where the new software version is exposed to a tiny fraction of live user traffic (e.g., 2% to 5%) or a specific user demographic. Automated Canary Analysis (ACA) evaluates real-time telemetry (p99 latency, HTTP 5xx errors) before incrementally scaling traffic to 100%.
* **Version Skew**: The operational state during a deployment where both version v1 and version v2 are simultaneously running, accepting user traffic, and querying the exact same production database.
* **Expand-Contract Pattern (Parallel Run)**: A multi-phase database refactoring pattern that guarantees breaking schema changes remain backward- and forward-compatible across concurrent application versions.

---

## 2. Distributed Systems Theory & Architecture

### The Problem of Version Skew & Concurrent Fleets

During both Rolling Updates and Canary Deployments, version skew is mathematically unavoidable. If a deployment takes 10 minutes to roll out across a 100-pod cluster:

```
T = 0m:   100 pods on v1 | 0 pods on v2    (Pure v1 State)
T = 5m:    50 pods on v1 | 50 pods on v2   (MAXIMUM VERSION SKEW: 50% / 50%)
T = 10m:    0 pods on v1 | 100 pods on v2  (Pure v2 State)
```

During the window $0 < t < 10\text{m}$, user request $A$ hits a pod running v1, while user request $B$ hits a pod running v2.

```
Request A (v1) ---> [ Application Pod v1 ]
                             \
                              +====> [ SHARED PRODUCTION DATABASE ]
                             /
Request B (v2) ---> [ Application Pod v2 ]
```

#### The Fundamental Architectural Consequence:
If version v2 requires a destructive database schema modification (e.g., renaming `column_a` to `column_b` or dropping a table), the instant the database migration executes, **all 50 active v1 pods will immediately crash with SQL runtime exceptions** (`ColumnNotFoundException`).

---

### The Expand-Contract (Parallel Run) Pattern

To eliminate version skew crashes, database schema changes must be executed in decoupled phases spanning multiple software deployment cycles:

```
+---------------------------------------------------------------------------------------+
| PHASE 1: EXPAND (Database Migration 1)                                                |
| Add new column 'full_name' alongside legacy 'first_name' and 'last_name'.             |
| Rule: New column MUST be nullable or have a safe default.                             |
| Status: v1 app continues running with zero disruption.                                |
+---------------------------------------------------------------------------------------+
                                           |
                                           v
+---------------------------------------------------------------------------------------+
| PHASE 2: DUAL WRITE (Deploy Application v2)                                           |
| Deploy v2 application code.                                                           |
| Behavior: Reads from 'full_name'; writes to BOTH ('first_name', 'last_name') AND      |
|           'full_name' simultaneously.                                                 |
| Status: v1 and v2 coexist safely during rolling update.                               |
+---------------------------------------------------------------------------------------+
                                           |
                                           v
+---------------------------------------------------------------------------------------+
| PHASE 3: ASYNCHRONOUS BACKFILL (Background Worker)                                    |
| Run an offline background script to populate 'full_name' for all historic rows.       |
| Status: Production traffic continues unaffected.                                      |
+---------------------------------------------------------------------------------------+
                                           |
                                           v
+---------------------------------------------------------------------------------------+
| PHASE 4: CONTRACT (Deploy Application v3 & Final Schema Cleanup)                      |
| Deploy v3 (Reads/writes exclusively to 'full_name').                                   |
| Once v1 and v2 are completely decommissioned, execute final schema migration:         |
| DROP COLUMN 'first_name', DROP COLUMN 'last_name'.                                    |
+---------------------------------------------------------------------------------------+
```

---

## 3. Core Mechanics & Deep Dive

### AWS Deployment Architecture: CodeDeploy & ALB Weighted Targets

AWS provides fine-grained traffic shifting primitives combining Application Load Balancers (ALBs) with AWS CodeDeploy:

```
[ Internet Traffic ]
         |
         v
[ AWS Application Load Balancer ]
         |
         +--- Target Group A (Blue Fleet: v1) [ Weight: 90% ] ---> EC2 / ECS / EKS
         |
         +--- Target Group B (Green Fleet: v2)[ Weight: 10% ] ---> EC2 / ECS / EKS
```

1. **ALB Weighted Target Groups**:
   * ALB listener rules support routing traffic across multiple target groups with integer weights (e.g., 90 to Target Group Blue, 10 to Target Group Green).
   * Supports **stickiness per target group**, ensuring a user whose session begins on the canary (Green) remains pinned to Green during their session.
2. **AWS CodeDeploy Deployment Configurations**:
   * **Linear**: Traffic shifts in equal increments over time (e.g., `Linear10PercentEvery1Minute` shifts 10% every minute, completing in 10 minutes).
   * **Canary**: Traffic shifts a small percentage, pauses for verification, then cuts over 100% (e.g., `Canary10Percent5Minutes` routes 10% to v2, waits 5 minutes while monitoring CloudWatch alarms, then shifts the remaining 90%).
   * **Automated Rollback Hooks**: CodeDeploy monitors CloudWatch Alarms (5xx errors, ALB target latency). If any alarm fires during traffic shifting, CodeDeploy immediately resets ALB weights to 100% Blue in **less than 2 seconds**.

---

### OCI Deployment Architecture: OCI DevOps & Load Balancer Routing

Oracle Cloud Infrastructure enforces zero-downtime deployments via the **OCI DevOps Service** paired with OCI Flexible Load Balancers:

```
[ Internet Traffic ]
         |
         v
[ OCI Flexible Load Balancer ]
         |
         +--- Backend Set A (Production Fleet: v1) [ Weight: 90 ] ---> OCI Compute / OKE
         |
         +--- Backend Set B (Canary Fleet: v2)     [ Weight: 10 ] ---> OCI Compute / OKE
                                 ^
                                 |
              [ OCI DevOps Deployment Pipeline ]
              - Stage 1: Deploy Canary Instances
              - Stage 2: Shift LB Weight to 10%
              - Stage 3: Automated Approval / Alarm Audit (5m)
              - Stage 4: Shift LB Weight to 100%
              - Stage 5: Terminate v1 Fleet
```

1. **OCI Load Balancer Weighted Backend Sets**:
   * OCI Flexible Load Balancers support dynamically updating backend weights (from 1 to 100) via the OCI API without dropping active TCP connections.
2. **OCI DevOps Deployment Pipeline Stages**:
   * **Canary Deployment Stage**: Automates deploying software to a designated canary compute instance pool or OKE pod replica set, configuring load balancer weights to a specified canary ratio.
   * **Automated Quality Gates**: Evaluates OCI Monitoring Alarms (e.g., `HttpRequests[1m]{responseCode = "500"}.sum() > 5`). If an alarm triggers during the soak period, the pipeline halts and executes automated rollback.
   * **Blue/Green Stage**: Manages environment swap by switching the Load Balancer Listener's default backend set pointer from Blue to Green.

---

## 4. Architecture & Data Flow Diagrams

### Automated Canary Analysis (ACA) Traffic Shift Sequence

```
Client Ingress           Edge Load Balancer           Canary Fleet (v2)       Telemetry Engine (CloudWatch/Prometheus)
      |                          |                            |                                |
[ T = 00:00: Baseline State ]    |                            |                                |
All traffic routed to Stable v1  |                            |                                |
      |                          |                            |                                |
[ T = 00:01: Deploy Canary ]     |                            |                                |
Pipeline launches Canary v2 --------------------------------->| Boots & passes readiness probe |
Pipeline updates LB Weight:      |                            |                                |
  Stable: 95% | Canary: 5% ----->|                            |                                |
      |                          |                            |                                |
[ T = 00:02: Shift 5% Traffic ]  |                            |                                |
95% Requests ------------------->| Stream to Stable (v1)      |                                |
 5% Requests ------------------->| Stream to Canary (v2) ---->|                                |
                                 |                            |--- Emit Error/Latency Spikes ->|
                                                                                               |
[ T = 00:04: Anomaly Detection ]                                                               |
CloudWatch / Prometheus evaluates SLI: p99 Latency > 2,000ms ----------------------------------+
Safety Alarm Fires: 'Canary-HighLatency-Alarm' = ALARM
                                 |
[ T = 00:05: AUTOMATED ROLLBACK ]|
Pipeline catches alarm trigger   |
Reverts LB Weight: 100% Stable ->|
Canary fleet v2 cordoned & terminated
All user traffic restored to 100% healthy v1 in < 2 seconds.
Blast radius strictly contained to 5% of traffic over a 3-minute window.
```

---

## 5. Side-by-Side Architectural Comparison: AWS vs. OCI

| Feature / Dimension | AWS Deployment Stack | OCI Deployment Stack |
| :--- | :--- | :--- |
| **Managed Pipeline Service** | AWS CodePipeline + CodeDeploy | **OCI DevOps Deployment Pipelines** |
| **Native Canary Support** | CodeDeploy Deployment Configurations | OCI DevOps Canary Deployment Stage |
| **Native Blue/Green Support**| CodeDeploy Blue/Green with ALB / ECS | OCI DevOps Blue/Green Deployment Stage |
| **Traffic Shifting Layer** | ALB Weighted Target Groups / Route 53 | OCI Load Balancer Weighted Backend Sets |
| **Kubernetes Progressive Delivery**| Argo Rollouts / Flagger on AWS EKS | Argo Rollouts / Flagger on OCI OKE |
| **Automated Rollback Triggers**| CloudWatch Alarms & Route 53 Health | **OCI Monitoring Alarms** (MQL metrics) |
| **Linear Deployment Pace** | Predefined (`Linear10PercentEvery1Minute`)| Fully customizable step percentage and intervals |
| **Session Stickiness on Canary**| ALB Target Group Stickiness | OCI Load Balancer Session Persistence Cookie |

---

## 6. Deep Implementation & Configuration (Terraform / CLI / SDK)

### AWS: ALB Weighted Target Groups for Canary Traffic Shifting (Terraform)

```hcl
# Application Load Balancer Listener with Weighted Forwarding Rules
resource "aws_lb_listener_rule" "canary_routing" {
  listener_arn = aws_lb_listener.front_end.arn
  priority     = 100

  action {
    type = "forward"

    forward {
      # Stable Production Target Group (90% Traffic)
      target_group {
        arn    = aws_lb_target_group.production_v1.arn
        weight = 90
      }

      # Canary Candidate Target Group (10% Traffic)
      target_group {
        arn    = aws_lb_target_group.canary_v2.arn
        weight = 10
      }

      # Enable stickiness so a user stays on canary for their session
      stickiness {
        enabled  = true
        duration = 300 # 5 minutes
      }
    }
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }
}
```

---

### OCI: Load Balancer Backend Set Weight Configuration (Terraform)

```hcl
# OCI Load Balancer Backend Set for Production Fleet (Weight 90)
resource "oci_load_balancer_backend_set" "stable_backend_set" {
  name             = "production-v1-backend-set"
  load_balancer_id = oci_load_balancer_load_balancer.public_lb.id
  policy           = "ROUND_ROBIN"

  health_checker {
    protocol          = "HTTP"
    port              = 8080
    url_path          = "/healthz"
    return_code       = 200
    interval_ms       = 5000
    timeout_in_millis = 2000
    retries           = 2
  }
}

# Add Backend Instances with Dynamic Weights
resource "oci_load_balancer_backend" "canary_backend_node" {
  load_balancer_id = oci_load_balancer_load_balancer.public_lb.id
  backend_set_name = oci_load_balancer_backend_set.stable_backend_set.name
  ip_address       = "10.0.1.50" # Canary instance private IP
  port             = 8080
  weight           = 10 # 10% share of traffic within backend set
  backup           = false
  drain            = false
  offline          = false
}
```

---

### Argo Rollouts Canary Specification for Kubernetes (EKS / OKE)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service-rollout
  namespace: production
spec:
  replicas: 10
  strategy:
    canary:
      # Integrated with ingress controller / service mesh
      steps:
        - setWeight: 5
        - pause: { duration: 10m } # Soak for 10 minutes at 5%
        - setWeight: 20
        - pause: { duration: 15m } # Soak for 15 minutes at 20%
        - setWeight: 50
        - pause: { duration: 10m }
      # Automated Canary Analysis (ACA)
      analysis:
        templates:
          - templateName: success-rate-analysis
        args:
          - name: service-name
            value: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-app
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/order-service:v2.1.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

---

## 7. Failure Modes, Edge Cases & Blast Radius

| Failure Mode | Trigger Mechanism | Blast Radius | Mitigation Strategy |
| :--- | :--- | :--- | :--- |
| **Breaking Database Schema Skew** | New code dropped column that existing v1 fleet was actively querying | 100% of v1 application fleet crashes during rolling update | Strictly mandate the **Expand-Contract Pattern** across separate deployment milestones. |
| **Sticky Canary Session Starvation** | Load balancer sticky sessions route a single heavy enterprise client to the canary | Canary pod crashes due to unexpected 100% load from one giant tenant | Use header-based canary routing or enforce per-client concurrency caps on canary pods. |
| **Silent Metric Blindspot** | Canary code contains a critical bug that does not emit HTTP 500s (e.g., writing corrupt data to DB) | Automated Canary Analysis passes; buggy code rolls out to 100% | Audit business domain metrics (e.g., checkout completion rate, payment success) in addition to HTTP 5xx codes. |
| **Zombie Pod Connection Bleed** | Kubernetes/EC2 terminates old version without waiting for active TCP connections to drain | Active user checkout requests severed midway (`502 Bad Gateway`) | Set `deregistration_delay.timeout_seconds = 30` on ALB and configure `terminationGracePeriodSeconds = 60` with `preStop` sleep hooks on pods. |

---

## 8. Security, Compliance & Threat Modeling

### Security Safeguards During Deployment Transitions

```
[ Canary Release v2 ] ---> Runs Vulnerability / Secret Scan in Production
                                    |
                                    v
                     [ Automated Compliance Verification ]
                     - Enforces non-root container user
                     - Enforces read-only root filesystem
                     - Validates mTLS mutual authentication
```

1. **Zero Downtime Vulnerability Patching**:
   * Blue/Green deployments allow deploying kernel patches and critical OpenSSL/Log4j zero-day security updates with **zero user disruption**.
2. **Access Control on Deployment Pipelines**:
   * Enforce strict separation of duties: developers can merge PRs to trigger automated deployments to Dev/Staging, but **Production Blue/Green cutover** requires an authorized approval gate from an SRE or SecOps lead.

---

## 9. Performance Tuning & Latency Engineering

### Connection Draining & Zero-Downtime Socket Handoff

```
Target Group Deregistration Delay (e.g., 30s)
[ Instance Deregistered ] ===> No New Requests Routed ===> In-Flight Requests Finish ===> Instance Terminated
```

1. **ALB Deregistration Delay (Connection Draining)**:
   * When an instance or pod is removed during a rolling update, the load balancer stops sending new requests and enters the **Deregistration Delay** window (default 300s, tuned to **30s** for microservices).
   * Ensures active long-running HTTP/2 streams and database transactions complete gracefully before SIGTERM/SIGKILL kills the container.

2. **Kubernetes `preStop` Lifecycle Hook**:
   * Kubernetes removes pod endpoints from kube-proxy asynchronously. To prevent ingress controllers from routing traffic to a dying pod:
     ```yaml
     lifecycle:
       preStop:
         exec:
           command: ["/bin/sh", "-c", "sleep 15"]
     ```
   * Gives ingress proxies 15 seconds to update routing tables before application process receives SIGTERM.

---

## 10. Observability, Telemetry & SRE Metrics

### Key Metrics for Automated Canary Analysis (ACA)

| Metric Name | Source | Description | Rollback Threshold |
| :--- | :--- | :--- | :--- |
| `HTTPCode_Target_5XX_Count` | AWS CloudWatch (ALB) | Total 5xx server errors emitted by target group | > 5 errors in 1 minute |
| `TargetResponseTime` | AWS CloudWatch (ALB) | Latency of the backend application response | p99 > 500 ms (or +25% over baseline) |
| `HttpRequests_Canary_5xx` | OCI Monitoring | HTTP 5xx error rate on OCI canary backend set | > 1% of total canary traffic |
| `OrderCheckoutSuccessRate` | Business Telemetry | Custom application metric tracking completed checkouts | Drop > 2% relative to stable fleet |

---

## 11. Cost Modeling & Capacity Planning

### Infrastructure Cost Overhead by Deployment Strategy

| Strategy | Compute Overhead During Deploy | Deployment Duration | Infrastructure Cost Premium |
| :--- | :--- | :--- | :--- |
| **Recreate / In-Place** | 0% (Uses existing nodes) | Fast (~2m) | **$0 (Zero Overhead)** |
| **Rolling Update** | +25% (`maxSurge = 25%`) | Moderate (~10m) | Minimal (Billed for extra pods for 10m) |
| **Canary Deployment** | +5% to +10% (Canary pods) | Extended (~30m soak)| Negligible (Billed for canary pods) |
| **Blue / Green** | **+100% (2x full production fleet)** | Fast (~5m cutover) | **+100% for deployment window duration** |

*Staff Insight*: While Blue/Green requires doubling your compute capacity, modern containerized architectures on EKS/OKE provision and terminate nodes dynamically within minutes. Running a duplicate 50-pod green fleet for 20 minutes costs **less than $5.00** in cloud compute fees, rendering "cost of duplicate compute" an obsolete excuse for avoiding Blue/Green in modern cloud architectures.

---

## 12. Operational Runbooks & Triage Playbooks

### Runbook: Manual Emergency Abort of Canary Deployment

```
[ PagerDuty Alert: Canary v2 Triggering Customer Complaints on Twitter/Support ]
                                    |
                                    v
                 Step 1: Execute Instant Traffic Reversion
       AWS: Reset ALB Target Group Weights:
       aws elbv2 modify-listener --listener-arn ... --default-actions ...

       OCI: Update Load Balancer Backend Set:
       oci lb backend update --weight 0 --backend-name 10.0.1.50:8080 ...
                                    |
                                    v
                 Step 2: Terminate Canary Workload
       Cordon and delete canary pods:
       kubectl delete pods -l app=order-service,version=v2
                                    |
                                    v
                 Step 3: Capture Diagnostic Data
       Export canary container logs and crash dumps to S3/Object Storage
       for offline engineering postmortem analysis
```

---

## 13. Edge Cases, Quirks & Gotchas

### Subtle Deployment Traps

1. **Client-Side Cache Poisoning on CDN Edge**:
   * If a new JavaScript/CSS bundle is deployed to S3/CloudFront with identical filenames (`app.js` instead of `app.a8f9b2.js`), browsers cache stale files while HTML references new code, breaking user UIs.
   * *Mandate*: Always enforce **content-hashed immutable filenames** (`bundle.[hash].js`) with long cache headers (`Cache-Control: max-age=31536000, immutable`).
2. **Kubernetes `maxSurge` Exceeding Cloud Subnet IP Capacity**:
   * If your AWS VPC CNI or OCI VCN-Native Pod Networking subnet has only 20 free IP addresses, and you deploy a 100-pod deployment with `maxSurge: 25%` (requiring 25 new IP addresses), the deployment **hangs permanently in pending state**.
   * *Mitigation*: Ensure subnets have sufficient IP buffers or configure `maxUnavailable: 25%` with `maxSurge: 0`.

---

## 14. Real-World Case Study / Postmortem

### Incident: The Database Lock Deadlock Outage

* **Context**: Top-3 global real-estate portal doing $2B in monthly transactions.
* **The Incident**: The engineering team deployed an application update via a standard Kubernetes Rolling Update.
* **The Root Cause**:
  1. The deployment migration script included an `ALTER TABLE listings ADD COLUMN verified_date TIMESTAMP NOT NULL DEFAULT NOW();` on a table with 80 million rows.
  2. In PostgreSQL, adding a column with a volatile default prior to Postgres 11 acquired an **AccessExclusiveLock**, locking the entire table for 45 seconds.
  3. Active v1 pods waiting on database queries exhausted their connection pools within 10 seconds.
  4. The rolling update failed; ALB marked all pods unhealthy, taking down the entire website for 18 minutes.
* **The Fix**: Mandated the Expand-Contract pattern: add nullable columns first without default values, deploy code that populates the column, and backfill historic rows asynchronously in background micro-batches.

---

## 15. Architectural Trade-Off Analysis

| Strategy | Rollback Speed | Version Skew Risk | Resource Overhead | Validation Depth |
| :--- | :--- | :--- | :--- | :--- |
| **Recreate** | Slow (Re-pull old image) | **Zero (Downtime enforced)** | 0% | Poor |
| **Rolling Update** | Moderate (Roll backward) | High (v1 and v2 coexist) | Low (+25%) | Moderate |
| **Blue / Green** | **Instantaneous (< 2s)** | **Zero (Clean cutover)** | High (+100%) | **Outstanding (Full test)** |
| **Canary** | Fast (< 5s) | High (v1 and v2 coexist) | Low (+5-10%) | **Maximum (Real traffic)** |

---

## 16. Cloud Migration & Dual-Cloud Considerations

### Multi-Cloud Cross-Provider Canary Shifting

In an advanced resilience architecture spanning AWS and OCI:

```
[ Global Traffic Steering: Anycast / Cloudflare ]
                        |
            +-----------+-----------+
            |                       |
   [ AWS Target: 90% ]     [ OCI Target: 10% (Canary) ]
   - EKS Production v1     - OKE Release Candidate v2
```

* **Cloud-to-Cloud Canary**: Direct 10% of global ingress traffic to an OCI cluster to validate software compatibility, networking performance, and cost benchmarks under live production workloads before executing a full platform migration.

---

## 17. Automated Verification & Testing

### Script: Automated Canary Health Check & Rollback Monitor (Python)

```python
import boto3
import time
import sys

def monitor_canary_health(alb_arn, target_group_arn, soak_seconds=300):
    """
    Monitors CloudWatch metrics for an active canary target group.
    Aborts and triggers rollback if 5xx errors exceed threshold during soak period.
    """
    cw = boto3.client('cloudwatch', region_name='us-east-1')
    print(f"Beginning Automated Canary Analysis for: {target_group_arn}")
    print(f"Soak period: {soak_seconds} seconds...")

    start_time = time.time()
    while time.time() - start_time < soak_seconds:
        # Query 5xx errors emitted by canary target group in the last 60 seconds
        res = cw.get_metric_data(
            MetricDataQueries=[{
                'Id': 'm1',
                'MetricStat': {
                    'Metric': {
                        'Namespace': 'AWS/ApplicationELB',
                        'MetricName': 'HTTPCode_Target_5XX_Count',
                        'Dimensions': [{'Name': 'TargetGroup', 'Value': target_group_arn.split('/')[-2] + '/' + target_group_arn.split('/')[-1]}]
                    },
                    'Period': 60,
                    'Stat': 'Sum'
                }
            }],
            StartTime=boto3.utils.rfc3339_to_datetime('2026-09-04T00:00:00Z'),
            EndTime=boto3.utils.rfc3339_to_datetime('2026-09-04T00:10:00Z')
        )

        values = res['MetricDataResults'][0]['Values']
        recent_errors = values[0] if values else 0
        print(f"Canary 5xx Error Rate (Last 60s): {recent_errors}")

        if recent_errors > 5:
            print("CRITICAL: Canary error threshold breached! TRIGGERING IMMEDIATE ROLLBACK.")
            sys.exit(2) # Exit code 2 triggers rollback in pipeline

        time.sleep(30)

    print("SUCCESS: Canary soak period completed with zero anomalies. Cutover approved.")

if __name__ == "__main__":
    pass
```

---

## 18. Staff+ Engineering Wisdom & Insights

### Production Deployment Golden Rules

1. **Deployments Must Be Reversible in Under 60 Seconds**: If a deployment takes 45 minutes to roll back, it is not a safe deployment. Architect every release so that an emergency rollback is a simple load balancer weight shift or DNS toggle.
2. **If You Haven't Solved Database Migrations, You Haven't Solved Zero Downtime**: Anyone can do a rolling update of stateless web servers. The mark of senior and staff engineering maturity is orchestrating zero-downtime database schema evolutions using the Expand-Contract pattern.
3. **Automate the Abort, Not Just the Deploy**: Do not rely on human engineers watching dashboards to decide whether to roll back a canary. Wire automated health checks to instant circuit breakers that revert traffic the moment p99 latency spikes.

---

## 19. Interview Defense & Technical Discussion Scenarios

### Scenario 1: Designing a Zero-Downtime Deployment for a High-Volume API

* **Interviewer**: "We process 50,000 requests/second on an API backed by PostgreSQL. How do you roll out a major version update that renames a core database column with zero downtime?"
* **Staff Candidate Response**:
  1. *Deconstruct the Deployment*: Split the rollout into **3 distinct pull requests** across separate days using the **Expand-Contract Pattern**.
  2. *Step 1 (Expand)*: Author a migration adding `new_column` as nullable. Deploy migration to PostgreSQL. v1 application code is completely unaffected.
  3. *Step 2 (Dual Write & Canary Deploy)*: Deploy v2 application code via **Canary Deployment** (ALB weighted target groups shifting 5% $\to$ 25% $\to$ 100%). v2 writes to both `old_column` and `new_column`, reading from `new_column`. If v2 encounters bugs, rollback to v1 is safe because `old_column` has been continuously kept in sync.
  4. *Step 3 (Backfill)*: Run an offline script in batches of 1,000 rows (`UPDATE table SET new_column = old_column WHERE new_column IS NULL`) to populate historic rows without table lock contention.
  5. *Step 4 (Contract)*: Deploy v3 (removes legacy read/write code). Once v1/v2 are completely gone, drop `old_column` from the database.

### Scenario 2: Choosing Between Blue/Green and Canary Deployments

* **Interviewer**: "When would you mandate Blue/Green over Canary, and vice versa?"
* **Staff Candidate Response**:
  * *Choose Blue/Green*: When deploying mission-critical enterprise workloads where **zero version skew is permissible** (e.g., core financial ledger systems, ERP systems where v1 and v2 simultaneously running would cause data inconsistencies), or when testing complex infrastructure upgrades (e.g., upgrading an entire Kubernetes cluster version from 1.28 to 1.30).
  * *Choose Canary*: When operating massive consumer-facing web or mobile applications (e.g., Netflix, Spotify, Amazon) where real-world production user behavior, client cache variations, and localized network conditions cannot be fully replicated in synthetic staging environments. Exposing 2% of live users to the candidate release surfaces latent bugs while protecting 98% of the customer base.

---

## 20. Summary Cheat Sheet & Reference Matrix

```
+---------------------------------------------------------------------------------------------------+
|                            DEPLOYMENT STRATEGIES CHEAT SHEET                                      |
+--------------------------+------------------------------------+-----------------------------------+
| Characteristic           | Blue / Green Deployment            | Canary Deployment                 |
+--------------------------+------------------------------------+-----------------------------------+
| Primary Focus            | Instantaneous zero-skew cutover    | Risk mitigation via real traffic  |
| Rollback Speed           | **Instantaneous (< 2 seconds)**    | **Fast (< 5 seconds)**            |
| Version Skew Coexistence | Zero (Strict cutover)              | Yes (v1 and v2 run concurrently)  |
| Compute Cost Overhead    | +100% (Duplicate fleet during test)| +5% to +10% (Canary pods only)    |
| Database Compatibility   | Requires forward-compatible schema | Requires Expand-Contract pattern  |
| AWS Native Engine        | CodeDeploy + ALB Target Groups     | CodeDeploy Linear/Canary configs  |
| OCI Native Engine        | OCI DevOps Blue/Green Stage        | OCI DevOps Canary Stage           |
| Kubernetes Standard      | Argo Rollouts / Service selector   | Argo Rollouts / Flagger           |
+--------------------------+------------------------------------+-----------------------------------+
```
