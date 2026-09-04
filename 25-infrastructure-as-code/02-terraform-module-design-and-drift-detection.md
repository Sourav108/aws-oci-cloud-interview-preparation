# Enterprise Terraform Module Architecture, Drift Detection & Multi-Provider Design (AWS vs. OCI)

---

## 1. Executive Summary & Core Definitions

In mature cloud engineering organizations, Infrastructure as Code is not written as monolithic, thousand-line flat scripts. Monolithic configurations create massive blast radiuses, slow plan executions, and prevent team collaboration. Enterprise infrastructure demands **modular, reusable, composable, and version-controlled architectural components**.

Simultaneously, even the most disciplined GitOps pipelines face the reality of **Configuration Drift**: out-of-band manual emergency changes executed via cloud consoles, automated auto-scaling adjustments, or external security automation modifying resources directly. Without continuous, automated drift detection and reconciliation, the codebase ceases to reflect physical cloud reality, turning subsequent deployments into high-risk events.

```
+---------------------------------------------------------------------------------------------------+
|                            ENTERPRISE MODULE COMPOSITION & DRIFT CYCLE                            |
+---------------------------------------------------------------------------------------------------+
|  [ Versioned Git Registry ]                                                                       |
|  - vpc-module (v2.1.0)                                                                            |
|  - compute-module (v1.4.0)                                                                        |
|            |                                                                                      |
|            v (Composition over Inheritance)                                                       |
|  [ Root Environment Module ]                                                                      |
|  (Production / Staging / Dev)                                                                     |
|            |                                                                                      |
|    terraform plan -detailed-exitcode                                                              |
|            |                                                                                      |
|            +=====> Exit Code 0: Clean (No Drift)                                                  |
|            |                                                                                      |
|            +=====> Exit Code 2: DRIFT DETECTED! ====> [ Alert SRE / Automated Re-Apply ]          |
+---------------------------------------------------------------------------------------------------+
```

### Core Terminology
* **Root Module**: The top-level Terraform directory where `terraform init` and `terraform apply` are executed, combining and instantiating child modules for a specific environment (e.g., `environments/prod`).
* **Child Module**: A self-contained, reusable package of Terraform configuration files invoked by a root module or another child module using a `module` block.
* **Composition over Inheritance**: An enterprise design pattern where infrastructure layers (networking, databases, compute) are written as decoupled child modules that communicate strictly via explicit input variables and output values rather than deep nested inheritance.
* **Configuration Drift**: The divergence between the real-world state of cloud resources and the declarative state declared in Git and recorded in the state file.
* **Detailed Exit Code (`-detailed-exitcode`)**: A Terraform CLI flag that returns exit code `2` if a plan detects any diff (drift), exit code `0` if no changes exist, and exit code `1` on execution errors.
* **Provider Aliasing**: A mechanism allowing multiple distinct instances of the same cloud provider (e.g., different AWS regions or different OCI compartments) to be configured within the same root configuration and passed explicitly to child modules.

---

## 2. Distributed Systems Theory & Architecture

### Enterprise Module Hierarchy & Layered Decoupling

Modern cloud architecture rejects "God Modules" that provision VPCs, Kubernetes clusters, and databases in a single execution tree. Instead, infrastructure is decomposed into independent, version-controlled tiers:

```
+---------------------------------------------------------------------------------------+
| LAYER 0: FOUNDATIONS (Managed by Core Cloud Platform Team - Low Velocity)             |
| Root State: AWS Organizations / OCI Compartments, IAM Policies, KMS Master Keys       |
+---------------------------------------------------------------------------------------+
                                           | Output: Compartment IDs / KMS ARNs
                                           v
+---------------------------------------------------------------------------------------+
| LAYER 1: NETWORKING (Managed by NetOps / Platform - Moderate Velocity)                |
| Root State: AWS VPCs, Transit Gateways / OCI VCNs, DRG, NAT Gateways                  |
+---------------------------------------------------------------------------------------+
                                           | Output: VPC IDs, Subnet IDs, Route Tables
                                           v
+---------------------------------------------------------------------------------------+
| LAYER 2: PLATFORM & DATA (Managed by Platform / SRE Team - Moderate Velocity)         |
| Root State: EKS / OKE Clusters, RDS Aurora / OCI Autonomous Database, Redis Caches    |
+---------------------------------------------------------------------------------------+
                                           | Output: Cluster Endpoints, DB Hostnames
                                           v
+---------------------------------------------------------------------------------------+
| LAYER 3: APPLICATIONS & WORKLOADS (Managed by Product Engineering - High Velocity)    |
| Root State: Kubernetes Manifests, Ingress Controllers, Service Mesh Routes            |
+---------------------------------------------------------------------------------------+
```

