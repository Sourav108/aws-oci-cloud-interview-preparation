# 01. CloudWatch vs. OCI Monitoring & Metric Alerting

## 1. Problem
In distributed cloud architectures, relying on naive, static threshold alerting (e.g., *"Page on-call engineer if CPU > 80% for 5 minutes"*) is the single primary cause of severe **Alert Fatigue**. A fleet of worker instances batch-processing video encoding will naturally pin CPU at 100% without customer impact, waking up SREs at 3:00 AM. Conversely, a critical silent bug returning HTTP 500 errors to 5% of checkout users might leave CPU utilization at a normal 25%, evading static CPU monitors entirely. Modern cloud reliability engineering requires **Service Level Objectives (SLOs), Error Budget burn-rate mathematics, and multi-window statistical alerting** implemented across AWS CloudWatch and OCI Monitoring.

## 2. Cloud Concept
### Push vs. Pull Metric Architectures
- **Pull-Based Metrics (Prometheus / OTel Pull)**:
  - The monitoring server scrapes an HTTP `/metrics` endpoint exposed by target instances at fixed scrape intervals (e.g., every 15 or 30 seconds).
  - *Risk*: If network firewalls block the scraper, or target pods crash before being scraped, metric samples are lost.
- **Push-Based Metrics (AWS CloudWatch / OCI Monitoring)**:
  - Hypervisors, managed services, and application agents push time-series metric data points directly to the cloud telemetry ingestion API over HTTPS.
  - *Advantages*: Serverless services (Lambda, Functions) and short-lived batch jobs push their final execution duration metrics immediately before terminating.

### Metric Resolution & Retention Horizons
Cloud metric engines automatically downsample historical metrics over time to balance query performance and storage economics:
- **AWS CloudWatch Metric Resolution & Retention**:
  - *High-Resolution Metrics*: Captured at **1-second, 5-second, 10-second, or 30-second intervals**. Retained at 1-second resolution for **3 hours** `[Doc: Amazon CloudWatch Retention, checked 2026-09-04]`.
  - *Standard Resolution*: 1-minute data points (retained for 15 days).
  - *5-minute rollup*: Retained for 63 days.
  - *1-hour rollup*: Retained for **455 days (15 months)**.
- **OCI Monitoring Service (MQL Engine)**:
  - Employs **Monitoring Query Language (MQL)**: a powerful, expressive functional query syntax for aggregating, filtering, and performing statistical transforms across time-series metrics.
  - Metrics are stored for **90 days by default**.
  - Supports cross-compartment metric querying in a single unified MQL expression!

### SRE Error Budget Mathematics & Burn-Rate Alerting
Instead of alerting on CPU or memory, Google SRE practices mandate alerting on **Error Budget Consumption**:
1. **Service Level Indicator (SLI)**: The quantifiable metric:
   $$\text{SLI} = \frac{\text{Count of Successful HTTP Requests (Status } < 500)}{\text{Total Count of HTTP Requests}}$$
2. **Service Level Objective (SLO)**: The agreed reliability target (e.g., $99.9\%$ availability over a 30-day rolling window).
3. **Error Budget**: The allowable unreliability:
   $$\text{Error Budget} = 100\% - 99.9\% = 0.1\% = 0.001$$
4. **Burn Rate ($B$)**: The rate at which you are consuming your error budget. A burn rate of $1\text{x}$ consumes exactly 100% of your error budget in 30 days.
   - **Multi-Window Multi-Burn-Rate Alerts**:
     - *Emergency Page (Burns 2% of budget in 1 hour)*:
       $$\text{Burn Rate} = \frac{30\text{ days} \times 24\text{ hrs}}{1\text{ hr}} \times 0.02 = \mathbf{14.4\text{x Burn Rate}}.$$
     - *Standard Ticket (Burns 5% of budget in 6 hours)*:
       $$\text{Burn Rate} = \frac{720\text{ hrs}}{6\text{ hrs}} \times 0.05 = \mathbf{6.0\text{x Burn Rate}}.$$
     - SREs are paged **only when an ongoing failure is mathematically fast enough to exhaust the monthly error budget**, completely eliminating alert fatigue!

## 3. Mental Model
Think of cloud metric alerting as an aircraft cockpit:
- **Static Threshold Alerts** is an alarm buzzing whenever the airplane's engines exceed 3,000 RPM. When the pilot throttles up to climb above a storm, the cockpit screams in panic even though the flight is 100% safe.
- **SLO Error Budget Burn-Rate Alerting** is a smart flight navigation computer calculating: *"Based on current fuel consumption and distance to the runway, you will run out of fuel in 45 minutes"*. It alerts the pilot only when catastrophic failure is mathematically guaranteed without intervention.

