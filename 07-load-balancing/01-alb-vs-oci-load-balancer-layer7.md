# 01. ALB vs. OCI Load Balancer (Layer 7)

## 1. Problem
Modern microservice architectures decompose monolithic backends into dozens of specialized services (e.g., `/orders`, `/users`, `/payments`). Exposing each service on a separate public IP or port wastes precious IPv4 space, complicates SSL/TLS certificate management, and creates a sprawling, vulnerable public attack surface. Layer 7 Application Load Balancers solve this by providing a unified public reverse-proxy ingress that inspects HTTP headers, path strings, and query parameters to route traffic dynamically to private backend fleets. However, when architects misunderstand how cloud L7 load balancers scale, terminate TLS, or manage IP addresses across AWS and OCI, they create brittle, expensive ingress choke points.

## 2. Cloud Concept
A Layer 7 Load Balancer is a software-defined reverse proxy operating at the **Application Layer (OSI Layer 7)**:
- **Two Independent TCP Connections**:
  $$\text{Client} \xleftrightarrow{\quad\text{TCP Connection 1 (TLS Terminated)}\quad} \text{L7 Load Balancer} \xleftrightarrow{\quad\text{TCP Connection 2 (Reused Socket)}\quad} \text{Backend Target}$$
  The client never talks directly to the backend container. The load balancer terminates the client's TCP and TLS connection, parses the complete HTTP request, evaluates routing rules, and forwards the payload over a pre-warmed internal persistent TCP connection pool.
- **Deep Content-Based Routing**:
  - *Path Routing*: `api.company.com/v1/orders` $\longrightarrow$ Target Group Orders.
  - *Host Routing*: `admin.company.com` $\longrightarrow$ Target Group Admin.
  - *HTTP Header / Method Routing*: Route requests containing `X-Canary: true` to the Canary release fleet.
  - *Redirects & Fixed Responses*: Return instant HTTP 301 redirects (`http -> https`) or HTTP 403/503 responses directly from the load balancer without touching backend compute.

### Dynamic DNS Scaling vs. Flexible Bandwidth Shaping
- **AWS Application Load Balancer (ALB)**: Uses **Dynamic DNS Scaling**. The ALB provisions a dynamic fleet of internal proxy instances behind an AWS-managed DNS hostname. As traffic ramps up, AWS automatically launches more internal proxy instances and publishes new IP addresses in DNS.
  - *Crucial Consequence*: **An AWS ALB never provides static IP addresses**. External firewalls cannot whitelist an ALB by IP address.
- **OCI Load Balancer**: Uses **Flexible Bandwidth Shaping**. When you provision an OCI Load Balancer, you define explicit minimum and maximum bandwidth bounds (e.g., Min: 50 Mbps, Max: 1,000 Mbps) `[Doc: OCI Flexible Load Balancers, checked 2026-09-03]`.
  - *Crucial Consequence*: **An OCI Load Balancer provides dedicated, static regional IP addresses** out-of-the-box, making it trivial for enterprise B2B partners to whitelist ingress IPs.

## 3. Mental Model
Think of a Layer 7 Load Balancer as the head concierge at a grand hotel:
- You walk up to the desk and speak to the concierge in person (Client-to-ALB connection).
- The concierge listens to your request (reads the HTTP path and headers).
- If you ask for dinner reservations (`/dining`), the concierge calls the restaurant on their private internal telephone line (ALB-to-Backend connection) and books your table.
- If you ask to speak with the general manager without an appointment, the concierge rejects you on the spot (fixed-response HTTP 403) without bothering the manager.

