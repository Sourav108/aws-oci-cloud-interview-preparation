# Module 06: DNS & Service Discovery

> **Architectural Objective**: *Master global name resolution, intelligent traffic steering, and dynamic microservice discovery. Deconstruct the mechanics of Amazon Route 53 and OCI DNS, evaluate advanced traffic management policies (Latency, Geolocation, Failover steering), and engineer resilient internal service discovery architectures across AWS and OCI.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. Route 53 & OCI DNS Architecture](01-route53-and-oci-dns-architecture.md)** | Authoritative Anycast DNS, Public vs. Private Zones, Split-Horizon DNS, Record Types (ALIAS vs. CNAME) | Full 20-Section Deep Dive (~2,200 words) |
| **[02. Traffic Management Steering & Health Checks](02-traffic-management-steering-and-health-checks.md)** | Route 53 Routing Policies vs. OCI Steering Policies (Weighted, Latency, Geolocation, Failover), Active Health Checks | Full 20-Section Deep Dive (~2,200 words) |
| **[03. Cloud Service Discovery](03-cloud-service-discovery.md)** | AWS Cloud Map, Kubernetes CoreDNS, HashiCorp Consul, Dynamic Registration, Client DNS Caching & JVM TTL Pitfalls | Abbreviated Service Discovery Guide (~900 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **ALIAS Records vs. Standard CNAME**: Why the DNS RFC prohibits CNAME records at the zone apex (`example.com`), how Route 53 ALIAS and OCI ALIAS records solve this at zero DNS query cost, and why CNAME introduces extra client round-trips.
2. **Global Traffic Management Steering**: How to design multi-region active-active or active-passive DNS steering policies, handle DNS caching delays (TTL propagation), and prevent thundering-herd failover loops.
3. **Internal Microservices Discovery**: When to use DNS-based service discovery (AWS Cloud Map / CoreDNS) versus client-side load balancing (gRPC / Envoy), and how to tune DNS TTLs to prevent routing traffic to terminated container pods.