#### Why Layered Decoupling is Mandatory:
1. **Blast Radius Containment**: A syntax error in an application deployment cannot accidentally drop a production VPC or Transit Gateway.
2. **Execution Velocity**: Planning an application deployment takes 10 seconds against a small state file rather than 10 minutes traversing 2,000 resources.
3. **Role-Based Access Control (RBAC)**: Product engineers are granted IAM write permissions only to Layer 3 state buckets, completely preventing unauthorized changes to core networking or KMS encryption tiers.

---

## 3. Core Mechanics & Deep Dive

### Multi-Provider Aliasing (AWS vs. OCI)

Real-world architectures frequently require coordinating resources across multiple geographic regions or corporate cloud tenancies.

```
[ Root Module ]
      |
      +---> Provider "aws" (Default: us-east-1)      ===> us-east-1 VPC & Aurora Primary
      +---> Provider "aws.west" (Alias: us-west-2)   ===> us-west-2 Secondary VPC & Standby DB
      |
      +---> Provider "oci" (Default: us-ashburn-1)   ===> Ashburn Production Compartment
      +---> Provider "oci.phx" (Alias: us-phoenix-1) ===> Phoenix DR Compartment
```

#### AWS Multi-Region Provider Aliasing Mechanics:
In AWS, providers are declared with an `alias` and passed to child modules via the `providers` map:

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "us_west_2"
  region = "us-west-2"
}

module "disaster_recovery_vpc" {
  source = "./modules/aws-vpc"
  providers = {
    aws = aws.us_west_2 # Explicitly binds the aliased provider to child module
  }
  cidr_block = "10.1.0.0/16"
}
```

#### OCI Multi-Region & Multi-Tenancy Provider Aliasing:
In OCI, provider aliasing is commonly used not only for multi-region architectures, but also for **multi-compartment or cross-tenancy delegations**:

```hcl
provider "oci" {
  region       = "us-ashburn-1"
  tenancy_ocid = var.root_tenancy_ocid
}

provider "oci" {
  alias        = "dr_phoenix"
  region       = "us-phoenix-1"
  tenancy_ocid = var.root_tenancy_ocid
}

module "dr_network" {
  source = "./modules/oci-vcn"
  providers = {
    oci = oci.dr_phoenix
  }
  compartment_ocid = var.dr_compartment_ocid
  cidr_block       = "10.2.0.0/16"
}
```

---

## 4. Architecture & Data Flow Diagrams

### Automated Drift Detection & Reconciliation Pipeline

```
[ Scheduled GitHub Actions Cron (Every 4 Hours) ]
                        |
                        v
          Step 1: Check out main branch
                        |
                        v
          Step 2: terraform init (Connect to S3 / OCI Backend)
                        |
                        v
          Step 3: terraform plan -detailed-exitcode -no-color
                        |
          +-------------+-------------+
          |                           |
    [ Exit Code 0 ]             [ Exit Code 2 ] (DRIFT DETECTED!)
          |                           |
          v                           v
    Log: "No Drift"             Step 4: Parse Execution Diff
    Pipeline Passes Cleanly     Identify drifted resources (e.g., Security Group rule modified)
                                      |
                                      v
                                Step 5: Trigger SRE PagerDuty / Slack Alert
                                      |
                                +-----+-----+
                                |           |
                          [ MANUAL ]     [ AUTOMATED ]
                          Review PR      Auto-apply Git state to overwrite manual drift
