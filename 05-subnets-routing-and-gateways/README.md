# Module 05: Subnets, Routing & Cloud Gateways

> **Architectural Objective**: *Master subnet tiering, cloud routing engines, and egress gateway mechanics. Deconstruct the architectural design of 3-tier subnets (Public Ingress, Private Application, Isolated Database), evaluate outbound NAT and private endpoint mechanisms (AWS VPC Endpoints vs. OCI Service Gateway), and master the systematic forensic troubleshooting of private network connectivity.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. Public, Private & Isolated Subnets](01-public-private-and-isolated-subnets.md)** | 3-Tier Subnet Segmentation, Route Table Precedence (Longest Prefix Match), Default Routes (`0.0.0.0/0`) | Full 20-Section Deep Dive (~2,000 words) |
| **[02. NAT Gateways, Service Gateways & VPC Endpoints](02-nat-gateways-service-gateways-and-vpc-endpoints.md)** | AWS NAT Gateway vs. OCI NAT Gateway, Gateway vs. Interface Endpoints (PrivateLink), OCI Service Gateway Economics | Full 20-Section Deep Dive (~2,200 words) |
| **[03. Private Connectivity Troubleshooting](03-private-connectivity-troubleshooting.md)** | Diagnostic Walkthrough: "App reaches DB but not internet", Asymmetric Routing, Blackhole Route Detection | Abbreviated Forensic Guide (~900 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **The 3-Tier Subnet Topology**: Why databases belong in **Isolated Subnets** with zero default routes, and how route tables enforce physical network segregation.
2. **Gateway Economics & NAT Elimination**: How routing S3 and DynamoDB traffic through free VPC Gateway Endpoints (AWS) or routing all Oracle services through the OCI Service Gateway eliminates thousands of dollars in monthly NAT processing charges.
3. **Forensic Egress Troubleshooting**: The exact 6-step diagnostic protocol to triage when an application server in a private subnet can query an internal database but hangs on external SaaS calls.
