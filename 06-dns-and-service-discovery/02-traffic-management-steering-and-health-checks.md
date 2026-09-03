# 02. Traffic Management Steering & DNS Health Checks

## 1. Problem
When disaster strikes an entire cloud region or when rolling out canary releases across global user bases, engineers cannot reconfigure millions of client applications individually. Global DNS steering provides the primary control lever for routing traffic dynamically across infrastructure tiers. However, if architects fail to calibrate health check thresholds, configure DNS TTLs appropriately, or understand the difference between latency-based routing and geolocation steering, automated failover systems can trigger violent thundering-herd routing oscillations that knock down surviving infrastructure.

## 2. Cloud Concept
### Global DNS Routing Policies
Unlike traditional static DNS that unconditionally returns the same IP address to all callers, modern cloud authoritative DNS evaluates context-aware policies:

1. **Weighted Routing**: Distributes traffic across endpoints according to assigned percentage weights (e.g., 90% to Blue production, 10% to Green canary).
2. **Latency-Based Routing (LBR)**: Uses continuous network latency measurements across cloud edge networks to return the endpoint that provides the lowest round-trip latency for the querying resolver.
3. **Geolocation Routing**: Routes traffic based on the geographic location of the client resolver (mapped via IP-to-country/continent databases or EDNS0 Client Subnet). Essential for strict data sovereignty and localization.
4. **Geoproximity Routing**: Routes traffic based on physical geographic coordinates of resources and users, allowing fine-grained geographic bias adjustments.
5. **Failover Routing (Active-Passive)**: Returns a primary endpoint as long as its associated health check passes; automatically fails over to a secondary standby endpoint when the primary health check fails.
6. **Multi-Value Answer**: Returns up to 8 healthy IP addresses selected randomly, providing rudimentary DNS-based client-side load balancing and health check filtering.

### Active DNS Health Checks
Both AWS Route 53 and OCI DNS employ fleets of distributed global health check probers:
- Probers establish connections from multiple physical locations worldwide every 10 to 30 seconds.
- Support HTTP, HTTPS, and TCP probes with string matching (e.g., must return HTTP 200 and include `"status":"UP"` in the response body).
- When consecutive failures exceed a threshold (e.g., 3 failures), the authoritative DNS nameserver withdraws the unhealthy endpoint from active DNS answers.

## 3. Mental Model
Think of DNS traffic management as an intelligent global air traffic controller:
- **Static DNS** gives every pilot the exact same flight plan regardless of weather.
- **Latency-Based Routing** constantly monitors wind speeds and jet streams, instructing each flight to take the route that will arrive in the fewest minutes.
- **Failover Routing** monitors airport runways. If a blizzard closes the runway in Frankfurt, the controller automatically redirects incoming transatlantic flights to Munich before the planes take off.

## 4. Architecture Diagram
```text
ACTIVE-PASSIVE MULTI-REGION AUTOMATED DNS FAILOVER:

[Client Anywhere in the World]
              │
              ▼ 1. Query: api.globalcompany.com
┌─────────────────────────────────────────────────────────────┐
│  Route 53 Failover Routing / OCI DNS Failover Steering      │
│                                                             │
│  Global Health Check Probers (Probing every 10s from 8 DCs) │
│  ┌────────────────────────┐     ┌────────────────────────┐  │
│  │ Primary Probe: us-east │     │ Secondary Probe: eu-cen│  │
│  │ [HEALTHY: 200 OK]      │     │ [STANDBY READY]        │  │
│  └───────────┬────────────┘     └───────────┬────────────┘  │
└──────────────┼──────────────────────────────┼───────────────┘
               │ (Normal State: Serves 100%)  │ (Failover: Serves 100% when Primary Dead)
               ▼                              ▼
      [PRIMARY CLOUD REGION]        [SECONDARY CLOUD REGION]
      (Active Load Balancer)        (Standby Disaster Recovery Fleet)
```

## 5. AWS Implementation
In AWS Route 53:
- **Health Check Mechanics**:
  - Probers execute from 16+ global AWS data centers.
  - Probing frequency: **Standard (30s)** or **Fast (10s)** `[Doc: Amazon Route 53 Health Checks, checked 2026-09-03]`.
  - Failure threshold: Configurable from 1 to 10 consecutive failures (default: 3).
  - Minimum time to detect failure: $10\text{s interval} \times 3\text{ failures} = \mathbf{30\text{ seconds}}$.