## 4. Architecture Diagram
```text
┌────────────────────────────────────────────────────────────────────────┐
│                   LAYER 7 APPLICATION ROUTING ENGINE                   │
│                                                                        │
│   Public Client Request: https://api.corp.com/v1/checkout             │
│   (TLS 1.3 / HTTP/2 Negotiated at Cloud Edge)                          │
│                                │                                       │
│                                ▼                                       │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ CLOUD L7 LOAD BALANCER LISTENER (Port 443)                     │   │
│   │ AWS: ALB (Managed DNS Name) | OCI: Flexible LB (Static IPs)    │   │
│   ├────────────────────────────────────────────────────────────────┤   │
│   │ * TLS Offloaded via Cloud ACM / OCI Certificates               │   │
│   │ * Injects X-Forwarded-For: 203.0.113.50, X-Forwarded-Proto: https│   │
│   │ * Rule 1: Host: api.corp.com & Path: /v1/checkout*             │   │
│   │ * Rule 2: Host: api.corp.com & Path: /v1/users*                │   │
│   └───────────────┬────────────────────────────────┬───────────────┘   │
│                   │                                │                   │
│                   │ Path matches /v1/checkout      │ Matches /v1/users │
│                   ▼                                ▼                   │
│   ┌───────────────────────────────┐ ┌──────────────────────────────┐   │
│   │ TARGET GROUP: CHECKOUT PODS   │ │ TARGET GROUP: USER PODS      │   │
│   │ [Pod 10.0.1.20] [Pod 10.0.1.21│ │ [Pod 10.0.2.30] [Pod 10.0.2.31│   │
│   │ (Private App Subnet)          │ │ (Private App Subnet)         │   │
│   └───────────────────────────────┘ └──────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS:
- **Application Load Balancer (ALB)**:
  - Supports HTTP/1.1, HTTP/2, gRPC, and WebSockets.
  - Native integration with **AWS WAF**: Inspects payload bodies, enforces rate-limiting rules, and blocks SQLi/XSS attacks at the load balancer layer before packets touch EC2/EKS.
  - Native **OIDC / OAuth2 / Amazon Cognito Authentication**: ALB can authenticate users against Okta, Auth0, or Google before forwarding requests to the application.
  - **Rule Processing Limit**: Supports up to 100 rules per ALB listener by default `[Doc: Application Load Balancer Quotas, checked 2026-09-03]`. Evaluated in strict priority order.
  - **Target Types**: `instance` (EC2 instance ID), `ip` (Pod IP or on-premises IP over Direct Connect), `lambda` (serverless function invocation).

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **OCI Load Balancer (Layer 7)**:
  - Native enterprise reverse proxy supporting HTTP/1.1, HTTP/2, gRPC, and WebSockets.
  - **Flexible Bandwidth Sizing**: Instead of opaque auto-scaling, OCI allows customers to specify exact bandwidth ranges:
    - Minimum bandwidth: 10 Mbps (guaranteed pre-allocated capacity; zero scaling lag).
    - Maximum bandwidth: Up to 8,000 Mbps (8 Gbps) `[Doc: OCI Load Balancing Service, checked 2026-09-03]`.
    - Bandwidth can be dynamically updated via API in seconds without dropping active connections.
  - **Static Regional IP Addresses**: Every public OCI Load Balancer is assigned two dedicated static public IP addresses (primary and failover) spanning multiple Availability Domains / Fault Domains.
  - **Path Route Sets & Rule Sets**:
    - *Path Route Sets*: Direct traffic to distinct Backend Sets based on exact, prefix, or regex URL path matching.
    - *Rule Sets*: Manipulate request/response headers, rewrite URLs, enforce access control whitelists, and inject security headers (HSTS, CSP).
  - **OCI Web Application Firewall (WAF)**: Integrates directly with the Load Balancer, providing bot management and threat intelligence filtering.

## 7. Configuration
Comparing L7 load balancer provisioning in Terraform across AWS and OCI:

### AWS Application Load Balancer with Path Routing (Terraform)
```hcl
# AWS ALB Ingress
resource "aws_lb" "main" {
  name               = "prod-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [var.alb_security_group_id]
  subnets            = var.public_subnet_ids # Spans 3 AZs

  enable_deletion_protection = true
  tags = { Name = "prod-alb" }
}

# HTTPS Listener with ACM Certificate
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.acm_cert_arn

  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "application/json"
      message_body = "{\"error\":\"Route Not Found\"}"
      status_code  = "404"
    }
  }
}

# Path-based routing rule for Checkout service
resource "aws_lb_listener_rule" "checkout" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 10

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.checkout.arn
  }

  condition {
    path_pattern {
      values = ["/v1/checkout*"]
    }
  }
}
```

### OCI Flexible Load Balancer with Path Route Set (Terraform)
```hcl
# OCI Flexible Shape Load Balancer
resource "oci_load_balancer" "main" {
  compartment_id = var.compartment_id
  display_name   = "prod-oci-lb"
  shape          = "flexible"
  subnet_ids     = [var.public_subnet_id] # Regional subnet!

  shape_details {
    minimum_bandwidth_in_mbps = 50   # Pre-warmed baseline!
    maximum_bandwidth_in_mbps = 1000 # Upper scaling cap
  }

  is_private = false
}

# Path Route Set for Checkout Service
resource "oci_load_balancer_path_route_set" "path_routes" {
  load_balancer_id = oci_load_balancer.main.id
  name             = "api-path-route-set"

  path_routes {
    path = "/v1/checkout"
    match_type {
      match_type = "PREFIX_MATCH"
    }
    backend_set_name = oci_load_balancer_backend_set.checkout.name
  }
}
```

## 8. Data Flow
```text
L7 Request Routing and Header Transformation:
1. Client sends: GET /v1/checkout HTTP/2 to Public ALB / OCI LB
2. Load Balancer decrypts TLS, parses HTTP/2 stream
3. Header Processing:
   * Strips or sanitizes client headers.
   * Appends Client Public IP to: X-Forwarded-For: 203.0.113.50
   * Injects: X-Forwarded-Proto: https
   * Injects: X-Forwarded-Port: 443
