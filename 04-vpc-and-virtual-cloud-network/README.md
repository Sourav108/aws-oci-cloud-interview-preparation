# Module 04: VPC & Virtual Cloud Network (VCN)

> **Architectural Objective**: *Master the design and operational implementation of software-defined isolated cloud networks. Deconstruct the architectural differences between AWS Amazon VPC and OCI Virtual Cloud Network (VCN), evaluate network firewall isolation models (Security Groups vs. NSGs vs. Security Lists), and engineer scalable cross-network transit topologies (Transit Gateway vs. DRG v2).*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. VPC & VCN Architecture](01-vpc-and-vcn-architecture.md)** | Software-Defined Networking, Overlay Encapsulation (Geneve vs. VXLAN), Zonal vs. Regional Subnet Scopes, CIDRs | Full 20-Section Deep Dive (~2,200 words) |
| **[02. Security Groups, NACLs, NSGs & Security Lists](02-security-groups-nacls-nsgs-and-security-lists.md)** | Stateful vs. Stateless Firewalls, Connection Tracking Tables (`conntrack`), VNIC vs. Subnet Boundaries | Full 20-Section Deep Dive (~2,400 words) |
| **[03. Peering, Transit Gateway & DRG](03-peering-transit-gateway-and-drg.md)** | Non-Transitive VPC Peering, AWS Transit Gateway Hub-and-Spoke, OCI Dynamic Routing Gateway (DRG v2) | Full 20-Section Deep Dive (~2,200 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **Subnet Scope Architecture**: Why AWS enforces strictly Zonal subnets while OCI provisions Regional subnets by default, and how this impacts multi-AZ load balancing and routing table management.
2. **The Cloud Firewall Security Matrix**: Why you must never equate an AWS Security Group with an OCI Security List, how OCI Network Security Groups (NSGs) decouple security policies from network topology, and the operational risks of stateful connection table exhaustion.
3. **Transit Mesh Engineering**: How to design a hub-and-spoke transit network connecting 50+ VPCs/VCNs across multiple accounts/tenancies, preventing route table bloat and managing inter-VPC encryption.
