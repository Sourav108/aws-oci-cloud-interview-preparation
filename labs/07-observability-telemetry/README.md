# Lab 07: Distributed Observability, OpenTelemetry & Multi-Burn Rate Alerting

---

## 1. Goal & Core Concept

The objective of this hands-on lab is to instrument a microservices workload with **OpenTelemetry (OTel)**, collect distributed traces, stream structured JSON logs, and configure multi-burn-rate error budget alerting across **AWS CloudWatch / X-Ray** and **OCI Monitoring / APM**.

### Core Architectural Concepts Tested
- **OpenTelemetry Collector Daemonset**: Exporting traces and metrics to cloud vendor backends without proprietary SDK lock-in.
- **The 4 Golden Signals**: Implementing real-time tracking for Latency, Traffic, Errors, and Saturation.
- **Distributed Context Propagation**: Passing W3C TraceContext headers (`traceparent`) across microservice boundaries.
- **Multi-Window Multi-Burn-Rate Alerting**: Implementing Google SRE error budget burn rate calculations (14.4x 1-hour burn vs. static threshold alerting).

---

## 2. Cost Warning & Resource Estimate

> [!NOTE]
> **ESTIMATED HOURLY RUN COST: `< $0.10 / hour`**
>
> | Cloud Provider | Resource Component | Quantity | Hourly Rate | Run Cost (2 Hours) |
> | :--- | :--- | :---: | :---: | :---: |
> | **AWS** | CloudWatch Metrics & Logs (< 1 GB logs) `[Doc: CloudWatch, checked 2026]` | 1 Log Group | $0.50 / GB ingestion | $0.01 |
> | **AWS** | AWS X-Ray Tracing (10,000 sampled traces) `[Doc: X-Ray, checked 2026]` | 10k Traces | Free Tier (100k free/mo) | $0.00 |
> | **OCI** | OCI Logging & Monitoring `[Doc: OCI Logging, checked 2026]` | 1 Log Group | Free Tier (10 GB free/mo) | $0.00 |
> | **OCI** | OCI APM (Application Performance Monitoring) `[Doc: OCI APM, checked 2026]` | 1 Domain | Free Tier | $0.00 |
>
> *Total Estimated Lab Cost for a 2-hour session: **$0.01 USD**.*
> *In No-Live-Account Fallback Mode, cost is **$0.00**.*

---

## 3. Architecture Diagram

```
========================================================================================================================
                          DISTRIBUTED OBSERVABILITY & TRACING TOPOLOGY
========================================================================================================================

  [ Client HTTP Request ] (Injects: traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01)
            │
            ▼
  [ API Service Container ]
    ├── Microservice Handler (Logs structured JSON: {"level": "info", "trace_id": "...", "duration_ms": 14.2})
    └── OTel SDK (Sends traces & metrics via gRPC localhost:4317)
            │
            ▼
  [ OpenTelemetry Collector Sidecar / Daemonset ]
            │
       ┌────┴───────────────────────────┐
       │ (AWS Exporters)                │ (OCI Exporters)
       ▼                                ▼
  [ AWS CloudWatch Logs & X-Ray ]       [ OCI Logging & OCI APM Domain ]
  - Metric Filter: 5xx Error Count      - MQL Metric Rule: ErrorRate > 2%
  - Composite Burn-Rate Alarm:          - Alarm Notification: PagerDuty / Slack
    14.4x burn rate over 1-hr window
========================================================================================================================
```

---

## 4. Prerequisites

1. Terraform CLI v1.8+.
2. Kubernetes or container environment to deploy OTel agent.

---

## 5. Infrastructure Code (Terraform & Collector Config)

