# AWS & OCI Cloud Engineering Interview Preparation

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Curriculum: SDE2 to Staff](https://img.shields.io/badge/Curriculum-SDE2%20→%20Senior%20→%20Staff-orange.svg)](CURRICULUM.md)
[![Cloud Bilingual](https://img.shields.io/badge/Cloud-AWS%20%2B%20OCI%20Bilingual-green.svg)](AWS_OCI_SERVICE_MAP.md)
[![Interview Framework](https://img.shields.io/badge/Framework-R--A--T--D--S--R--S--O--C--F--T-purple.svg)](CLOUD_INTERVIEW_FRAMEWORK.md)

An advanced, implementation-first cloud architecture and infrastructure engineering interview curriculum. Designed specifically for experienced backend and systems engineers preparing for **SDE2, Senior, and Staff-level** Cloud Architect, Platform Engineer, DevOps, and SRE roles across **Amazon Web Services (AWS)** and **Oracle Cloud Infrastructure (OCI)**.

---

## 🧭 Core Architectural Philosophy

```text
Business Requirement ──> Architecture ──> Cloud Services ──> Networking ──> Security ──>
Compute ──> Storage ──> Database ──> Messaging ──> Observability ──> Reliability ──>
Scaling ──> Cost ──> Implementation ──> Production Operations
```

This repository does **not** teach entry-level cloud certification trivia or shallow definitions. It trains engineers to reason from first principles:
- *Why this specific cloud service over alternatives?*
- *How does the packet traverse internet gateways, load balancers, and private subnets?*
- *What happens under the hood during a primary database crash or regional fiber cut?*
- *How do AWS Security Groups (ENI-level) fundamentally differ from OCI Security Lists (subnet-level) and NSGs?*
- *How do you optimize cloud unit economics without silently compromising multi-AZ resilience or security guardrails?*

**Skill Progression Arc**:
$$\text{UNDERSTAND} \to \text{DESIGN} \to \text{IMPLEMENT} \to \text{DEPLOY} \to \text{SECURE} \to \text{OBSERVE} \to \text{SCALE} \to \text{DEBUG} \to \text{OPTIMIZE} \to \text{DEFEND TRADE-OFFS}$$

---

## 🌐 Dual-Cloud Strategy & Depth Calibration (AWS : OCI)

Rather than treating OCI as an afterthought, this curriculum enforces **rigorous cloud bilingualism**:

| Dimension | AWS Platform | OCI Platform | Architectural & Semantic Difference |
| :--- | :--- | :--- | :--- |
| **Compute** | EC2 Nitro (Fixed families: c6i, m6i, r6i) | OCI Compute (Flexible Shapes: `VM.Standard.E5.Flex`) | OCI decouples OCPU from Memory, allowing fine-grained RAM allocation without paying for unused cores. |
| **Containers** | ECS Fargate & EKS | Container Instances & OKE | OKE provides a free basic control plane; OCI Container Instances launch containers without orchestrator overhead. |
| **Virtual Network** | VPC (Regional; Subnets are strictly Zonal) | VCN (Regional; Subnets can be Regional or AD-specific) | OCI Regional Subnets span all Availability Domains by default, simplifying multi-AD routing. |
| **Firewalls** | Security Groups (stateful, ENI) & NACLs (stateless, subnet) | NSGs (stateful/stateless, VNIC) & Security Lists (stateful/stateless, subnet) | OCI NSGs decouple security policy from network CIDR blocks; Security Lists apply to the entire subnet. |
| **Service Endpoints**| VPC Endpoints (Gateway for S3/DynamoDB; Interface for others) | Service Gateway (`all-services-in-region`) | OCI routes to all regional Oracle services over the internal backbone with a single gateway and zero data fees. |
| **Managed Relational**| RDS & Aurora Multi-AZ Cluster (6-way quorum storage) | Base Database Service & Autonomous DB (Exadata, Data Guard) | Aurora decouples distributed storage; Autonomous DB automates index tuning, patching, and elastic scaling. |
| **Object Storage** | Amazon S3 (Strong consistency, lifecycle tiers) | OCI Object Storage (Strong consistency, native auto-tiering) | S3 uses bucket-level namespaces; OCI organizes buckets within Tenancy Namespaces and Compartments. |

*Depth Ratio Rule*: For every lesson and project, OCI covers the exact same architectural sub-topics as AWS, with OCI word count never falling below **40%** of AWS.

---

## 🎯 The 11-Step Cloud Interview Framework

When tackling Senior/Staff cloud system design questions, follow the **R-A-T-D-S-R-S-O-C-F-T** framework detailed in [CLOUD_INTERVIEW_FRAMEWORK.md](CLOUD_INTERVIEW_FRAMEWORK.md):

1. **R — Requirements & Constraints**: Scope functional/non-functional SLOs, QPS, data volume, and availability targets.
2. **A — High-Level Architecture**: Component topology, boundaries, and dual-cloud service mapping.
3. **T — Traffic Flow & Ingress Path**: Complete packet trace from DNS to TLS termination, reverse proxy, and subnet traversal.
4. **D — Data Flow & Storage Engine**: Relational vs. NoSQL access patterns, replication topologies, and consistency models.
5. **S — Security & Zero-Trust**: Network segmentation, credential-less workload identity, and KMS envelope encryption.
6. **R — Reliability & High Availability**: Multi-AZ/AD distribution, liveness/readiness health probes, and automated failover.
7. **S — Scaling & Capacity**: Bottleneck identification, horizontal autoscaling triggers, and connection pool sizing.
8. **O — Observability & Telemetry**: Golden signals, distributed tracing, structured logs, and SLO error budget alerts.
9. **C — Cost & Unit Economics**: FinOps right-sizing, baseline commitment models, and network egress minimization.
10. **F — Failure Modes & Incidents**: Simulating AZ blackholes, primary DB crashes, poison pills, and network partitions.
11. **T — Trade-off Defense**: Articulating compromises and defending architectural choices under interviewer cross-examination.

---

## 📚 Master Repository Structure

```text
aws-oci-cloud-interview-preparation/
├── README.md                          # Master Guide & Repository Overview
├── CURRICULUM.md                      # Detailed 30-Module Syllabus & Learning Arc
├── ROADMAP.md                         # 6-Week Structured Study & Practice Schedule
├── STATE.md                           # Session Continuity & Execution Tracker
├── CLOUD_VERSION_MATRIX.md            # Version Baseline & Grounding Reference Matrix
├── COVERAGE_MATRIX.md                 # Concept-by-Concept 11-Dimension Tracking Matrix
├── DEPENDENCY_GRAPH.md                # Architectural Prerequisite Tree & Flow Diagram
├── AWS_OCI_SERVICE_MAP.md             # Comprehensive AWS vs. OCI Capability Mapping
├── CLOUD_INTERVIEW_FRAMEWORK.md       # The 11-Step R-A-T-D-S-R-S-O-C-F-T Interview Method
├── CLOUD_DESIGN_PATTERNS.md           # 16 Production Cloud Design Patterns Catalog
├── CLOUD_ANTI_PATTERNS.md             # 15 Architectural Anti-Patterns & Remediations
├── CLOUD_SECURITY_CHECKLIST.md        # Zero-Trust & Defense-in-Depth Audit Framework
├── CLOUD_RELIABILITY_CHECKLIST.md     # Multi-AZ, Health Probes & Disaster Recovery Audit
├── CLOUD_COST_CHECKLIST.md            # FinOps Unit Economics & Cost Optimization Audit
├── LABS.md                            # Hands-On Lab Index & Cost Governance Standard
├── PRODUCTION_RUNBOOKS.md             # 16 Emergency Production Operational Runbooks
├── TROUBLESHOOTING.md                 # Scientific 8-Stage Triage & Root Cause Methodology
├── CONTRIBUTING.md                    # Editorial Rules, Originality Policy & Standards
├── LICENSE                            # MIT License
├── .gitignore                         # Production Git Ignore Specifications
│
├── 01-cloud-foundations/
├── 02-regions-azs-and-global-infrastructure/
├── 03-networking-fundamentals/
├── 04-vpc-and-virtual-cloud-network/
├── 05-subnets-routing-and-gateways/
├── 06-dns-and-service-discovery/
├── 07-load-balancing/
├── 08-compute-and-virtual-machines/
├── 09-containers/
├── 10-kubernetes/
├── 11-serverless/
├── 12-object-storage/
├── 13-block-and-file-storage/
├── 14-managed-databases/
├── 15-nosql-and-distributed-data/
├── 16-caching/
├── 17-messaging-and-eventing/
├── 18-iam-and-cloud-security/
├── 19-secrets-encryption-and-key-management/
├── 20-observability-monitoring-and-logging/
├── 21-autoscaling-and-capacity/
├── 22-high-availability-and-disaster-recovery/
├── 23-resilience-and-fault-tolerance/
├── 24-cloud-migrations/
├── 25-infrastructure-as-code/
├── 26-ci-cd-and-cloud-deployments/
├── 27-cloud-performance-optimization/
├── 28-cloud-cost-optimization/
├── 29-cloud-interview-questions/      # 500 Questions (Sub-phases 29.1 to 29.5)
├── 30-cloud-system-design-and-projects/
│
├── labs/                              # Hands-On Terraform & Architecture Labs
├── projects/                          # 8 Production End-to-End Reference Systems
└── templates/                         # Architecture, Runbook & Lesson Templates
```

---

## 🏷️ Grounding & Fact-Checking Standard (§5)

To maintain absolute factual accuracy across evolving cloud features and pricing, every time-sensitive technical parameter carries one of three explicit citation tags:

1. `[Doc: <service>, checked <date>]` — Verified directly in official documentation in the current session.
2. `[Inference]` — Deduced from stable, fundamental distributed systems principles.
3. `[Approximation — verify before relying on this]` — Baseline estimate when live documentation lookup is unavailable.

---

## 🔒 Hands-on Labs & Safe Cost Governance (§14)

All hands-on labs in `labs/` enforce hard financial safeguards:
- **Free-Tier / Sub-$1/Hour Default**: Optimized for minimal infrastructure expense.
- **Cost Warning Gate**: Any lab requiring billable infrastructure features an explicit boxed cost estimate before all command blocks.
- **No Automated Apply**: Infrastructure is never applied automatically to live cloud accounts without explicit operator consent.
- **No-Live-Account Fallback**: Complete Terraform code and architecture diagrams are provided alongside simulated plan outputs and log traces for zero-cost study.

---

## 🤝 Contributing & License

Contributions must strictly follow our [Contributing Guidelines](CONTRIBUTING.md) and [Content Originality Policy](CONTRIBUTING.md#content-originality--anti-plagiarism-6).

Licensed under the [MIT License](LICENSE).