## 4. Architecture Diagram
```text
SRE ERROR BUDGET BURN-RATE ALERTING TOPOLOGY:

[Application API Fleet]
        │
        ├──► Emits Metrics: RequestCount & ErrorCount (Every 10s)
        │
        ▼
┌────────────────────────────────────────────────────────────────────────┐
│ CLOUD TELEMETRY ENGINE (AWS CloudWatch / OCI Monitoring)               │
│                                                                        │
│   METRIC MATH / MQL EXPRESSION:                                        │
│   SLI = 1 - (Sum(5xx_Errors) / Sum(Total_Requests))                    │
│   Current_Burn_Rate = (1 - SLI) / 0.001 (For 99.9% SLO)                │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ SHORT WINDOW (1 Hour): Burn Rate > 14.4x                       │   │
│   │ AND LONG WINDOW (5 Mins): Burn Rate > 14.4x                    │   │
│   │ ──► CONSUMING 2% OF MONTHLY BUDGET IN 60 MINUTES!              │   │
│   └──────────────────────────────┬─────────────────────────────────┘   │
└──────────────────────────────────┼─────────────────────────────────────┘
                                   │ Dual-Condition Met (Zero Flapping!)
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│ INCIDENT RESPONSE: PagerDuty / OCI Notifications                       │
│ * Pages Primary On-Call Engineer: High Severity Production Outage!     │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS CloudWatch:
- **CloudWatch Metric Math**:
  - Allows combining multiple metrics using mathematical and statistical operators:
    ```text
    e1 = (m1 / m2) * 100  # Percentage calculation
    e2 = IF(e1 > 95, 1, 0)
    ```
- **Composite Alarms**:
  - Alarms that evaluate the state of multiple underlying metric alarms using boolean logic (`AND`, `OR`, `NOT`).
  - Eliminates alert storms during network outages:
    `ALARM(AppDown) AND NOT ALARM(NetworkOutage)` prevents 50 microservices from all paging simultaneously when a central NAT Gateway goes offline.
- **High-Resolution Metrics**:
  - Can be published via `PutMetricData` with `StorageResolution = 1`.
  - Enables sub-10 second automated anomaly detection for high-frequency algorithmic trading systems.

## 6. OCI Implementation
In Oracle Cloud Infrastructure Monitoring:
- **The Monitoring Query Language (MQL)**:
  - An expressive, programmatic query syntax `[Doc: OCI Monitoring MQL, checked 2026-09-04]`:
    ```sql
    CpuUtilization[1m]{resourceId = "ocid1.instance..."}.mean() > 80
    ```
  - **Cross-Compartment Telemetry Aggregation**:
    - Query metrics across all compartments in a tenancy:
      ```sql
      CpuUtilization[5m].grouping().mean()
      ```
    - Allows calculating organizational-wide CPU or network utilization in a single chart.
- **OCI Alarms Engine**:
  - Evaluates MQL expressions against defined thresholds.
  - Supports **Split Notifications**: sends notification when the alarm triggers, and a **Resolution Notification** when the metric returns to healthy levels.
  - Integrates natively with **OCI Notifications (ONS)** to dispatch webhooks directly to Slack, PagerDuty, or trigger auto-remediation via **OCI Functions**.
- **Metric Ingestion Architecture**:
  - OCI services emit metrics automatically into standard service namespaces (`oci_computeagent`, `oci_vcn`, `oci_autonomous_database`).
  - Supports custom metric ingestion via the OCI Monitoring REST API.

## 7. Configuration
Comparing metric alarms in Terraform across AWS and OCI:

### AWS CloudWatch Metric Math Alarm (Terraform)
```hcl
# AWS CloudWatch Alarm with Metric Math (Error Rate %)
resource "aws_cloudwatch_metric_alarm" "api_error_rate" {
  alarm_name          = "api-high-error-rate-burn"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  threshold           = 5.0 # 5% Error Rate Threshold!
  alarm_actions       = [var.pagerduty_sns_topic_arn]

  metric_query {
    id          = "e1"
    expression  = "(m2 / m1) * 100"
    label       = "Error Rate Percentage"
    return_data = "true"
  }

  metric_query {
    id = "m1"
    metric {
      metric_name = "RequestCount"
      namespace   = "AWS/ApplicationELB"
      period      = 60
      stat        = "Sum"
      dimensions  = { LoadBalancer = var.alb_arn_suffix }
    }
  }

  metric_query {
    id = "m2"
    metric {
      metric_name = "HTTPCode_Target_5XX_Count"
      namespace   = "AWS/ApplicationELB"
      period      = 60
      stat        = "Sum"
      dimensions  = { LoadBalancer = var.alb_arn_suffix }
    }
  }
}
```

### OCI Monitoring Alarm with MQL Query (Terraform)
```hcl
# OCI Monitoring Alarm using MQL Expression
resource "oci_monitoring_alarm" "high_cpu_alarm" {
  compartment_id        = var.compartment_id
  destinations          = [var.ons_topic_id]
  display_name          = "high-compute-cpu-burn"
  is_enabled            = true
  metric_compartment_id = var.compartment_id
  namespace             = "oci_computeagent"

  # MQL Query Expression
  query                 = "CpuUtilization[1m]{compartmentId = \"${var.compartment_id}\"}.mean() > 85"
  severity              = "CRITICAL"
  pending_duration      = "PT5M" # Must persist for 5 minutes!

  body                  = "Critical: Compute pool in production compartment exceeds 85% CPU utilization."
}
```

## 8. Data Flow
```text
SRE Multi-Burn-Rate Alert Evaluation:
1. Microservice emits metrics: Requests (10,000), 5xx Errors (150).
2. Cloud Telemetry Engine ingests points every 60s.
3. Engine calculates SLI = 1 - (150 / 10,000) = 98.5% Availability.
4. Error Budget Target is 99.9% (Allowable Error = 0.1%).
5. Current Error Rate = 1.5%.
6. Current Burn Rate = 1.5% / 0.1% = 15x Burn Rate!
7. Check Conditions:
   - Is Burn Rate > 14.4x in the 1-hour window? YES.
   - Is Burn Rate > 14.4x in the 5-minute confirmation window? YES.
