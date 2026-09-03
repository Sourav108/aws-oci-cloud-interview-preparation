# Module 03: Networking Fundamentals & Packet Flow

> **Architectural Objective**: *Master the foundational mechanics of computer networking through the lens of hyperscale cloud environments. Deconstruct IPv4/IPv6 CIDR math, transport and application-layer protocols, socket buffers, TLS handshakes, MTU limits, and the complete deterministic lifecycle of network packets traversing cloud boundaries.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. IPv4, IPv6, CIDR & Subnet Math](01-ipv4-ipv6-cidr-and-subnet-math.md)** | Subnet Sizing, CIDR Bitmasking, Reserved IP Allocations (AWS 5 vs. OCI 3), Non-Overlapping Planning | Full 20-Section Deep Dive (~2,000 words) |
| **[02. TCP, UDP, TLS & OSI in Cloud Systems](02-tcp-udp-tls-and-osi-in-cloud-networking.md)** | L4 vs. L7 Networking, TCP 3-Way Handshake, TIME_WAIT, MTU (1500 vs. 9001), TLS 1.3 Handshake & ALPN | Full 20-Section Deep Dive (~2,200 words) |
| **[03. Packet Lifecycle & Cloud Traffic Flow](03-packet-lifecycle-and-cloud-traffic-flow.md)** | Complete Ingress/Egress Packet Walkthrough, SNAT vs. DNAT, Proxy Protocol, Private Egress Debugging | Full 20-Section Deep Dive (~2,500 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **Subnet Math & Cloud IP Reservation**: Exactly which IP addresses are reserved in an AWS `/24` subnet (5 reserved IPs) versus an OCI `/24` subnet (3 reserved IPs), and why planning CIDR blocks without overlapping is critical for enterprise VPC peering and Transit Gateways.
2. **Layer 4 vs. Layer 7 Routing Mechanics**: Why an ALB/OCI Load Balancer (L7) introduces higher CPU and latency overhead than an NLB (L4), how connection multiplexing works, and when pass-through routing is mandatory.
3. **The End-to-End Packet Trace**: Tracing every hop, header modification (SNAT/DNAT, MAC re-writing, X-Forwarded-For injection), and routing lookup of an HTTP packet from an internet browser down to a container pod.
4. **The Broken Egress Troubleshooting Scenario**: Step-by-step forensic diagnosis when an internal microservice can successfully communicate with a private PostgreSQL database but hangs indefinitely when calling an external SaaS API.