### 5.1 OpenTelemetry Collector Configuration (`otel_collector.yaml`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        timeout: 1s
        send_batch_size: 256
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 20

    exporters:
      awsxray:
        region: us-east-1
      awsemf:
        region: us-east-1
        log_group_name: "/aws/ecs/otel-telemetry"
      otlp/oci:
        endpoint: https://apm-dotnet.<region>.oci.oraclecloud.com/20200101/observations/public-span?dataFormat=otlp&dataFormatVersion=1.0.0
        headers:
          Authorization: "dataKey ${OCI_APM_DATA_KEY}"

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [awsxray, otlp/oci]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [awsemf]
```

### 5.2 AWS Multi-Burn Rate CloudWatch Alarm (`aws_alarms.tf`)

```hcl
# AWS Reference Implementation: Multi-Burn-Rate Error Budget Alarm (14.4x 1-hr burn)
resource "aws_cloudwatch_metric_alarm" "p1_burn_rate_alarm" {
  alarm_name          = "p1-error-budget-burn-rate-14x"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "5xxErrorCount"
  namespace           = "CustomApp/Tier1"
  period              = 300 # 5-minute evaluation
  statistic           = "Sum"
  threshold           = 14.4 # Consumes 2% of budget in 1 hour
  alarm_description   = "Critical Error Budget Burn Rate Exceeded"
  alarm_actions       = [var.pagerduty_sns_topic_arn]
}
```

---

## 6. Step-by-Step Deployment Guide

```bash
terraform init -backend=false
terraform validate
terraform plan
```

---

## 7. Expected Validation Results

```text
[Statically validated — not applied to a live account]

Structured JSON Log Entry Validation:
{
  "timestamp": "2026-09-07T12:00:00.123Z",
  "level": "INFO",
  "service": "order-api",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "http_method": "POST",
  "http_route": "/api/v1/orders",
  "http_status": 200,
  "duration_ms": 18.4
}
```

---

## 8. Failure Injection Drill: Error Budget Exhaustion

### The Scenario
Simulate a buggy software release that returns `HTTP 500` on 15% of checkout requests, rapidly burning the monthly error budget.

### The Injection
```bash
# Inject 500 Internal Server Errors for 10 minutes
for i in {1..200}; do curl -s -X POST https://${API_URL}/api/v1/orders -d '{"trigger_bug": true}'; done
```

### Manifested Symptoms
- CloudWatch `5xxErrorCount` exceeds threshold.
- The 1-hour fast-burn alert trips in $< 5\text{ minutes}$.
- Automated rollback webhook triggers in CI/CD pipeline.

---

## 9. Debugging Walkthrough

```text
====================================================================================================
                        DISTRIBUTED TRACE LATENCY DRILL
====================================================================================================

$ aws xray get-trace-summaries --start-time 1725720000 --end-time 1725723600     --filter-expression 'annotation.http_status = 500'
Output:
Trace ID: 1-5f0c1234-abcd...
Segment Timeline:
  - Ingress Proxy: 1.2ms
  - Order Controller: 2.1ms
  - Payment Client: 25,000ms [FAILED: Downstream Read Timeout]
Root Cause: External payment processor socket timeout exhausted container connection pool.
====================================================================================================
```

---

## 10. Teardown & Verification

```bash
terraform destroy -auto-approve
```

---

## 11. Senior Interview Defense & Takeaways

> **Interviewer**: *"Why use multi-burn-rate error budget alerting instead of traditional static threshold alarms (e.g., alert if CPU > 80% or error rate > 1%)?"*

**Candidate Defense**:
*"Traditional static threshold alarms suffer from catastrophic false-positive and false-negative failure modes. Alerting when error rate exceeds 1% might page on-call engineers at 3 AM for a brief 10-second blip that consumed only 0.001% of the monthly error budget. Conversely, a subtle 0.2% error rate sustained for two weeks will burn 100% of an error budget without ever crossing a 1% alarm threshold!*

*Google SRE multi-burn-rate alerting calculates the mathematical rate of error budget consumption. A 14.4x burn rate over 1 hour consumes 2% of our 30-day budget—an emergency warranting immediate paging. A 6x burn rate over 6 hours consumes 5%—warranting a high-priority ticket. This eliminates alert fatigue while guaranteeing immediate notification on true degradation."*