- **Calculated Health Checks**: Route 53 allows combining up to 256 individual health checks using boolean logic (`AND`, `OR`, `NOT`). Example: Mark an entire region unhealthy only if *both* the web tier and database tier fail.
- **CloudWatch Alarm Integration**: Route 53 health checks can evaluate CloudWatch metrics (e.g., metric health checks triggering failover based on custom application error budget burn rates).
- **Latency-Based Routing (LBR)**: AWS maintains dynamic latency tables mapping global ISP network prefixes to AWS regions, updating latency metrics continuously.

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **OCI Traffic Management Steering Policies**:
  - OCI structures advanced DNS routing into dedicated, declarative **Steering Policies** `[Doc: OCI Traffic Management Steering Policies, checked 2026-09-03]`.
  - Supports four primary policy types:
    1. *Failover*: Active-passive or multi-tier priority pools with automated health check eviction.
    2. *Load Balancer*: Weighted round-robin across multiple backend endpoints or regions.
    3. *Geolocation Steering*: Routes queries based on the geographic continent, country, or state/province of the user.
    4. *ASN Steering*: Routes queries based on the client's Autonomous System Number (identifying specific ISPs or enterprise networks).
- **OCI DNS Health Checks**:
  - Built on the OCI Health Checks service, executing periodic HTTP/HTTPS/TCP probes from global vantage points.
  - Supports probing intervals down to **10 seconds** and allows deep pattern matching against response payloads.
- **Pool Prioritization in OCI Failover**:
  - In OCI, endpoints are grouped into **Pools** (Pool 1: Primary, Pool 2: Secondary, Pool 3: Tertiary).
  - If all endpoints in Pool 1 fail their health checks, the steering policy atomically fails over to Pool 2. When Pool 1 recovers, traffic automatically fails back.

## 7. Configuration
Comparing multi-region failover configuration in Terraform across AWS and OCI:

### AWS Route 53 Failover Routing (Terraform)
```hcl
# 1. Health Check for Primary Region (Fast 10s interval)
resource "aws_route53_health_check" "primary" {
  fqdn              = "api-us-east.mycompany.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  request_interval  = 10 # Fast interval!
  failure_threshold = 3
  tags = { Name = "primary-health-check" }
}

# 2. Primary Record (Active)
resource "aws_route53_record" "primary" {
  zone_id = var.hosted_zone_id
  name    = "api.mycompany.com"
  type    = "A"
  ttl     = 30

  failover_routing_policy {
    type = "PRIMARY"
  }
  set_identifier  = "primary-us-east"
  records         = [var.primary_alb_ip]
  health_check_id = aws_route53_health_check.primary.id
}

# 3. Secondary Record (Standby DR)
resource "aws_route53_record" "secondary" {
  zone_id = var.hosted_zone_id
  name    = "api.mycompany.com"
  type    = "A"
  ttl     = 30

  failover_routing_policy {
    type = "SECONDARY"
  }
  set_identifier = "secondary-eu-central"
  records        = [var.secondary_alb_ip]
}
```

### OCI DNS Traffic Management Failover Steering (Terraform)
```hcl
# OCI Traffic Management Failover Steering Policy
resource "oci_dns_steering_policy" "api_failover" {
  compartment_id = var.compartment_id
  display_name   = "api-global-failover-policy"
  template       = "FAILOVER"
  ttl            = 30

  # Define Pool 1: Primary Region (Ashburn)
  answers {
    name       = "primary-ashburn"
    rdata      = var.primary_lb_ip
    rtype      = "A"
    pool       = "primary-pool"
    is_disabled = false
  }

  # Define Pool 2: Standby Region (Phoenix)
  answers {
    name       = "standby-phoenix"
    rdata      = var.standby_lb_ip
    rtype      = "A"
    pool       = "standby-pool"
    is_disabled = false
  }

  # Health Check Attachment
  health_check_monitor_id = oci_health_checks_http_monitor.api_monitor.id

  rules {
    rule_type = "FAILOVER"
    cases {
      case_condition = "answer.pool == 'primary-pool'"
      answer_data {
        answer_condition = "answer.is_healthy == true"
        should_keep      = true
        value            = 1
      }
    }
  }
}
```

