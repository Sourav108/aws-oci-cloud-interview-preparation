# Module 07: Load Balancing & Traffic Ingress

> **Architectural Objective**: *Master cloud traffic ingress, reverse proxying, and connection scheduling. Deconstruct Layer 7 Application Load Balancers (AWS ALB vs. OCI Load Balancer) and Layer 4 Network Load Balancers (AWS NLB vs. OCI NLB), evaluate flexible bandwidth shaping against dynamic DNS scaling, and master backend pool health management and connection lifecycle controls.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. ALB vs. OCI Load Balancer (Layer 7)](01-alb-vs-oci-load-balancer-layer7.md)** | HTTP/2, gRPC, Path/Host Routing, TLS Offloading, OCI Flexible Bandwidth Shaping vs. AWS ALB DNS Scaling | Full 20-Section Deep Dive (~2,400 words) |
| **[02. NLB vs. OCI Network Load Balancer (Layer 4)](02-nlb-vs-oci-network-load-balancer-layer4.md)** | Sub-Millisecond Pass-Through Routing, Static Anycast IPs, Proxy Protocol v2, Ultra-High Spike Resistance | Full 20-Section Deep Dive (~2,200 words) |
| **[03. Health Checks, Connection Draining & Session Affinity](03-health-checks-connection-draining-and-sticky-sessions.md)** | Target Groups vs. Backend Sets, Liveness vs. Readiness Probes, Deregistration Delay, Sticky Sessions Trade-offs | Full 20-Section Deep Dive (~2,000 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **L4 vs. L7 Trade-off Matrix**: Why an L4 NLB delivers 10x higher throughput at sub-millisecond latency compared to an L7 ALB, and when application requirements (gRPC, path routing, WAF inspection) mandate an L7 proxy.
2. **Static IPs vs. DNS Hostnames**: Why OCI Load Balancers provide dedicated regional static IP addresses out-of-the-box while AWS ALBs expose dynamic DNS names, and how this impacts corporate firewall whitelisting.
3. **Graceful Connection Draining**: How to configure Deregistration Delay to achieve zero-downtime rolling deployments and prevent truncated HTTP responses during autoscaling scale-in events.