```

---

## 5. Side-by-Side Architectural Comparison: AWS vs. OCI

| Feature / Dimension | AWS IaC Module Architecture | OCI IaC Module Architecture |
| :--- | :--- | :--- |
| **Standard Module Registry**| Terraform Public Registry / AWS Private Registry | Terraform Registry / OCI Resource Manager Catalog |
| **Multi-Tenancy Context** | AWS Account ID / Region | **Compartment OCID** (Explicit input to all child modules) |
| **Resource Naming Patterns**| Descriptive string tags (`Name = "prod-vpc"`) | `display_name` + Defined & Freeform Tags |
| **Provider Aliasing** | Primarily for multi-region VPC/DB peering | Multi-region AND multi-compartment delegations |
| **Native Drift Detection** | AWS CloudFormation Drift Detection | **OCI Resource Manager Native Drift Detection** |
| **Drift Reporting Format** | JSON / CloudWatch Events | Detailed visual console diff & OCI Monitoring events |
| **Variable Validation** | Native HCL `validation` blocks | Native HCL `validation` blocks |
| **Workspace Strategy** | Supported (`terraform workspace select prod`) | OCI Resource Manager Stacks (1 stack per environment) |

---

## 6. Deep Implementation & Configuration (Terraform / CLI / SDK)

### Enterprise Production Child Module: Multi-AZ AWS VPC (`modules/aws-vpc/main.tf`)

```hcl
terraform {
  required_version = ">= 1.8.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

variable "cidr_block" {
  type        = string
  description = "IPv4 CIDR block for the VPC"

  validation {
    condition     = can(cidrhost(var.cidr_block, 0)) && can(regex("^10\\.", var.cidr_block))
    error_message = "The cidr_block variable must be a valid 10.x.x.x private network CIDR."
  }
}

variable "environment" {
  type        = string
  description = "Deployment environment name"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "azs" {
  type        = list(string)
  description = "List of Availability Zones to deploy subnets into"
}

resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.cidr_block, 4, count.index)
  availability_zone = var.azs[count.index]

  tags = {
    Name        = "${var.environment}-private-subnet-${var.azs[count.index]}"
    Environment = var.environment
    Tier        = "Private"
  }
}

output "vpc_id" {
  value       = aws_vpc.this.id
  description = "Physical identifier of the provisioned VPC"
}

output "private_subnet_ids" {
  value       = aws_subnet.private[*].id
  description = "List of private subnet IDs"
}
```

---

### Enterprise Production Child Module: OCI VCN with Compartment Binding (`modules/oci-vcn/main.tf`)

```hcl
terraform {
  required_version = ">= 1.8.0"
  required_providers {
    oci = {
      source  = "oracle/oci"
      version = ">= 5.0"
    }
  }
}

variable "compartment_id" {
  type        = string
  description = "OCID of the target compartment"

  validation {
    condition     = can(regex("^ocid1\\.compartment\\.oc1\\.", var.compartment_id))
    error_message = "The compartment_id must be a valid OCI compartment OCID."
  }
}

variable "vcn_cidr" {
  type        = string
  description = "CIDR block for the Virtual Cloud Network"
  default     = "10.0.0.0/16"
}

variable "environment" {
  type = string
}

resource "oci_core_vcn" "this" {
  compartment_id = var.compartment_id
  cidr_block     = var.vcn_cidr
  display_name   = "${var.environment}-core-vcn"
  dns_label      = "${var.environment}vcn"

  freeform_tags = {
    "Environment" = var.environment
    "ManagedBy"   = "Terraform"
  }
}

resource "oci_core_subnet" "private" {
  compartment_id             = var.compartment_id
  vcn_id                     = oci_core_vcn.this.id
  cidr_block                 = cidrsubnet(var.vcn_cidr, 4, 1)
  display_name               = "${var.environment}-private-subnet"
  prohibit_public_ip_on_vnic = true # Strict private subnet
}

