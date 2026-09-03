# Contributing & Engineering Standards Guide

Thank you for contributing to the **AWS & OCI Cloud Engineering Interview Preparation** repository.

This repository is designed as a rigorous, senior-to-staff level architectural and implementation curriculum. To maintain absolute technical depth, factual grounding, and editorial integrity, all contributions must adhere strictly to the engineering rules detailed below.

---

## 🏛️ Core Architectural Principles

1. **Dual-Cloud Bilingualism (AWS + OCI)**:
   - Every module, lesson, and system design project must provide complete, production-grade coverage for both **AWS** and **OCI**.
   - **Depth Ratio Rule**: OCI coverage must address the exact same set of architectural sub-topics as AWS, with OCI word count never falling below **40%** of AWS word count.
   - Never claim two cloud services are identical without examining their specific semantic, structural, and network differences.
2. **Implementation-First Pedagogy**:
   - Focus on how cloud infrastructure is engineered, deployed, secured, operated, and recovered in production.
   - Avoid generic certification trivia. Ground all discussions in real architectural trade-offs, failure modes, packet flow, and telemetry.

---

## 🏷️ Grounding & Citation Policy (§5)

Any concrete assertion that could change across cloud releases, regions, or time—such as service quotas, default timeouts, pricing metrics, CLI flags, or IAM action names—must carry an explicit inline tag:

- `[Doc: <service/tool>, checked <YYYY-MM-DD>]` — Directly looked up and validated in official vendor documentation during the current session.
- `[Inference]` — Deduced from invariant architectural and distributed systems principles (e.g., CAP theorem, TCP handshake).
- `[Approximation — verify before relying on this]` — Educated baseline when direct documentation lookup is unavailable.

---

## ✍️ Content Originality & Anti-Plagiarism (§6)

- **Vendor Documentation is a Source of Facts, Not Prose**: Never copy-paste or closely paraphrase vendor documentation, whitepapers, or marketing copy. Explain concepts in your own technical voice and system architecture framing.
- **Short Exact Strings Only**: Quotations from vendor sources are strictly restricted to exact CLI flags, error messages, or IAM action strings where exactness is technically necessary.
- **Original Diagrams**: Render all architectural flows using original ASCII or Mermaid diagrams. Do not reproduce copyrighted vendor diagrams.

---

## 📐 Lesson Structure & Depth Calibration (§9)

### 1. Major Architectural Topics (Target: 1,500 – 3,000 words)
Major foundational domains (e.g., VPC/VCN, EKS/OKE, IAM, HA/DR, Managed Databases) must follow the full 20-section template:
1. Problem
2. Cloud Concept
3. Mental Model
4. Architecture Diagram
5. AWS Implementation
6. OCI Implementation
7. Configuration
8. Data Flow
9. Security
10. Reliability
11. Scaling
12. Observability
13. Cost
14. Failure Modes
15. Troubleshooting
16. Common Mistakes
17. Trade-offs
18. Interview Questions
19. Interview Answer
20. Hands-on Exercise

### 2. Supporting Topics (Target: 300 – 800 words)
Targeted components (e.g., specific storage classes, routing options, auxiliary gateways) follow an abbreviated 6-to-8 section format:
- Problem, Concept, AWS, OCI, Trade-offs, Failure Mode, and Senior Interview Question.

---

## 🔒 Cost Gates & Terraform Safety Standard

- Default all Terraform configurations to free-tier or sub-$1/hour resource specifications.
- Never run `terraform apply` against live cloud accounts automatically.
- Ensure every lab has a tested `terraform destroy` cleanup block.
- Support **No-Live-Account Fallback Mode** with static validation and detailed expected output narration.

---

## 🚀 Commit Message Standards

Follow Conventional Commits matching the repository phase table:
- `chore: initialize AWS OCI cloud interview curriculum`
- `feat: add cloud foundations`
- `feat: add cloud networking`
- `feat: add cloud compute and storage`
- `feat: add cloud data and messaging`
- `feat: add cloud security and observability`
- `feat: add scaling and reliability`
- `feat: add migration, iac, cicd`
- `feat: add cloud optimization`
- `feat: add interview questions <category set>`
- `feat: add cloud system design and projects`
- `feat: add cloud labs`