8. Dual-window rule met: Dispatches PagerDuty alert in < 3 minutes of outage!
```

## 9. Security
- **Telemetry Data Ingestion Security**:
  - Custom metrics pushed to CloudWatch or OCI Monitoring must authenticate via IAM Roles / Instance Principals.
  - Never include PII (Personally Identifiable Information, email addresses, credit cards) in metric dimensions. Metric dimensions are stored in cleartext and indexed globally across monitoring dashboards.

## 10. Reliability
- **Mitigating Alert Flapping via Evaluation Periods**:
  - Setting `evaluation_periods = 1` causes alarms to flap continuously during momentary 1-second CPU spikes.
  - Enforce `evaluation_periods = 3` and `datapoints_to_alarm = 2` (e.g., must breach in 2 of the last 3 checks), filtering transient noise while catching sustained degradation.

## 11. Scaling
- **Cardinality Explosion in Metric Dimensions**:
  - High-cardinality dimensions (e.g., tagging metrics with `user_id` or `order_id`) create millions of unique metric time-series streams.
  - In AWS CloudWatch, every custom metric time-series costs **\$0.30 per month**. A cardinality explosion with 500,000 unique user IDs generates a surprise **\$150,000/month bill**!
  - *Golden Rule*: Keep metric dimensions low-cardinality (`region`, `environment`, `http_status_class`). Store high-cardinality data in **Logs or Traces**.

## 12. Observability
- **Synthesizing Metrics with Dashboards**:
  - AWS CloudWatch Dashboards and OCI Console Dashboards combine metrics, alarms, and log query results in a single unified glass pane.

## 13. Cost
- **Metric Billing Math**:
  - AWS CloudWatch: First 10 metrics are free; **\$0.30 per metric-month** for the first 10,000 metrics `[Doc: Amazon CloudWatch Pricing, checked 2026-09-04]`.
  - OCI Monitoring: First 500 million metric ingestion data points and 1 billion retrieval data points per month are **100% Free** under the OCI Always Free tier `[Doc: OCI Monitoring Pricing, checked 2026-09-04]`.

## 14. Failure Modes
- **The "Everything is Green" False Negative Outage**: An application crashes and stops emitting metrics entirely. The CloudWatch alarm is configured with `treat_missing_data = "ignore"`. Because no new data points arrive, the alarm assumes the system is healthy and stays `OK` while users experience a total outage! *Remediation: Always set `treat_missing_data = "breaching"` on availability alarms.*
- **The Alert Storm During Database Maintenance**: A planned database reboot takes down 20 connected microservices. 20 separate teams receive urgent pages simultaneously, causing mass chaos. *Remediation: Use Composite Alarms with maintenance window suppression.*

## 15. Troubleshooting
When alarms fail to trigger:
1. **Check Missing Data Treatment**:
   - Verify `treat_missing_data` configuration (`missing`, `ignore`, `breaching`, `notBreaching`).
2. **Inspect Metric Timestamp Alignment**:
   - Ensure the application server clock is synchronized via NTP. If server timestamps are skewed by $> 15\text{ minutes}$, CloudWatch drops the data points as invalid.
3. **Verify Alarm Evaluation Window**: Check if the metric query period matches the alarm period.

## 16. Common Mistakes
- **Alerting on Raw Instance CPU Instead of User Experience**: Paging engineers on high worker CPU. Workers are meant to utilize CPU. Alert on **Latency (p99)** and **HTTP Error Rates**, not CPU.
- **Using 1-Minute Period for Instant Disaster Alarms**: Setting a 15-minute evaluation period on customer checkout failure, delaying incident response by a quarter of an hour.

## 17. Trade-offs
| Telemetry Strategy | Detection Speed | Alert Fatigue Risk | Financial Cost |
| :--- | :--- | :--- | :--- |
| **Static Thresholds (CPU > 80%)** | Slow / Misleading | **Severe (Constant false alarms)**| Low |
| **SLO Multi-Burn-Rate** | **Fast (< 3 mins for severe)**| **Lowest (Pages only on real impact)**| Moderate |
| **1-Second High-Resolution** | Near Real-Time (< 5s) | Moderate | Higher (\$0.30/metric) |
| **OCI MQL Cross-Compartment** | High | Low | **Free (Always Free Tier)** |

## 18. Interview Questions
1. *Why does static threshold alerting (e.g., CPU > 80% or ErrorCount > 10) inevitably fail in enterprise distributed systems? Explain how Google SRE multi-window burn-rate alerting eliminates alert fatigue while guaranteeing fast incident response.*
2. *What is Cardinality Explosion in time-series metrics? How does adding a single high-cardinality dimension trigger a \$100,000 cloud bill shock in AWS CloudWatch?*
3. *How does OCI Monitoring Query Language (MQL) differ from AWS CloudWatch Metric Math in syntax and cross-compartment querying capabilities?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "Static threshold alerting fails in production distributed systems because it fundamentally confuses **System Resource Consumption** with **User Experience Degradation**:
>
> 1. **The Flaws of Static Thresholds**:
>    - **False Positives (Alert Fatigue)**: A batch-processing worker pool naturally runs at 95% CPU during nightly reconciliation. Static alerts wake up engineers at 3:00 AM for healthy system behavior. Over time, engineers ignore all alerts.
>    - **False Negatives (Silent Outages)**: A critical bug causes 3% of checkout payments to fail with HTTP 500. Because traffic is distributed across 100 containers, aggregate CPU remains at 25%, and raw error count per instance stays below 10. Static monitors stay green while the business loses revenue.
>
> 2. **The SRE Multi-Window Multi-Burn-Rate Architecture**:
>    - We replace static metrics with an **SLO-based Error Budget Burn-Rate model**:
>      - We define an SLI: Availability = $\frac{\text{Successful Requests}}{\text{Total Requests}}$.
>      - We set a 30-day SLO of **99.9% Availability**, yielding an **Error Budget of 0.1%**.
>      - The **Burn Rate** measures the speed of budget consumption ($1\text{x}$ consumes 100% of the budget in 30 days).
>
> 3. **The Dual-Window Firing Condition**:
>    - To catch rapid, critical outages, we alert when **Burn Rate > 14.4x** (consuming 2% of our monthly budget in 1 hour).
>    - To prevent alert flapping, we enforce a **Dual-Window Requirement**:
>      1. *Long Window (1 Hour)*: Verifies that the 14.4x burn rate is sustained.
>      2. *Short Window (5 Minutes)*: Verifies that the issue is actively ongoing right now (not an isolated blip that already resolved).
>
> 4. **The Operational Outcome**:
>    - Critical outages that burn budget rapidly page on-call engineers in **under 3 minutes**.
>    - Slow-burning bugs generate low-priority Jira tickets during business hours.
>    - Transient network spikes page nobody, delivering 99.9% reliability with zero alert fatigue."

## 20. Hands-on Exercise
**Objective**: Create a CloudWatch Metric Math expression calculating 5xx Error Rate percentage and verify alarm state changes via CLI.

### Verification Steps
1. Create a CloudWatch Alarm evaluating error percentage via Metric Math.
2. Publish simulated request and error metrics using AWS CLI:
   ```bash
   aws cloudwatch put-metric-data --namespace "AppService" --metric-name "TotalRequests" --value 1000
   aws cloudwatch put-metric-data --namespace "AppService" --metric-name "5xxErrors" --value 150
   ```
3. Verify that the calculated error rate evaluates to $15\%$.
4. Check alarm state:
   ```bash
   aws cloudwatch describe-alarms --alarm-names "api-high-error-rate-burn" \
     --query "MetricAlarms[0].StateValue"
   ```
5. Confirm the alarm transitions to `ALARM` state within the evaluation window.
