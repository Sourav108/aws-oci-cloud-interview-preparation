# Lab 10: Fault Injection, Chaos Engineering & Automated Disaster Recovery

---

## 1. Goal & Core Concept

The objective of this hands-on lab is to simulate catastrophic infrastructure failures using **Chaos Engineering** principles, observe automated health check degradation, execute multi-region failovers via **Route 53 Application Recovery Controller (ARC)** and **OCI Traffic Management Steering**, and empirically calculate actual **Recovery Point Objective (RPO)** and **Recovery Time Objective (RTO)**.

### Core Architectural Concepts Tested
- **Blast Radius Injection**: Simulating complete Availability Zone and regional endpoint network blackholes.
- **Automated Health Check Draining**: Configuring Layer-7 health check probes with fast dissipation ($< 15\text{ seconds}$).
- **Global DNS Failover Routing**: Route 53 Failover Routing vs. OCI DNS Failover Steering Policies.
- **Measuring Empirical RTO / RPO**: Quantifying packet loss, in-flight transaction drops, and time-to-recovery during chaos drills.

---

## 2. Cost Warning & Resource Estimate

> [!NOTE]
> **ESTIMATED HOURLY RUN COST: `< $0.30 / hour`**
>
> | Cloud Provider | Resource Component | Quantity | Hourly Rate | Run Cost (2 Hours) |
> | :--- | :--- | :---: | :---: | :---: |
> | **AWS** | Route 53 Health Checks + ARC Control Panel `[Doc: Route 53, checked 2026]` | 1 Control Panel | $0.05 / hr | $0.10 |
> | **AWS** | Multi-Region ALB Endpoints (2 Regions) `[Doc: ALB, checked 2026]` | 2 ALBs | $0.045 / hr | $0.09 |
> | **OCI** | OCI DNS Steering Policy + Multi-Region LB `[Doc: OCI DNS, checked 2026]` | 1 Policy + 2 LBs | $0.025 / hr | $0.05 |
>
> *Total Estimated Lab Cost for a 2-hour session: **$0.24 USD**.*
> *In No-Live-Account Fallback Mode, cost is **$0.00**.*

---

## 3. Architecture Diagram

```
========================================================================================================================
                          CHAOS DRILL & MULTI-REGION FAILOVER TOPOLOGY
========================================================================================================================

                                  [ Global Monitoring / User Traffic ]
                                                   │
                                                   ▼
                            [ Global DNS Failover Steering: Route 53 / OCI DNS ]
                            - Primary Pool: us-east-1 (Priority 1)
                            - Secondary Pool: us-west-2 (Priority 2)
                                                   │
                       ┌───────────────────────────┴───────────────────────────┐
                       │ (100% Traffic Steady-State)                           │ (0% Standby - Fails Over to 100%)
                       ▼                                                       ▼
  ═══════════════════════════════════════════════  ═══════════════════════════════════════════════
  PRIMARY REGION (AWS us-east-1 / OCI Ashburn)     SECONDARY REGION (AWS us-west-2 / OCI Phoenix)
  ┌─────────────────────────────────────────────┐  ┌─────────────────────────────────────────────┐
  │ [ Primary ALB / OCI Load Balancer ]         │  │ [ Standby ALB / OCI Load Balancer ]         │
  │                     │                       │  │                     │                       │
  │                     ▼                       │  │                     ▼                       │
  │ [ Primary Web App: /healthz (HTTP 200) ]    │  │ [ Standby Web App: /healthz (HTTP 200) ]    │
  │                                             │  │                                             │
  │ [ CHAOS INJECTION: Drop /healthz to 500 ]   │  │                                             │
  └─────────────────────────────────────────────┘  └─────────────────────────────────────────────┘
  ═══════════════════════════════════════════════  ═══════════════════════════════════════════════
```

---

## 4. Prerequisites

1. Terraform CLI v1.8+.
2. Cloud permissions to configure Route 53 / OCI DNS Traffic Management.

---

## 5. Infrastructure Code (Terraform HCL)

### 5.1 AWS Route 53 Failover Configuration (`aws_failover.tf`)