4. Route Rule Evaluation:
   * Matches path rule: /v1/checkout*
   * Target selected from healthy pool using Round Robin / Least Outstanding Requests.
5. Internal Forwarding:
   * Transmits HTTP request over private VPC/VCN socket to Target (10.0.1.20:8080).
```

## 9. Security
- **Centralized TLS Offloading**: Terminating TLS at the load balancer centralizes certificate lifecycle management (via AWS ACM or OCI Certificates) with automated DNS-based renewal, offloading cryptographic overhead from backend compute.
- **Header Injection & Spoofing Defense**:
  - Malicious clients frequently forge `X-Forwarded-For` headers.
  - The ALB and OCI Load Balancer append the true connecting IP to the end of the `X-Forwarded-For` chain. Backend applications must inspect the **rightmost** (or trusted upstream) IP address in the header to prevent IP spoofing.

## 10. Reliability
- **Cross-Zone / Cross-AD Load Balancing**:
  - AWS ALB distributes traffic evenly across all registered targets in all enabled Availability Zones, regardless of which AZ received the initial connection (Cross-Zone Load Balancing is permanently enabled on ALBs).
  - OCI Load Balancer runs in active-standby redundancy across Availability Domains or Fault Domains, maintaining high availability with zero cross-AD network transfer fees.

## 11. Scaling
- **The AWS ALB Pre-Warming Constraint**:
  - AWS ALB scales out by dynamically adding proxy instances behind DNS.
  - *Failure Mode*: If traffic surges by 500% in 60 seconds (e.g., Super Bowl commercial, flash sale), the ALB cannot scale fast enough, returning HTTP 502/504 errors.
  - *Remediation*: Contact AWS Support to **pre-warm** the ALB ahead of known marketing events.
- **OCI Instant Elasticity**:
  - OCI's flexible shape allows architects to increase minimum provisioned bandwidth from 50 Mbps to 2,000 Mbps instantly via CLI/API before an event without opening support tickets.

## 12. Observability
- **ALB Access Logs**: Pushed directly to Amazon S3 in compressed gzip format. Essential for forensic analysis:
  ```text
  http 2026-09-03T12:00:00.000000Z app/prod-alb/1234 203.0.113.50:51234 10.0.1.20:8080 0.001 0.050 0.000 200 200 450 1200 "GET https://api.corp.com/v1/checkout HTTP/2.0"
  ```
- **The 5XX Metric Split**:
  - `HTTPCode_ELB_5XX_Count`: Generated by the ALB itself (e.g., 502 Bad Gateway due to target connection refusal or timeout).
  - `HTTPCode_Target_5XX_Count`: Generated by the backend application code (e.g., uncaught Java/Python exception).

## 13. Cost
- **AWS ALB Pricing**:
  - Hourly fee: **\$0.0225 per hour** ($\approx \$16.20/\text{month}$) `[Doc: AWS Elastic Load Balancing Pricing, checked 2026-09-03]`.
  - Usage fee: **\$0.008 per LCU-hour (Load Balancer Capacity Unit)**.
  - An LCU measures the highest dimension of: new connections (25/sec), active connections (3,000/min), processed bytes (1 GB/hr), or rule evaluations.
- **OCI Load Balancer Pricing**:
  - Billed based on provisioned bandwidth shape: approximately **\$0.0113 per hour** for base instances plus bandwidth consumption fees `[Doc: OCI Networking Pricing, checked 2026-09-03]`.

## 14. Failure Modes
- **The 504 Gateway Timeout Cascade**: The ALB's default idle timeout is 60 seconds. A backend database query freezes, causing backend application worker threads to block. When client requests exceed 60 seconds waiting for the backend to respond, the ALB abruptly terminates the connection and returns **HTTP 504 Gateway Timeout**. Upstream clients retry, triggering a catastrophic thundering herd.
- **Health Check Path CPU Saturation**: Configuring the ALB health check path to `/api/v1/health` where the endpoint executes complex database validation queries. When the cluster scales to 100 targets, health checks alone generate 500 queries/second against the database, causing a database CPU spike that takes down production.

## 15. Troubleshooting
Diagnosing HTTP 502 vs. HTTP 504 errors on an L7 Load Balancer:
1. **HTTP 502 Bad Gateway**:
   - The load balancer attempted to connect to the backend target, but the target refused the TCP connection (port closed or process crashed) or sent a malformed TCP RST.
   - *Fix*: Verify the backend process is running and listening on the configured target port.
2. **HTTP 504 Gateway Timeout**:
   - The load balancer connected to the target, but the target failed to return an HTTP response before the load balancer's idle timeout elapsed.
   - *Fix*: Profile slow backend database queries or increase the ALB idle timeout.

## 16. Common Mistakes
- **Assuming ALB Has Static IPs for Whitelisting**: Handing an enterprise customer the current IP addresses resolved from an ALB DNS name. When the ALB autoscales or an AZ degrades, those IPs disappear, breaking customer integrations. Use an **NLB with Elastic IPs** or an **OCI Load Balancer with Static IPs** if static ingress is required.
- **Failing to Enable Access Logging**: Operating production load balancers without S3 access logging enabled, leaving SREs with zero forensic request telemetry during security breaches.

## 17. Trade-offs
| Feature | AWS Application Load Balancer | OCI Load Balancer (Flexible) |
| :--- | :--- | :--- |
| **Ingress Addressing** | Dynamic DNS names only | Dedicated Static Public IP addresses |
| **Scaling Mechanism** | Automatic behind DNS (Subject to pre-warm limits) | Flexible bandwidth bounds (Pre-allocated minimum) |
| **Path/Host Routing** | Rich listener rules (Priority ordered) | Path Route Sets & Rule Sets |
| **WAF Integration** | Native AWS WAF association | Native OCI WAF association |
| **Authentication** | Built-in OIDC / Cognito / Okta auth | Handled downstream or via API Gateway |

## 18. Interview Questions
1. *In an enterprise B2B architecture, a banking partner demands a static public IPv4 address to whitelist in their corporate firewall. Can you give them an AWS ALB IP? How do you solve this in AWS vs. OCI?*
2. *Explain the architectural difference between an HTTP 502 Bad Gateway error and an HTTP 504 Gateway Timeout error on an Application Load Balancer. Where does the root cause lie in each case?*
3. *How does OCI's flexible bandwidth shape design eliminate the 'pre-warming' problem that plagues AWS ALBs during sudden traffic surges?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "You **cannot** provide a static public IPv4 address directly from an AWS Application Load Balancer (ALB).
>
> 1. **Why ALB Cannot Provide Static IPs**:
>    - AWS ALBs are designed around dynamic horizontal scaling. AWS provisions and scales internal proxy instances behind an AWS-managed DNS record (e.g., `app-12345.elb.amazonaws.com`).
>    - The IP addresses returned by this DNS name change dynamically as the load balancer scales or as underlying hypervisors are cycled. Whitelisting an ALB's current IP address will cause total connection failure as soon as AWS rotates proxy nodes.
>
> 2. **The AWS Solution (NLB in front of ALB)**:
>    - To satisfy the static IP requirement in AWS, we deploy an **AWS Network Load Balancer (NLB)** in front of the ALB.
>    - We assign a dedicated static **Elastic IP (EIP)** to the NLB in each Availability Zone.
>    - We configure the NLB to target the ALB directly (using the ALB's target group integration).
>    - The banking partner whitelists the NLB's static Elastic IPs in their firewall. The NLB passes traffic through to the ALB at Layer 4, and the ALB terminates TLS and handles Layer 7 path routing.
>    - Alternatively, we can deploy **AWS Global Accelerator**, which provides two global Anycast static IPs routed directly to the ALB.
>
> 3. **The OCI Solution (Native Static IPs)**:
>    - In OCI, this workaround is completely unnecessary.
>    - Every public **OCI Load Balancer is automatically provisioned with dedicated, static regional IPv4 addresses** by default.
>    - We provide these static public IPs directly to the banking partner with zero architectural workarounds or added costs."

## 20. Hands-on Exercise
**Objective**: Configure path-based routing in an L7 load balancer and observe traffic routing to distinct target groups.

### Verification Steps
1. Deploy an ALB/OCI LB with two target groups: Target Group Orders (`/orders`) and Target Group Users (`/users`).
2. Deploy a lightweight HTTP container into each target group returning its service name.
3. Execute: `curl https://<lb-endpoint>/orders` $\longrightarrow$ Verify response: `{"service":"orders"}`.
4. Execute: `curl https://<lb-endpoint>/users` $\longrightarrow$ Verify response: `{"service":"users"}`.
5. Execute: `curl https://<lb-endpoint>/unknown` $\longrightarrow$ Verify default 404 response returned directly by the load balancer without hitting backend compute.