output "vcn_id" {
  value       = oci_core_vcn.this.id
  description = "OCID of the provisioned VCN"
}

output "private_subnet_id" {
  value       = oci_core_subnet.private.id
  description = "OCID of the private subnet"
}
```

---

### GitHub Actions Scheduled Drift Detection Workflow (`.github/workflows/drift.yml`)

```yaml
name: Scheduled Terraform Drift Detection

on:
  schedule:
    - cron: '0 */4 * * *' # Run every 4 hours
  workflow_dispatch:

jobs:
  detect-drift:
    name: Audit Production Drift
    runs-on: ubuntu-latest

    permissions:
      id-token: write
      contents: read

    steps:
      - name: Check out Git Repository
        uses: actions/checkout@v4

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDriftDetectorRole
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.8.5

      - name: Initialize Backend
        run: terraform init
        working-directory: ./environments/prod

      - name: Execute Plan with Detailed Exit Code
        id: plan
        run: |
          set +e
          terraform plan -detailed-exitcode -no-color -out=drift.tfplan
          EXIT_CODE=$?
          echo "exit_code=${EXIT_CODE}" >> $GITHUB_OUTPUT
          exit 0
        working-directory: ./environments/prod

      - name: Handle Drift Results
        if: steps.plan.outputs.exit_code == '2'
        run: |
          echo "::error::CRITICAL: Infrastructure drift detected in Production!"
          curl -X POST -H 'Content-type: application/json' \
            --data '{"text":"🚨 *ALERT*: Configuration Drift Detected in AWS Production! Runbook: https://wiki.internal/drift"}' \
            ${{ secrets.SLACK_WEBHOOK_URL }}
          exit 2
```

---

## 7. Failure Modes, Edge Cases & Blast Radius

| Failure Mode | Trigger Mechanism | Blast Radius | Mitigation Strategy |
| :--- | :--- | :--- | :--- |
| **Unpinned Child Module Version** | Module source declared as `ref=main`; upstream author pushes breaking change | Next `terraform init -upgrade` downloads breaking code; pipeline fails or destroys resources | Strictly pin module versions using semantic Git tags (`?ref=v2.1.4`) or registry versions (`~> 2.1.0`). |
| **Console ClickOps Drift Overwrite** | SRE fixes emergency outage via AWS/OCI console; subsequent CI apply silently deletes manual fix | Production outage re-triggered automatically upon merge | Enforce continuous drift alerting; backport emergency changes into Git immediately following incidents. |
| **Cross-Module Output Cycle** | Module A takes output of Module B; Module B takes output of Module A | DAG construction fails with `Error: Cycle in graph` | Decouple shared dependency into a dedicated Tier 0 Foundation module; never cross-reference horizontally. |
| **State Bloat Plan Latency** | Monolithic root module contains 800 resources; `terraform plan` takes 18 minutes | Deployment velocity grinds to a halt; API rate limits saturated | Refactor into independent root modules separated by lifecycle and velocity. |

---

## 8. Security, Compliance & Threat Modeling

### Guarding Module Registries & IaC Supply Chains

```
[ Developer PR ] ---> [ Static Policy Scan (Checkov / Trivy) ]
                                    |
                                    v
                      [ Hashicorp Sentinel / OPA Gate ]
                                    |
                                    v
                      [ Approved Enterprise Module Registry ]