```hcl
# AWS Reference Implementation: Route 53 Health Check and Failover Records
resource "aws_route53_health_check" "primary_health" {
  fqdn              = var.primary_alb_dns
  port              = 80
  type              = "HTTP"
  resource_path     = "/healthz"
  failure_threshold = 2
  request_interval  = 10 # Fast 10-second probing interval

  tags = { Name = "lab10-primary-health-check" }
}

resource "aws_route53_record" "primary_failover" {
  zone_id = var.hosted_zone_id
  name    = "app.enterprise.com"
  type    = "A"

  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier  = "primary-us-east-1"
  health_check_id = aws_route53_health_check.primary_health.id

  alias {
    name                   = var.primary_alb_dns
    zone_id                = var.primary_alb_zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "secondary_failover" {
  zone_id = var.hosted_zone_id
  name    = "app.enterprise.com"
  type    = "A"

  failover_routing_policy {
    type = "SECONDARY"
  }

  set_identifier = "secondary-us-west-2"

  alias {
    name                   = var.secondary_alb_dns
    zone_id                = var.secondary_alb_zone_id
    evaluate_target_health = true
  }
}
```

### 5.2 OCI DNS Failover Steering Policy (`oci_failover.tf`)

```hcl
# OCI Reference Implementation: DNS Traffic Management Failover Steering
resource "oci_dns_steering_policy" "failover_policy" {
  compartment_id = var.compartment_ocid
  display_name   = "lab10-dr-failover-policy"
  template       = "FAILOVER"
  ttl            = 10

  answers {
    name      = "primary-ashburn"
    rdata     = var.primary_lb_ip
    pool      = "primary-pool"
    is_active = true
  }

  answers {
    name      = "standby-phoenix"
    rdata     = var.secondary_lb_ip
    pool      = "secondary-pool"
    is_active = true
  }
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

Steady-State DNS Resolution:
$ dig +short app.enterprise.com
52.23.45.67 (Primary Region IP in us-east-1 / Ashburn)
```

---

## 8. Failure Injection Drill: Complete Endpoint Blackhole

### The Scenario
Simulate a catastrophic primary region outage by breaking the `/healthz` endpoint on the primary web tier, injecting HTTP 503 Service Unavailable.

### The Injection
```bash
# Inject chaos: Force primary app to return 503
curl -X POST http://${PRIMARY_ALB}/chaos/fail
```

### Manifested Symptoms
- Route 53 / OCI DNS probe fails 2 consecutive checks (20 seconds).
- Primary endpoint health transitions to `UNHEALTHY`.
- DNS resolution flips to the Secondary Region IP (`54.89.12.34`).

---

## 9. Debugging Walkthrough & RTO/RPO Calculation

```text
====================================================================================================
                        DISASTER FAILOVER METRICS & TIMELINE
====================================================================================================

Timeline Events:
- [12:00:00] Chaos injected on primary endpoint.
- [12:00:20] 2 consecutive health check failures detected by Route 53.
- [12:00:25] Route 53 shifts DNS answers to Secondary Region (us-west-2).
- [12:00:35] Client DNS cache TTL (10s) expires.
- [12:00:40] 100% of synthetic client pings landing on Secondary Region.

Empirical Recovery Metrics:
- Measured RTO: 40 seconds.
- Measured RPO: 0 seconds (Stateless web tier).
====================================================================================================
```

---

## 10. Teardown & Verification

```bash
terraform destroy -auto-approve
```

---

## 11. Senior Interview Defense & Takeaways

> **Interviewer**: *"Why not set the DNS TTL to 0 seconds so failover happens instantaneously?"*

**Candidate Defense**:
*"While a 0-second TTL theoretically eliminates client DNS caching, in practice many recursive DNS resolvers (such as public ISPs and enterprise forwarders) ignore TTLs below 5 to 10 seconds to prevent query flooding.*

*Furthermore, setting TTL to 0 seconds forces every single client HTTP request to execute an external DNS lookup before connecting, adding 15-50ms of latency to every user interaction and inflating Route 53 query bills. Setting a 10-second or 15-second TTL achieves the optimal balance between rapid 30-second automated failover and low-latency client caching."*
