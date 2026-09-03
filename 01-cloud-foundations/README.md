# Module 01: Cloud Foundations & Shared Responsibility

> **Architectural Objective**: *Deconstruct cloud computing from first principles. Transition beyond superficial vendor marketing into rigorous systems engineering: service boundaries, failure domain ownership under the Shared Responsibility Model, mathematical models of availability and durability, elasticity dynamics, and the true unit economics of managed vs. self-managed infrastructure.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. Service Models & Shared Responsibility](01-cloud-service-models-and-shared-responsibility.md)** | IaaS vs. PaaS vs. SaaS, AWS vs. OCI Shared Responsibility Boundaries, Security Primitives | Full 20-Section Deep Dive (~2,200 words) |
| **[02. Elasticity, Durability & Availability Economics](02-elasticity-durability-and-availability-economics.md)** | Elasticity vs. Scalability, Multi-Nines Math, Serial vs. Parallel Systems, FinOps Unit Economics | Full 20-Section Deep Dive (~2,000 words) |
| **[03. Managed vs. Self-Managed Infrastructure](03-managed-vs-self-managed-tradeoffs.md)** | Operational Toil, Hidden Cloud Costs, Upgrades, Blast Radius, Vendor Lock-in Reality | Abbreviated Trade-off Analysis (~800 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **The Shared Responsibility Boundary**: Exactly where your team's operational liability ends and AWS/OCI's hypervisor, network, and hardware guarantees begin across VMs, containers, and serverless.
2. **Availability vs. Durability**: Why Amazon S3 / OCI Object Storage can promise 11 nines of durability ($99.999999999\%$) while offering only 3 to 4 nines of availability ($99.9\% \text{ to } 99.99\%$).
3. **Serial vs. Parallel Reliability Math**: How to calculate the aggregate availability of a microservices call graph and prevent systemic cascading failure.
4. **The Managed Service Trade-off Matrix**: How to defend choosing a managed service (e.g., RDS / Autonomous DB) vs. running self-managed state on VMs in a staff-level interview.
