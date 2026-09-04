# Module 26: CI/CD, Deployment Pipelines & GitOps (AWS vs. OCI)

---

## 1. Module Overview & Learning Objectives

In enterprise cloud platform engineering, the release pipeline is the primary bridge connecting developer source code to production infrastructure. A broken, manual, or insecure deployment pipeline slows engineering velocity, introduces human error, causes production outages, and opens catastrophic vulnerabilities across the software supply chain. Modern cloud delivery mandates automated, zero-downtime deployment strategies, declarative GitOps reconciliation, and cryptographic software provenance.

This module delivers comprehensive technical and architectural mastery over advanced deployment strategies (Blue/Green, Canary, Rolling), GitOps delivery with ArgoCD across AWS EKS and OCI OKE, native cloud pipeline services (AWS CodePipeline vs. OCI DevOps), and end-to-end Software Supply Chain Security.

### What You Will Master
1. **Zero-Downtime Deployment Strategies**: Architectural implementation of Rolling Updates, Blue/Green (Red/Black) deployments, and Progressive Canary releases with Automated Canary Analysis (ACA).
2. **Database Migration Coordination**: Operating the Expand-Contract (Parallel Run) schema migration pattern during asynchronous multi-version rollouts.
3. **GitOps & Declarative Delivery**: Managing Kubernetes workloads on EKS and OKE with ArgoCD, ApplicationSets, Sync Waves, and self-healing reconciliation loops.
4. **Cloud-Native CI/CD Pipelines**: Comparing AWS Developer Tools (CodePipeline, CodeBuild, CodeDeploy) against OCI DevOps Service (Build Pipelines, Deployment Pipelines, Artifact Repositories).
5. **GitOps Secret Management**: Operating External Secrets Operator (ESO) integrated with AWS Secrets Manager and OCI Vault KMS.
6. **Software Supply Chain Security & SBOM**: Enforcing SLSA frameworks, Software Bill of Materials (SBOM via Syft/Trivy), container image signing via Cosign / AWS Signer, and OKE admission control verification.

---

## 2. Directory Roadmap & Lesson Catalog

```
26-ci-cd-and-cloud-deployments/
├── README.md                                                  # Module guide & architectural index
├── 01-deployment-strategies-blue-green-canary-rolling.md     # [Major] Blue/Green, Canary, Rolling, Expand-Contract schema
├── 02-gitops-and-continuous-delivery-argocd-pipelines.md     # [Major] ArgoCD on EKS/OKE, AWS CodePipeline vs OCI DevOps, ESO
└── 03-pipeline-security-supply-chain-and-sbom.md             # [Supporting] Supply chain security, SLSA, SBOM, Cosign/AWS Signer
```

---

## 3. The Deployment Strategy Spectrum: Trade-Off Matrix

```
Downtime / Risk
   ^
   |  [ Recreate / In-Place ] (Downtime: Minutes/Hours, Cost: 1x, Rollback: Slow)
   |          \
   |           [ Rolling Update ] (Downtime: Zero, Cost: 1.25x, Skew: High, Rollback: Moderate)
   |                     \
   |                      [ Blue / Green ] (Downtime: Zero, Cost: 2x, Skew: Zero, Rollback: Instant)
   |                                \
   |                                 [ Canary Deployment ] (Downtime: Zero, Cost: 1.1x, Risk: Minimized)
   +----------------------------------------------------------------------------------------------------> Engineering Complexity
```

---

## 4. Side-by-Side Dual-Cloud CI/CD Primitives

| CI/CD & Delivery Domain | AWS Ecosystem Primitives | OCI Ecosystem Primitives |
| :--- | :--- | :--- |
| **Managed Pipeline Service**| AWS CodePipeline | **OCI DevOps Service** (Build & Deploy Pipelines) |
| **Managed Build Runner** | AWS CodeBuild | OCI DevOps Build Runner (Managed compute) |
| **Deployment Engine** | AWS CodeDeploy (Canary, Linear, Blue/Green) | OCI DevOps Deployment Pipelines |
| **Artifact Repository** | AWS CodeArtifact / Amazon ECR | OCI Artifact Registry / OCI Container Registry (OCIR) |
| **GitOps Engine** | ArgoCD / Flux on Amazon EKS | ArgoCD / Flux on OCI Container Engine (OKE) |
| **Traffic Shifting Primitive**| ALB Weighted Target Groups / Route 53 Weighted | OCI Load Balancer Weighted Backend Sets |
| **Container Image Signing** | AWS Signer for Amazon ECR | OCI Container Image Signing & KMS Verification |
| **Secret Synchronization** | External Secrets Operator -> AWS Secrets Manager| External Secrets Operator -> OCI Vault Secrets |

---

## 5. Staff-Level Engineering Scenarios Covered

* **Zero-Downtime Database Schema Migration**: Executing the Expand-Contract pattern to migrate relational schemas without breaking concurrent Blue and Green application fleets.
* **Automated Canary Rollback**: Wiring Prometheus/CloudWatch p99 latency and 5xx error rate thresholds to automated canary abort triggers within service meshes and ingress controllers.
* **Supply Chain Hardening**: Implementing cryptographic image signatures and admission controller webhooks on Kubernetes to reject any container image lacking verified provenance.
