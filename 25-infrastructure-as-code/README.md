# Module 25: Infrastructure as Code (IaC) & Terraform (AWS vs. OCI)

---

## 1. Module Overview & Learning Objectives

Infrastructure as Code (IaC) is the foundational engineering discipline that treats cloud infrastructure provisioning, configuration, and lifecycle management with the exact same rigor, reproducibility, and version control as software application code. Manual console operations ("ClickOps") introduce configuration drift, security blind spots, human error, and irreproducible environments. Modern cloud platforms mandate declarative, immutable, and testable infrastructure defined in code.

This module delivers advanced engineering depth on **HashiCorp Terraform / OpenTofu**, state management, remote backends, enterprise module architecture, drift detection, and Policy as Code across both Amazon Web Services (AWS) and Oracle Cloud Infrastructure (OCI).

### What You Will Master
1. **Terraform Core Engine Internals**: Graph theory, Directed Acyclic Graph (DAG) construction, topological sort, concurrency pools, and state file serialization.
2. **State Management & Remote Backends**: AWS S3 + DynamoDB distributed locking vs. OCI Object Storage + OCI Resource Manager (ORM).
3. **State Surgery & Lifecycle Operations**: `terraform state mv`, `rm`, `import`, and native declarative `import` blocks (Terraform 1.5+).
4. **Enterprise Reusable Module Architecture**: Composition patterns, module versioning, variable validation, and multi-provider aliasing across multi-region AWS and multi-compartment OCI environments.
5. **Drift Detection & Automated Reconciliation**: CI/CD drift detection workflows, `-detailed-exitcode` automation, and native OCI Resource Manager drift scanning.
6. **Policy as Code & Static Analysis**: Enforcing security guardrails and compliance with Open Policy Agent (OPA) / Rego, Checkov, Trivy, and native `terraform test`.

---

## 2. Directory Roadmap & Lesson Catalog

```
25-infrastructure-as-code/
├── README.md                                                  # Module guide & architectural index
├── 01-terraform-architecture-state-and-backends.md           # [Major] Terraform DAG, S3/DynamoDB vs OCI Object Storage/ORM, state surgery
├── 02-terraform-module-design-and-drift-detection.md         # [Major] Enterprise modules, dual-cloud aliasing, drift reconciliation
└── 03-iac-testing-static-analysis-and-policy-as-code.md       # [Supporting] Policy-as-Code (OPA/Checkov), Infracost, terraform test
```

---

## 3. Terraform Engine Execution Graph

```
[ Configuration (.tf files) ]
              |
              v
[ 1. Syntax Parsing & HCL Decode ]
              |
              v
[ 2. Directed Acyclic Graph (DAG) Construction ]
     Identifies dependencies: VPC -> Subnets -> Route Tables -> EC2
              |
              v
[ 3. Topological Sort & Concurrency Execution ]
     Executes independent nodes in parallel (default -parallelism=10)
              |
              v
[ 4. State Refresh & Delta Comparison (Desired vs Actual) ]
              |
              v
[ 5. Plan Generation (Create, Update in-place, Destroy) ]
              |
              v
[ 6. Apply Execution & State File Serialization ]
```

---

## 4. Side-by-Side Dual-Cloud IaC Ecosystems

| IaC Domain | AWS Native & Ecosystem | OCI Native & Ecosystem |
| :--- | :--- | :--- |
| **Primary Declarative Tool**| Terraform / OpenTofu / CloudFormation / AWS CDK | Terraform / OpenTofu / OCI Resource Manager (ORM) |
| **Official Provider** | `hashicorp/aws` | `oracle/oci` |
| **Remote State Storage** | Amazon S3 (Versioned, Encrypted) | OCI Object Storage (S3-compatible or native) |
| **Distributed State Lock** | Amazon DynamoDB (`LockID` attribute) | OCI Object Storage native HTTP lease lock / ORM |
| **Managed IaC Service** | AWS CloudFormation / Proton | **OCI Resource Manager (ORM)** (Managed Terraform SaaS) |
| **Drift Detection** | CloudFormation Drift Detection / Scheduled Actions | Native **OCI Resource Manager Drift Detection** |
| **Multi-Tenancy Primitives**| Accounts & Organizations | Compartments (`compartment_id` on every resource) |

---

## 5. Staff-Level Engineering Scenarios Covered

* **State File Corruption Recovery**: Restoring from S3/OCI versioned state histories, fixing orphan lock IDs, and repairing JSON state trees.
* **Refactoring Without Downtime**: Renaming resource addresses and extracting resources into child modules using `moved` blocks without triggering resource destruction.
* **Dual-Cloud Multi-Provider Aliasing**: Structuring clean, decoupled root and child modules that provision interconnected AWS VPCs and OCI VCNs without provider race conditions.
