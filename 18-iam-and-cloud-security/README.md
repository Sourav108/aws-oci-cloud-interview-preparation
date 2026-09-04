# Module 18: IAM, Compartments & Workload Identity

> **Architectural Objective**: *Master cloud identity architectures, fine-grained access control, resource hierarchy isolation, and machine workload identity federation. Deconstruct AWS IAM against OCI Identity and Access Management (OCI IAM Domains & Compartments), evaluate policy evaluation logic engines (AWS Explicit Deny vs. OCI Verb Hierarchies), and master zero-trust federation across multi-account, cross-tenancy, and Kubernetes environments.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. IAM vs. OCI Identity Domains & Compartments](01-iam-vs-oci-identity-domains-and-compartments.md)** | AWS IAM vs. OCI IAM Domains, OCI Compartment Hierarchies (Nested 6 levels) & Policy Inheritance, IAM Evaluation Logic (Explicit Deny vs. OCI Verbs: Inspect/Read/Use/Manage), Network Sources | Full 20-Section Deep Dive (~2,500 words) |
| **[02. Workload Identity & Enterprise Federation](02-workload-identity-and-federation.md)** | Machine Identity (IRSA / EKS Pod Identity vs. OCI Dynamic Groups & Workload Identity), Cross-Account STS AssumeRole (Confused Deputy & External ID) vs. Cross-Tenancy Policies (Endorse/Admit), OIDC & SAML 2.0 | Full 20-Section Deep Dive (~2,400 words) |
| **[03. IAM Least Privilege & Access Governance](03-iam-least-privilege-and-governance.md)** | AWS IAM Access Analyzer & CloudTrail Mining vs. OCI Policy Advisor, AWS Permission Boundaries vs. OCI Compartment Quotas, Privilege Escalation Vector Audits | Abbreviated Governance Guide (~950 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **The Policy Evaluation Engine**: Exactly how AWS IAM evaluates requests across SCPs, Resource Policies, Identity Policies, and Permissions Boundaries, compared to OCI's declarative verb hierarchy (`inspect` $\to$ `read` $\to$ `use` $\to$ `manage`).
2. **Resource Isolation Architectures**: When to isolate enterprise workloads across separate AWS Accounts in an AWS Organization versus nested OCI Compartments within a single tenancy.
3. **Machine Identity Federation & Confused Deputy Mitigation**: How to architect least-privilege short-lived STS tokens for container pods and third-party SaaS integrations while mathematically eliminating the confused deputy vulnerability using cryptographically enforced External IDs.
