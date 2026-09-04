# Module 21: Autoscaling, Throttling & Capacity Planning

> **Architectural Objective**: *Master dynamic cloud elasticity, reactive and predictive autoscaling policies, rate limiting and throttling mathematics (Token Bucket vs. Leaky Bucket), and hardware capacity reservation economics. Deconstruct AWS Auto Scaling and API Gateway throttling against OCI Autoscaling (Instance Pools) and OCI API Gateway rate limits, evaluate autoscaling flapping and cooldown mechanics, and architect guaranteed compute capacity across cloud availability zones and fault domains.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. Autoscaling Mechanisms & Scaling Policies](01-autoscaling-mechanisms-and-scaling-policies.md)** | Reactive vs. Predictive ML Scaling, AWS EC2 Auto Scaling Groups (Target Tracking, Step Scaling, Launch Templates), OCI Instance Pools & Metric/Schedule Policies, Cooldown & Flapping Mitigation | Full 20-Section Deep Dive (~2,400 words) |
| **[02. Rate Limiting Algorithms & Throttling](02-rate-limiting-algorithms-and-throttling.md)** | Mathematical Models (Token Bucket, Leaky Bucket, Sliding Window Log/Counter), AWS API Gateway Usage Plans & WAF Rate Rules vs. OCI API Gateway Rate Limits & OCI WAF, HTTP 429 & Retry-After | Full 20-Section Deep Dive (~2,400 words) |
| **[03. Cloud Capacity Reservations & Quotas](03-cloud-capacity-reservations-and-quotas.md)** | Guaranteed Hardware Capacity (AWS On-Demand Capacity Reservations / ODCR vs. OCI Capacity Reservations), Zonal Capacity Exhaustion during Outages, Service Quotas Governance | Abbreviated Capacity Guide (~950 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **Target Tracking vs. Step Scaling Decision Framework**: When to deploy Target Tracking scaling (ALB Request Count Per Target) versus Step Scaling for sudden flash-mob traffic spikes, and how to calibrate cooldown periods to prevent autoscaling thrashing.
2. **Throttling Algorithm Mechanics**: The exact mathematical mechanics of the Token Bucket algorithm used by AWS API Gateway and OCI API Gateway, and how burst capacities buffer traffic spikes without rejecting requests.
3. **Zonal Capacity Reservation for Disaster Recovery**: Why standard Auto Scaling Groups fail during regional disaster recovery failovers due to cloud hypervisor capacity exhaustion (`InsufficientInstanceCapacity`), and how to guarantee instance availability using On-Demand Capacity Reservations (ODCR) and OCI Capacity Reservations.