```

1. **Private Module Registries**:
   * Forbid developers from sourcing unvetted public modules (`source = "github.com/random-user/..."`).
   * Curate hardened, security-scanned modules inside an internal Git repository or private Terraform Cloud / OCI Resource Manager catalog.
2. **Mandatory Tagging Enforcement**:
   * Enforce organizational cost allocation tags (`CostCenter`, `Owner`, `Environment`) within child modules using variable validations or OPA policies, rejecting any resource missing mandatory governance tags.

---

## 9. Performance Tuning & Latency Engineering

### Optimizing Module Graph Compilation

1. **Pruning Stale `data` Sources**:
   * Uncached data source lookups (e.g., `data "aws_ami"` or `data "oci_core_images"`) query cloud APIs on every single plan. If invoked inside nested loops (`for_each`), they multiply API latency by $N \times$.
   * Query data sources once at the root level and pass the resolved ID to child modules.

2. **Avoiding Nested Module Sprawl**:
   * Deep module nesting (Module calls Sub-module calls Sub-module calls Sub-module) significantly degrades readability and complicates graph compilation. Limit module hierarchy to **maximum 2 levels** (Root Module $\to$ Reusable Child Module).

---

## 10. Observability, Telemetry & SRE Metrics

### IaC Health & Drift Telemetry

| Metric Name | Source | Description | SRE Alert Threshold |
| :--- | :--- | :--- | :--- |
| `ConfigurationDriftCount` | CI/CD Drift Cron | Number of resources deviating from Git state | Count > 0 (Slack alert) |
| `TerraformGraphNodeCount` | `terraform graph` | Total vertices evaluated in internal DAG | Node count > 500 (Refactor warning) |
| `ModuleVersionAgeDays` | SRE Audit Scanner | Number of days a root module is behind latest child module tag | > 90 days out of date |
| `ORMDriftJobStatus` | OCI Monitoring | Status of native OCI Resource Manager drift check | Status == `DRIFT_DETECTED` |

---

## 11. Cost Modeling & Capacity Planning

### Financial Impact of Module Architecture: Workspaces vs. Directory Roots

| Architectural Pattern | Maintenance Cost | Blast Radius Risk | Cloud Cost Transparency |
| :--- | :--- | :--- | :--- |
| **Single State with Workspaces** | Low (Single code tree) | High (Accidental `prod` overwrite) | Poor (Shared state backend) |
| **Directory Roots (`envs/prod`, `envs/dev`)** | Moderate (Small duplication) | **Zero (Completely isolated states)**| **Outstanding (Clean cost center mapping)** |
| **Terragrunt / Atmos Layering** | Moderate (DRY abstraction) | Zero (Isolated per-component states)| Outstanding |

*Staff Architectural Consensus*: For production enterprise infrastructure, **directory-based roots** or **Terragrunt** are overwhelmingly preferred over native Terraform Workspaces. Workspaces share identical backend configurations, making it trivially easy for an engineer to accidentally apply staging variables against production state.

---

## 12. Operational Runbooks & Triage Playbooks

### Runbook: Reconciling High-Severity Configuration Drift

```
[ PagerDuty Alert: Drift Detected in 'production-security-groups' ]
                                 |
                                 v
              Step 1: Inspect Detailed Plan Diff Output
       (Which specific rule or attribute was modified out-of-band?)
                                 |
              +------------------+------------------+
              |                                     |
    [ Emergency Hotfix Found ]            [ Rogue Console Edit / Malware ]
    (e.g., SRE opened SSH at 03:00)       (Unauthorized IP added to whitelist)
              |                                     |
              v                                     v
    Step 2: Reconcile to Git              Step 2: Overwrite Immediately
    Add security rule to Git code;        Execute 'terraform apply' to
    Merge PR cleanly to reflect reality   purge unauthorized rule
              |                                     |
              +------------------+------------------+
                                 |
              Step 3: Verify Zero Drift on Re-Plan:
              terraform plan -detailed-exitcode ===> Returns 0
