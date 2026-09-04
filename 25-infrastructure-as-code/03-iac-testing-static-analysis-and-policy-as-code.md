# IaC Testing, Static Analysis & Policy-as-Code: OPA, Checkov & Native Testing (AWS vs. OCI)

---

## 1. Executive Summary & Core Definitions

In high-velocity cloud engineering environments, code reviews alone cannot prevent security misconfigurations, compliance breaches, or catastrophic architectural anti-patterns from leaking into production. An engineer might accidentally write a security group rule opening port 22 or port 3389 to `0.0.0.0/0`, provision an unencrypted S3 bucket or OCI Block Volume, or deploy an oversized bare-metal database instance that costs $20,000 per month.

**Policy as Code (PaC)** and **IaC Static Analysis** automate governance by treating architectural policies, compliance requirements (SOC 2, PCI-DSS, HIPAA), and cost guardrails as testable, version-controlled software rules executed inside pull request CI/CD gates prior to infrastructure provisioning.

```
       TRADITIONAL GOVERNANCE (Reactive & Slow)
Terraform Apply ---> Cloud Provisioned ---> Security Audit (30 Days Later) ---> Security Incident

       POLICY-AS-CODE GUARDRAILS (Proactive & Automated)
Pull Request Created ---> Static Scan (Checkov / Trivy) ---> OPA / Rego Policy Gate ---> Merge Approved
                                 |                                    |
                           Failed Scan?                         Policy Breached?
                                 v                                    v
                           [ PR BLOCKED ]                       [ PR BLOCKED ]
```

### Core Terminology
* **Policy as Code (PaC)**: The practice of defining security, compliance, and architectural constraints in declarative programming languages that evaluate IaC plans programmatically.
* **Open Policy Agent (OPA) / Rego**: A vendor-neutral, general-purpose open-source policy engine that uses the declarative language **Rego** to evaluate JSON-formatted Terraform plan files [Doc: Open Policy Agent, checked 2026].
* **Checkov**: A static code analysis tool specifically engineered to scan Terraform, CloudFormation, and Kubernetes manifests for security and compliance misconfigurations against 1,000+ pre-built industry benchmarks (CIS Benchmarks, NIST).
* **`terraform test`**: The native testing framework introduced in Terraform 1.6+ that allows authors to write automated unit and integration tests for modules using native HCL `.tftest.hcl` files.
* **Infracost**: A cloud cost estimation engine that parses Terraform plan files and posts automated pull request comments detailing the exact monthly dollar impact before merging.

---

## 2. Architectural Deep Dive: Static Analysis & Policy-as-Code Frameworks

### The IaC Testing Pyramid

```
                / \
               /   \
              / E2E \          End-to-End Tests (Deploy real ephemeral resources via 'terraform apply')
             / Tests \         Slow (~10m), Expensive ($$$), High Fidelity
            /---------\
           /  Module   \       Integration Tests ('terraform test' with mock providers)
          / Integration \      Moderate speed (~30s), Verifies DAG & variable interpolation
         /---------------\
        / Policy as Code  \    OPA / Rego & Sentinel (Validates JSON execution plans against guardrails)
       /   (OPA / Rego)    \   Fast (~5s), Zero Cloud Spend, Hard Compliance Gates
      /---------------------\
     / Static Code Analysis  \ Static Linters (Checkov, Trivy, tflint)
    /   (Checkov / Trivy)     \ Instant (< 2s), Catches basic misconfigurations and CVEs
   +---------------------------+
```

---

## 3. Side-by-Side Comparison: Policy-as-Code Engines

| Capability / Dimension | Open Policy Agent (OPA) / Rego | Checkov | Native `terraform test` (v1.6+) |
| :--- | :--- | :--- | :--- |
| **Language Engine** | **Rego** (Declarative query language) | Python / YAML | **Native HCL** (`.tftest.hcl`) |
| **Primary Scope** | General-purpose policy evaluation | Security & compliance posture | Module logic & output validation |
| **Input Format** | JSON representation of plan (`terraform show -json`) | Raw `.tf` files or JSON plans | Native Terraform configuration |
| **Pre-built Rule Sets** | Community libraries | **1,000+ out-of-the-box security checks** | None (User authors assertions) |
| **Custom Rule Authoring**| Outstanding (Extreme mathematical power)| Supported via custom Python/YAML | Supported via native HCL `assert` |
| **Cost Estimation** | Not supported | Integrated with Bridgecrew | Not supported |
| **Execution Speed** | Sub-second evaluation | Fast (~2 to 10 seconds) | Fast (Mock) to Moderate (Apply) |

---

## 4. Implementation & Configuration (Terraform / Rego / CLI)

### OPA / Rego Policy: Forbid Public Ingress on Port 22/3389 (`policy/security.rego`)

