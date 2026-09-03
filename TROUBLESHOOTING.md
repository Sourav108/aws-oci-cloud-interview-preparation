# The Scientific Cloud Troubleshooting Methodology

> **The Senior SRE Axiom**: *Junior engineers guess and reboot. Senior and Staff engineers observe, isolate blast radius, form testable hypotheses, collect evidence, eliminate root causes systematically, verify remediations, and engineer prevention into the architecture.*
>
> **Anti-Pattern Warning**: *Never execute "restart everything" as step one during an outage. Blind reboots destroy volatile kernel buffers, wipe transient socket states, eliminate forensic core dumps, and often worsen outages by triggering violent thundering-herd reconnect storms.*

---

## 🔬 The 8-Stage Troubleshooting Loop

```text
1. OBSERVE      ──> Detect Anomalies, Alarms & Customer Symptoms
      │
2. SCOPE        ──> Define Blast Radius (Region, AZ, Tenant, Route)
      │
3. HYPOTHESIZE  ──> Form Mutually Exclusive, Testable Failure Hypotheses
      │
4. COLLECT      ──> Inspect Metrics, Distributed Traces & Flow Logs
      │
5. ELIMINATE    ──> Falsify Hypotheses with Hard Telemetric Evidence
      │
6. MITIGATE     ──> Apply Surgical Remediation to Restore Service Fast
      │
7. VERIFY       ──> Confirm Golden Signals Return to Steady State
      │
8. PREVENT      ──> Automate Guardrails & Conduct Blameless Postmortem
```

---

## 🧭 Step-by-Step Incident Execution Protocol

### Step 1: Observe & Triage
- **Acknowledge the Signal**: What triggered the investigation? (SLO error budget burn rate alarm, customer ticket surge, synthetic canary probe failure).
- **Inspect Golden Signals**:
  - *Latency*: Is the surge at the edge, application runtime, or database?
  - *Traffic*: Is there an unexpected traffic surge (DDoS, crawler) or sudden traffic drop?
  - *Errors*: What are the exact HTTP status codes? (4xx client vs 5xx server).
  - *Saturation*: Are CPU, memory, socket descriptors, or disk IOPS maxed out?

### Step 2: Scope the Blast Radius
Determine the exact boundary of the disruption before touching anything:
- **Geographic Scope**: Is it global, multi-region, single-region, or single-AZ/AD?
- **Network Scope**: Is it affecting public internet ingress, VPC peering, or internal private endpoints?
- **Identity / Tenant Scope**: Is it impacting all customers or a specific shard/tenant?
- **Deployment Scope**: Was a release, Terraform apply, or configuration change deployed in the last 60 minutes?

### Step 3: Formulate Scientific Hypotheses
Formulate at least 3 distinct, testable hypotheses based on the symptoms:
- *Hypothesis A (Network/Transport)*: Upstream load balancer cannot reach backend targets due to security group rule corruption or routing table failure.
- *Hypothesis B (Resource Saturation)*: Backend application connection pool to PostgreSQL is exhausted due to an unindexed query lock.
- *Hypothesis C (External Dependency)*: Downstream payment gateway is timing out, causing worker threads to block until thread pools are exhausted.

### Step 4: Collect Telemetric Evidence
Gather objective proof to validate or refute each hypothesis:
1. **Metrics**: Correlate timestamps across ALB `TargetResponseTime`, EC2 `CPUUtilization`, and RDS `DatabaseConnections`.
2. **Distributed Tracing (AWS X-Ray / OCI APM)**: Examine the trace waterfall of slow requests. Which specific span dominates latency?
3. **Structured Logs (CloudWatch Logs / OCI Logging)**: Query logs around the degradation onset using CloudWatch Logs Insights or OCI Log Analytics.
4. **Packet & Network Flow Logs**: Inspect VPC Flow Logs or VCN Flow Logs for `REJECT` flags between client and server IP addresses.

### Step 5: Systematically Eliminate Root Causes
- Disprove hypotheses using hard negative evidence:
  - *"If CPU is at 25% and memory is stable, we can eliminate host hardware saturation."*
  - *"If VPC Flow Logs show `ACCEPT` on port 5432, we can eliminate Security Group network blockage."*
- Drill down on the surviving hypothesis until the singular point of failure is identified.

### Step 6: Surgical Mitigation (Restore Service First)
The goal during an active outage is **restoration of service**, not root-cause investigation.
- **Rollback**: If triggered by a recent deployment, immediately roll back to the previous stable release.
- **Traffic Shedding / Rerouting**: If an entire AZ is impaired, adjust Route 53 / OCI DNS health checks or ALB target group configurations to route traffic away from the degraded zone.
- **Capacity Scaling**: If traffic exceeded forecast, scale out the compute fleet or increase database IOPS.
- **Circuit Breaking**: Trip the circuit breaker on the failing non-critical downstream dependency to return graceful fallbacks.

### Step 7: Verify Recovery
- Confirm the system has returned to steady state:
  - Do HTTP 5xx errors return to baseline ($<0.01\%$)?
  - Has p99 latency stabilized below SLA?
  - Are queues draining normally without backlog accumulation?
- Monitor for at least 30 minutes to ensure recovery is not transient or creating secondary thundering-herd issues.

### Step 8: Prevent Recurrence & Blameless Postmortem
- Conduct a blameless postmortem within 48 hours:
  - Construct a chronological timeline of events.
  - Apply the **5 Whys** methodology to identify systemic and architectural root causes rather than human error.
  - Create actionable engineering tasks with assigned owners: add automated canary rollbacks, implement circuit breakers, configure VPC endpoints, improve alert sensitivity.