```

---

## 13. Edge Cases, Quirks & Gotchas

### Subtle Terraform Production Quirks

1. **The Difference Between `count` and `for_each`**:
   * *The Trap of `count`*: If you declare subnets using `count = length(var.subnets)`, the state keys them by index: `aws_subnet.this[0]`, `aws_subnet.this[1]`, `aws_subnet.this[2]`. If you delete the first item from the variable list, Terraform shifts all indices down. As a result, Terraform **destroys and recreates all subnets**!
   * *Mandate*: Always use `for_each = toset(var.subnets)` for resources with unique identities. Deleting an item removes only that specific named resource without shifting siblings.
2. **OCI Flexible Shape Memory/CPU Validation**:
   * When instantiating `VM.Standard3.Flex` in OCI, memory must maintain a strict ratio to OCPUs (between 1 GB and 64 GB per OCPU). Encode this rule directly into your module's input validation blocks to catch misconfigurations during `terraform validate` rather than 2 minutes into an apply.

---

## 14. Real-World Case Study / Postmortem

### Incident: The `count` Index Shift Catastrophe

* **Context**: Global retail platform operating 5 primary database read replicas provisioned via Terraform on AWS.
* **The Incident**: A junior engineer removed the oldest read replica (`db-read-01`) from the input variable list:
  ```hcl
  # Old list: ["db-read-01", "db-read-02", "db-read-03", "db-read-04", "db-read-05"]
  # New list: ["db-read-02", "db-read-03", "db-read-04", "db-read-05"]
  ```
* **The Failure**:
  1. The module was authored using `count = length(var.replicas)`.
  2. Terraform calculated:
     * `aws_db_instance.replica[0]` (was `db-read-01`) must be modified to become `db-read-02`.
     * `aws_db_instance.replica[1]` (was `db-read-02`) must be modified to become `db-read-03`.
     * `aws_db_instance.replica[4]` must be destroyed.
  3. Because database identifiers cannot be changed in-place, Terraform generated a plan to **destroy and recreate all 4 remaining active read replicas simultaneously**!
  4. The PR was approved without reading the detailed diff. 100% of read replica capacity vanished, crashing production during peak hours.
* **The Fix**: Re-architected all enterprise modules to strictly mandate `for_each` over stable map keys (`for_each = { for r in var.replicas : r.name => r }`).

---

## 15. Architectural Trade-Off Analysis

| Module Structure | Code Reusability | Blast Radius Containment | Refactoring Flexibility | Cognitive Overhead |
| :--- | :--- | :--- | :--- | :--- |
| **Flat Monolith** | None (Zero reuse) | Terrible (Single state) | High (Everything local) | Low (Simple to start) |
| **Deep Nested Modules** | High | Poor (Coupled DAG) | Terrible (Breaking shifts) | Extreme |
| **Layered Modular Roots**| **High** (Versioned child)| **Maximum (Isolated tiers)**| **High (Decoupled contracts)** | Moderate (Enterprise standard) |

---

## 16. Cloud Migration & Dual-Cloud Considerations

### Harmonizing Modules Across AWS and OCI

When building a unified multi-cloud platform:

```
                  [ Enterprise Git Monorepo ]
                               |
            +------------------+------------------+
            |                                     |
   [ modules/aws-network ]               [ modules/oci-network ]
   - Inputs: cidr, env                   - Inputs: cidr, env, compartment_id
   - Outputs: vpc_id, subnets            - Outputs: vcn_id, subnets
            \                                     /
             +-----------------+-----------------+
                               |
               [ Common Contract Normalization ]
               - Normalized naming conventions
               - Standard tag keys: Project, Owner, CostCenter
```

* **Interface Parity**: Ensure child modules for AWS and OCI export functionally equivalent output contracts (e.g., both export a list of private subnet IDs), allowing upstream orchestration layers or Kubernetes deployment pipelines to consume outputs agnostically.

---

## 17. Automated Verification & Testing

### Script: Static Module Linting & Validation (Bash)

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=== Commencing Enterprise Terraform Module Validation ==="

MODULE_DIRS=$(find modules -maxdepth 1 -mindepth 1 -type d)

for mod in ${MODULE_DIRS}; do
    echo "Auditing module: ${mod}"
    terraform -chdir="${mod}" init -backend=false
    terraform -chdir="${mod}" validate
    terraform -chdir="${mod}" fmt -check
done

echo "SUCCESS: All enterprise child modules passed structural validation."
```

---

## 18. Staff+ Engineering Wisdom & Insights

### Production IaC Architectural Laws