```rego
package terraform.security

import future.keywords.in

default allow = false

# Allow deployment only if zero critical security violations exist
allow {
    count(violations) == 0
}

# Rule: Deny any security group rule allowing 0.0.0.0/0 on SSH (22) or RDP (3389)
violations[msg] {
    some resource in input.resource_changes
    resource.type == "aws_security_group_rule"
    resource.change.after.type == "ingress"
    "0.0.0.0/0" in resource.change.after.cidr_blocks

    port := resource.change.after.from_port
    port in [22, 3389]

    msg := sprintf("SECURITY VIOLATION: Resource '%v' opens unrestricted public access (0.0.0.0/0) to port %v!", [resource.address, port])
}

# OCI Rule: Forbid open CIDR on OCI Security Lists
violations[msg] {
    some resource in input.resource_changes
    resource.type == "oci_core_security_list"
    some rule in resource.change.after.ingress_security_rules
    rule.source == "0.0.0.0/0"
    rule.tcp_options[0].destination_port_range[0].min in [22, 3389]

    msg := sprintf("SECURITY VIOLATION: OCI Security List '%v' allows public SSH/RDP from 0.0.0.0/0!", [resource.address])
}
```

---

### Native Terraform 1.6+ Module Test (`tests/vpc_test.tftest.hcl`)

```hcl
# Unit testing the VPC module with native assertions
run "verify_vpc_cidr_and_subnets" {
  command = plan # Evaluates plan without creating physical infrastructure

  variables {
    cidr_block  = "10.0.0.0/16"
    environment = "prod"
    azs         = ["us-east-1a", "us-east-1b"]
  }

  # Assert that VPC receives correct CIDR
  assert {
    condition     = aws_vpc.this.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block does not match configured input variable."
  }

  # Assert that exactly 2 private subnets are planned
  assert {
    condition     = length(aws_subnet.private) == 2
    error_message = "Module failed to plan exactly two private subnets."
  }

  # Assert DNS hostnames are enabled
  assert {
    condition     = aws_vpc.this.enable_dns_hostnames == true
    error_message = "Production VPC must have DNS hostnames enabled."
  }
}
```

---

## 5. Failure Modes, Edge Cases & Guardrail Bypass Risks

```
[ Developer Bypasses Policy Gate ]
Manual 'terraform apply' locally via personal AWS credentials ---> CI/CD Guardrails BYPASSED!
                                  |
                                  v
[ ARCHITECTURAL MITIGATION: ZERO LOCAL WRITE CREDENTIALS ]
Developers have zero AWS/OCI write access. All provisioning executes exclusively via CI/CD runners!
```

### Critical Edge Cases
1. **Dynamic HCL Evaluation Blind Spots**:
   * Static linters scanning raw `.tf` files cannot evaluate dynamic values generated at plan-time (e.g., `cidrsubnet()`, string concatenations, or remote state lookups).
   * *Mitigation*: Run policy engines against the **compiled JSON plan** (`terraform show -json tfplan.binary`) where all variables and dynamic interpolations are fully resolved.
2. **Alert Fatigue from False Positives**:
   * If Checkov or OPA emits 50 non-critical warnings on every PR, developers will configure CI to ignore exit codes.
   * *Rule*: Strictly separate policies into **Warnings** (informational Slack notification) and **Hard Blocking Errors** (blocks merge: e.g., unencrypted storage, public S3, open port 22).

---

## 6. Real-World Case Study / Security Breach Prevented by IaC Guardrail

* **Context**: Global FinTech payment processing company.
* **The Incident**: A developer needed to test database connectivity from a home office and changed an AWS Security Group in Terraform:
  ```hcl
  cidr_blocks = ["0.0.0.0/0"] # Quick test change
  ```
* **The Defense**:
  * When the pull request was submitted, the GitHub Actions pipeline executed OPA / Rego against the JSON plan.
  * The rule `deny_public_database_ingress` caught the `0.0.0.0/0` rule on port 5432 (PostgreSQL).
  * The pipeline immediately returned exit code 1, blocked the merge, and tagged the security on-call engineer.
  * What would have been a catastrophic data breach exposing customer credit card ledgers was neutralized in **3 seconds** before a single cloud API call was executed.

---

## 7. Interview Defense & Technical Trade-Offs

### Scenario: Implementing Policy-as-Code in an Enterprise Without Stalling Velocity

* **Interviewer**: "We want to enforce strict cloud security standards, but developers complain that CI/CD policy gates slow down deployments. How do you roll out Policy as Code?"
* **Staff Candidate Response**:
  1. *Phase 1: Audit Mode (Zero Blocking)*: Deploy Checkov and OPA in "Audit / Soft Fail" mode for 30 days. Collect telemetry on common violations without failing pipelines.
  2. *Phase 2: Fix the Golden Modules*: Update the centralized enterprise child modules to ensure default configurations comply with all rules. If modules are secure by default, developers rarely violate policies.
  3. *Phase 3: Enforce Hard Blocks on Tier-1 Critical Violations Only*: Enforce hard pipeline blocks strictly on existential security risks (unencrypted storage, public database access, public SSH/RDP, missing cost center tags).
  4. *Phase 4: Fast Feedback*: Run static linters (Checkov) locally via pre-commit hooks (`pre-commit install`) so engineers catch violations in their IDE in 500 ms before pushing code.
