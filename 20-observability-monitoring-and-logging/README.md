# Module 20: Observability, Monitoring & Logging

> **Architectural Objective**: *Master cloud telemetry engineering, the Three Pillars of Observability (Metrics, Logs, Distributed Tracing), Site Reliability Engineering (SRE) error budget mathematics, and automated incident triage. Deconstruct AWS CloudWatch, CloudTrail, and X-Ray against OCI Monitoring, OCI Logging Analytics, and OCI Application Performance Monitoring (APM), evaluate metric resolution and retention economics, and engineer resilient multi-window burn-rate alerting.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. CloudWatch vs. OCI Monitoring & Metrics](01-cloudwatch-vs-oci-monitoring-and-metrics.md)** | Metrics Architecture (Push vs. Pull, Standard vs. 1-Second High-Resolution), OCI MQL (Monitoring Query Language), SRE Error Budgets, Multi-Window Burn-Rate Alerts | Full 20-Section Deep Dive (~2,400 words) |
| **[02. CloudTrail, Flow Logs & OCI Logging](02-cloudtrail-flow-logs-and-oci-logging.md)** | Centralized Log Aggregation (CloudWatch Logs Insights vs. OCI Logging Analytics), Audit Governance (CloudTrail vs. OCI Audit 365-Day Stream), VPC/VCN Flow Logs | Full 20-Section Deep Dive (~2,400 words) |
| **[03. Distributed Tracing & APM](03-distributed-tracing-and-apm.md)** | W3C Trace Context & Spans, AWS X-Ray (Service Maps, Subsegments) vs. OCI APM, OpenTelemetry (OTel) Integration, Async p99 Latency Forensics | Abbreviated Observability Guide (~950 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **The Multi-Window Multi-Burn-Rate Alerting Architecture**: Why traditional static threshold alarms (e.g., CPU > 80%) generate crippling alert fatigue, and how to calculate Google SRE-style 14.4x (1-hour) and 6x (6-hour) error budget burn rates.
2. **Audit Telemetry & Forensic Immutability**: How AWS CloudTrail SHA-256 digest validation and OCI Audit's default 365-day immutable event stream provide tamper-evident compliance trails for SOC 2 and PCI-DSS audits.
3. **Distributed Trace Propagation & W3C Trace Context**: How trace headers (`traceparent`) traverse asynchronous messaging queues and API gateways to stitch microservice execution graphs together across multi-cloud boundaries.
