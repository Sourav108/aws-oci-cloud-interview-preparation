# Module 02: Regions, AZs, ADs & Global Infrastructure

> **Architectural Objective**: *Master the physical and logical topology of hyperscale cloud infrastructure. Deconstruct how physical distance, optical fiber propagation, power grids, and fault domains dictate latency, high availability boundaries, distributed consensus limits, data residency compliance, and disaster recovery architectures across AWS and OCI.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. AWS Regions, AZs & Global Infrastructure](01-aws-regions-azs-and-global-infrastructure.md)** | AWS Regions, Physical AZs vs. Logical AZ IDs, Edge Locations, Local Zones, Optical Interconnects | Full 20-Section Deep Dive (~2,200 words) |
| **[02. OCI Regions, Availability Domains & Fault Domains](02-oci-regions-availability-domains-and-fault-domains.md)** | OCI Realms, Single-AD vs. 3-AD Regions, 3 Fault Domains per AD, Anti-Affinity Mechanics | Full 20-Section Deep Dive (~2,000 words) |
| **[03. HA, Latency & Data Residency Architecture](03-high-availability-latency-and-data-residency-architecture.md)** | The Physical Hierarchy ($\text{Region} \to \text{AZ/AD} \to \text{FD} \to \text{Rack}$), Latency Budgets, Multi-Region DR Topologies, Sovereign Clouds | Full 20-Section Deep Dive (~1,800 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **AZ Mapping & Physical Hotspotting**: Why `us-east-1a` in Account A is physically different from `us-east-1a` in Account B, and how to use AZ IDs (`use1-az1`) to coordinate low-latency inter-account clustering.
2. **OCI Fault Domains vs. AWS AZs**: How OCI guarantees hardware anti-affinity within single-AD regions, and why a single-AD OCI region provides rack-level resilience that single-AZ AWS deployments lack.
3. **Consensus Across Failure Domains**: How light propagation in optical fiber ($\approx 5\mu\text{s/km}$) imposes hard latency boundaries on synchronous replication and Paxos/Raft quorums across AZs and regions.
4. **Data Sovereignty vs. High Availability**: How to design disaster recovery architectures that comply with GDPR/HIPAA data residency boundaries without compromising 99.99% availability.