## 8. Data Flow
```text
The Automated Failover Sequence:
1. Normal State: Primary Region responds HTTP 200 to Health Check Probers.
   DNS returns Primary IP (54.210.10.20).
2. T0: Disaster strikes Primary Region (Power Grid Outage).
3. T0 + 10s: Prober 1 fails connection.
4. T0 + 20s: Prober 2 fails connection.
5. T0 + 30s: Prober 3 fails connection (Threshold 3 reached).
   * Health Check status changes to UNHEALTHY.
   * Route 53 / OCI DNS Steering updates internal Anycast routing engine.
6. T0 + 31s: New DNS query arrives.
   * DNS returns Secondary IP (198.51.100.50).
7. T0 + 31s to T0 + 61s (DNS TTL window):
   * Clients with cached DNS records continue attempting Primary.
   * As client TTLs expire (TTL = 30s), 100% of global traffic shifts to Secondary.
```

## 9. Security
- **Health Check Target Hardening**:
  - Health check endpoints (`/health`) should not expose sensitive internal telemetry (e.g., database connection strings, internal IP addresses).
  - Restrict health check endpoints to return lightweight payloads (HTTP 200 with `{"status":"UP"}`).
- **Mitigating DNS Failover Manipulation**:
  - An attacker executing an HTTP flood against the health check endpoint could trick Route 53 into failing over away from a healthy primary region.
  - Shield the health check endpoint behind AWS WAF or OCI WAF with strict rate-limiting rules.

## 10. Reliability
- **The DNS TTL vs. RTO Calculation**:
  $$\text{Minimum Total Failover Time} = \text{Detection Time} + \text{DNS TTL} = (\text{Interval} \times \text{Threshold}) + \text{TTL}$$
  For standard settings ($30\text{s interval} \times 3\text{ failures} = 90\text{s detection}$, plus $60\text{s TTL}$):
  $$\text{Total Failover Time} = 90\text{s} + 60\text{s} = \mathbf{150\text{ seconds}}.$$
- **Preventing Flapping (Hysteresis)**:
  Health checks enforce asymmetry between failure and recovery:
  - Requires 3 consecutive failures to mark unhealthy.
  - Requires 3 consecutive successes to mark healthy again, preventing rapid oscillation during intermittent network brownouts.

## 11. Scaling
- **EDNS0 Client Subnet (ECS)**:
  - Traditional DNS only sees the IP address of the recursive resolver (e.g., Google `8.8.8.8`), not the end user. If a user in Singapore uses a US-based DNS resolver, latency-based routing would mistakenly route them to a US region!
  - Route 53 and OCI DNS support **EDNS0 Client Subnet (RFC 7871)**, where compliant resolvers pass truncated client IP prefixes (e.g., `203.0.113.0/24`), allowing the cloud nameserver to make accurate routing decisions based on the actual client location.

## 12. Observability
- **Health Check Inversion & Alarms**:
  - CloudWatch alarms integrated directly with `HealthCheckStatus` (1 = Healthy, 0 = Unhealthy).
  - Invert health check option allows alerting when a disaster recovery standby node unexpectedly comes online.

## 13. Cost
- **Health Check Pricing**:
  - AWS charges **\$0.50 per month** for standard AWS endpoint health checks, and **\$0.75/month** for fast (10s) intervals `[Doc: Amazon Route 53 Pricing, checked 2026-09-03]`.
  - Non-AWS endpoints cost **\$1.00/month** (or \$2.00 for fast interval).
  - OCI provides health check probing included within standard enterprise tier quotas.

## 14. Failure Modes
- **The "Standby Collapse" (Thundering Herd Failover)**: A primary region handles 50,000 RPS. The secondary region runs a warm standby compute fleet sized for 5,000 RPS. When the primary fails, Route 53 shifts 50,000 RPS to the secondary region. The secondary compute fleet instantly crashes under 10x overload. *Mitigation: Auto-scale secondary compute ahead of time, or use Weighted Routing to shed non-essential traffic.*
- **The Deep Health Check Cascade**: A team configures the `/health` endpoint to query the primary database, secondary cache, and third-party CRM. When the third-party CRM degrades, the `/health` endpoint returns 500, causing Route 53 to fail over the entire application even though the core application was 100% healthy.

## 15. Troubleshooting
When traffic fails to failover after a primary outage:
1. **Query Authoritative Nameserver Directly**:
   ```bash
   dig @ns-1234.awsdns-50.org api.mycompany.com +nocache
   ```
   If the authoritative nameserver still returns the primary IP, inspect the health check console: has the consecutive failure threshold been met?