1. **Variables Must Have Types, Descriptions, and Validations**: A variable declared as `variable "subnets" {}` with no type, description, or validation is an architectural failure. High-performing engineering organizations mandate strict types, clear documentation, and defensive regex validation on all module inputs.
2. **Never Let Drift Go Unchecked**: If drift is not caught within 24 hours, the state file diverges irreversibly from reality. Schedule automated drift detection every 4 hours and treat drift alerts with the exact same severity as production software bugs.
3. **Always Prefer `for_each` over `count`**: Unless provisioning identical, anonymous, ephemeral worker nodes where order is completely irrelevant, never use `count`. Index-shift deletions have caused more enterprise database outages than any other single Terraform antipattern.

---

## 19. Interview Defense & Technical Discussion Scenarios

### Scenario 1: Mitigating Infrastructure Drift in Regulated Environments

* **Interviewer**: "In our financial platform, developers sometimes make out-of-band console changes during emergencies. How do you detect and permanently prevent drift?"
* **Staff Candidate Response**:
  1. *Prevention (Zero Console Access)*: Implement Service Control Policies (AWS SCPs) and OCI IAM policies that revoke write permissions (`ec2:*`, `oci:core:*`) for all human user roles in production. Human access is strictly read-only.
  2. *Break-Glass Protocol*: Emergency changes must require activating a short-lived, audited "Break-Glass Role" that emits high-priority security alarms to the SOC upon assumption.
  3. *Automated Detection*: Configure scheduled GitHub Actions running `terraform plan -detailed-exitcode` every 4 hours (or native OCI Resource Manager drift detection). If exit code `2` occurs, page the SRE team.
  4. *Automated Remediation*: For non-destructive drift, configure CI/CD to automatically re-apply the approved Git configuration, continuously resetting cloud reality to the declarative standard.

### Scenario 2: Refactoring a Monolithic Terraform Repository

* **Interviewer**: "We have a single 15,000-line `main.tf` file that provisions our entire company infrastructure. It takes 25 minutes to run `terraform plan`. How do you safely decompose it?"
* **Staff Candidate Response**:
  1. *Define the Decoupled Layers*: Categorize infrastructure into 4 tiers: Foundations (IAM/KMS), Networking (VPC/VCN), Data (RDS/Autonomous DB), and Compute (EKS/OKE).
  2. *Author Reusable Child Modules*: Extract resources into versioned child modules in `modules/`.
  3. *Create Independent Root States*: Create separate directories with independent S3/DynamoDB or OCI backend configurations (`roots/01-networking`, `roots/02-data`, etc.).
  4. *State Migration via `terraform state mv`*: Use `terraform state mv` to surgically migrate existing physical resource IDs from the monolithic state into the targeted micro-state files.
  5. *Verify with Dry Run*: Run `terraform plan` against each newly decomposed root directory to verify **zero planned adds, changes, or destroys**. Plan execution duration drops from 25 minutes to under 30 seconds per tier.

---

## 20. Summary Cheat Sheet & Reference Matrix

```
+---------------------------------------------------------------------------------------------------+
|                        MODULE DESIGN & DRIFT DETECTION CHEAT SHEET                                |
+--------------------------+------------------------------------+-----------------------------------+
| Characteristic           | Recommended Best Practice          | Anti-Pattern to Avoid             |
+--------------------------+------------------------------------+-----------------------------------+
| Resource Iteration       | `for_each` over named maps/sets    | `count` with index-sensitive lists|
| Module Dependency        | Explicit composition (Inputs/Outs) | Cross-module horizontal cycles    |
| Module Versioning        | Semantic Git tags (`?ref=v1.2.0`)  | Unpinned `ref=main`               |
| Variable Governance      | Strict types + `validation` blocks | Untyped `variable "x" {}`         |
| Environment Strategy     | Directory roots or Terragrunt      | Shared workspace states in prod   |
| Drift Detection Exit Code| `terraform plan -detailed-exitcode`| Blind manual console checks       |
| Exit Code 0              | Succeeded, zero differences (Clean)| N/A                               |
| Exit Code 2              | **Differences detected (DRIFT!)**  | Ignored drift warnings            |
| OCI Drift Engine         | **Native OCI Resource Manager**    | Manual state downloads            |
+--------------------------+------------------------------------+-----------------------------------+
```