2. **Inspect Global Prober Status**: Check which specific regional probers are reporting healthy vs. unhealthy. If 12 of 16 probers report healthy, the endpoint remains marked healthy overall.
3. **Verify Client Resolver TTL Compliance**: Inspect client DNS cache. Some enterprise resolvers ignore TTLs under 300 seconds.

## 16. Common Mistakes
- **Setting DNS TTL to 0**: Believing that setting TTL to 0 forces instant failover. Many recursive resolvers (ISPs) reject TTL 0 and clamp it to 300 seconds, while legitimate clients flood authoritative nameservers with redundant queries, driving up DNS bills. Use a realistic TTL of **30 to 60 seconds**.
- **Failing to Configure Health Checks on Secondary Endpoints**: Failing over to a secondary region whose own load balancer has been offline or misconfigured, turning a partial outage into a complete blackout.

## 17. Trade-offs
| Routing Policy | Primary Goal | Trade-off / Risk | Best Use Case |
| :--- | :--- | :--- | :--- |
| **Weighted** | Canary deployments & gradual migration | Manual percentage management | Blue/Green releases |
| **Latency-Based (LBR)**| Minimum round-trip time for users | Traffic can shift unexpectedly due to global fiber routing | Global active-active APIs |
| **Geolocation** | Strict data residency & legal compliance | Users roaming abroad are locked to origin country | GDPR, sovereign apps |
| **Failover (Active-Passive)**| Clean disaster recovery with zero split-brain | Requires warm standby capacity; failover takes 60–120s | Tier-1 mission-critical databases |

## 18. Interview Questions
1. *You need to design a disaster recovery failover system for a global banking API with an RTO < 60 seconds. Can DNS health-check failover satisfy this requirement? Explain why or why not.*
2. *How does Latency-Based Routing (LBR) determine the closest region for a user, and what happens when an entire AWS or OCI region goes dark?*
3. *What is the difference between Geolocation Routing and Geoproximity Routing in Route 53?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "Achieving an RTO of strictly $< 60$ seconds using purely **DNS-based health-check failover** is extremely high-risk and generally cannot be guaranteed due to the decentralized nature of DNS caching:
>
> 1. **The Mathematical Timeline**:
>    - **Detection Time**: Even with fast health checks (10-second probing intervals) and a conservative failure threshold of 3 consecutive drops, detection takes **30 seconds** ($10\text{s} \times 3$).
>    - **DNS Propagation Time**: If we configure a DNS TTL of **30 seconds**, the theoretical failover time is $30\text{s (detection)} + 30\text{s (TTL)} = \mathbf{60\text{ seconds}}$.
> 2. **The Real-World DNS Caching Problem**:
>    - In real-world production, dozens of mobile telecom networks, corporate proxy resolvers, and public ISPs **deliberately violate DNS RFC standards by overriding low TTLs**. Many resolvers cache DNS records for a minimum of 300 seconds (5 minutes) regardless of what the authoritative nameserver dictates.
>    - Therefore, while 80% of traffic might fail over in 60 seconds, 20% of global clients will remain trapped attempting to reach the dead primary region for 5 to 15 minutes.
> 3. **The Staff-Level Architectural Solution**:
>    - To guarantee a true $< 60$-second RTO, do not rely on DNS manipulation at the edge.
>    - Instead, deploy **Anycast Static IP Routing** using **AWS Global Accelerator** or **OCI Load Balancers with Regional Anycast**.
>    - Under Anycast, clients communicate with static Anycast IP addresses that never change. Global traffic steering occurs at the BGP and edge proxy routing layer within the cloud provider's backbone.
>    - When a region fails, edge proxies reroute TCP connections to the secondary region within **10 to 20 seconds**, completely eliminating client-side DNS caching delays."

## 20. Hands-on Exercise
**Objective**: Deploy an automated Route 53 or OCI failover steering policy and measure exact failover propagation delay.

### Verification Steps
1. Provision two web servers in different regions (Primary in Region A, Secondary in Region B).
2. Configure an active health check probing `/health` on Primary every 10 seconds.
3. Configure Route 53 Failover record (or OCI Steering Policy) with `TTL = 10`.
4. In a terminal loop, query the domain every 2 seconds:
   ```bash
   while true; do dig api.mycompany.com +short; sleep 2; done
   ```
5. Terminate the web process on Primary.
6. Measure the exact elapsed time from process termination until the terminal query outputs the Secondary IP address.
