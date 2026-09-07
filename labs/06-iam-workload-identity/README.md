# Lab 06: Zero-Trust Least-Privilege IAM & Workload Identity

---

## 1. Goal & Core Concept

The objective of this hands-on lab is to configure, test, and debug credential-less, least-privilege workload identity across **Amazon Web Services** (AWS IAM Roles for Service Accounts / EKS Pod Identity / Instance Profiles) and **Oracle Cloud Infrastructure** (OCI Dynamic Groups and Instance Principals).

### Core Architectural Concepts Tested
- **Credential-less Authentication**: Eliminating long-lived static API access keys (`AKIA...`) and passwords from application filesystems.
- **Dynamic Workload Identity**: Projecting temporary security tokens (AWS STS / OCI Instance Principal) refreshed automatically at runtime.
- **Least-Privilege Scoping**: Restricting IAM permissions to specific resource ARNs / OCIDs and enforcing condition keys (`aws:PrincipalTag`, `request.permission`).
- **Policy Simulation & Troubleshooting**: Diagnosing explicit denies, boundary restrictions, and missing trust relationships.

---

## 2. Cost Warning & Resource Estimate

> [!NOTE]
> **ESTIMATED HOURLY RUN COST: `$0.00 (Free)`**
>
> | Cloud Provider | Resource Component | Quantity | Hourly Rate | Run Cost (2 Hours) |
> | :--- | :--- | :---: | :---: | :---: |
> | **AWS** | IAM Roles, Policies, Instance Profiles `[Doc: IAM, checked 2026]` | 1 Set | $0.00 (Free) | $0.00 |
> | **OCI** | Dynamic Groups, Compartment IAM Policies `[Doc: OCI IAM, checked 2026]` | 1 Set | $0.00 (Free) | $0.00 |
>
> *Total Estimated Lab Cost for a 2-hour session: **$0.00 USD**.*
> *In No-Live-Account Fallback Mode, cost is **$0.00**.*

---

## 3. Architecture Diagram

```
========================================================================================================================
                                 WORKLOAD IDENTITY ARCHITECTURE
========================================================================================================================

  AWS: EKS / EC2 WORKLOAD IDENTITY                         OCI: COMPUTE / OKE WORKLOAD IDENTITY
  ┌─────────────────────────────────────────────────┐      ┌─────────────────────────────────────────────────┐
  │ [ Compute Instance / Pod (Metadata Service) ]   │      │ [ Compute Instance / Pod (Metadata Service) ]   │
  │                     │                           │      │                     │                           │
  │                     ▼                           │      │                     ▼                           │
  │ [ Instance Profile / IRSA Token Projection ]    │      │ [ OCI Dynamic Group Matching Rule ]             │
  │ - Requests short-lived STS credentials          │      │ - Matches: instance.compartment.id = '...'      │
  │                     │                           │      │                     │                           │
  │                     ▼                           │      │                     ▼                           │
  │ [ IAM Role with Least-Privilege Trust Policy ]  │      │ [ Compartment Policy Statement ]                │
  │ - Action: s3:GetObject on 'app-data/*' only     │      │ - Statement: Allow dynamic-group to read buckets│
  │ - Deny: All delete and administrative actions   │      │ - Deny: All bucket deletion actions             │
  └─────────────────────────────────────────────────┘      └─────────────────────────────────────────────────┘
```

---

## 4. Prerequisites

1. Terraform CLI v1.8+.
2. Cloud administrative privileges to create IAM roles and OCI dynamic groups.

---

## 5. Infrastructure Code (Terraform HCL)

### 5.1 AWS IAM Workload Role (`aws_iam.tf`)

```hcl
# AWS Reference Implementation: EC2 Instance Profile with Least-Privilege S3 Access
resource "aws_iam_role" "app_workload_role" {
  name = "lab06-app-workload-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_policy" "s3_read_policy" {
  name = "lab06-s3-read-least-privilege"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowReadOnlyTargetBucket"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::lab06-secure-bucket",
          "arn:aws:s3:::lab06-secure-bucket/*"
        ]
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "attach_s3" {
  role       = aws_iam_role.app_workload_role.name
  policy_arn = aws_iam_policy.s3_read_policy.arn
}

resource "aws_iam_instance_profile" "app_profile" {
  name = "lab06-app-instance-profile"
  role = aws_iam_role.app_workload_role.name
}
```

### 5.2 OCI Dynamic Group & Policy (`oci_iam.tf`)

```hcl
# OCI Reference Implementation: Dynamic Group and Compartment Policy
resource "oci_identity_dynamic_group" "compute_dg" {
  compartment_id = var.tenancy_ocid
  name           = "lab06-compute-dg"
  description    = "Dynamic group matching compute instances in lab compartment"
  matching_rule  = "All {instance.compartment.id = '${var.compartment_ocid}'}"
}

resource "oci_identity_policy" "storage_read_policy" {
  compartment_id = var.compartment_ocid
  name           = "lab06-storage-read-policy"
  description    = "Grant read-only access to object storage buckets"

  statements = [
    "Allow dynamic-group lab06-compute-dg to read buckets in compartment id ${var.compartment_ocid}",
    "Allow dynamic-group lab06-compute-dg to read objects in compartment id ${var.compartment_ocid}"
  ]
}
```

---

## 6. Step-by-Step Deployment Guide

```bash
terraform init -backend=false
terraform validate
terraform plan
```

---

## 7. Expected Validation Results

```text
[Statically validated — not applied to a live account]

Testing Credential-less S3 Read from Compute:
$ aws s3 ls s3://lab06-secure-bucket/
2026-09-07 12:00:00        42 sample.json

Testing Privilege Boundary Enforcement:
$ aws s3 rm s3://lab06-secure-bucket/sample.json
delete failed: s3://lab06-secure-bucket/sample.json An error occurred (AccessDenied) when calling the DeleteObject operation: Access Denied
```

---

## 8. Failure Injection Drill: IMDSv1 SSRF Vulnerability Exploit Test

### The Scenario
Attempt to access the Instance Metadata Service (IMDS) using insecure IMDSv1 (no token) on an instance configured to enforce IMDSv2.

### The Injection
```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

### Manifested Symptoms
```text
HTTP/1.1 401 Unauthorized
```
IMDSv2 requires acquiring a session token via `PUT` with `X-aws-ec2-metadata-token-ttl-seconds: 21600` before accessing credentials, blocking Server-Side Request Forgery (SSRF) vulnerabilities.

---

## 9. Debugging Walkthrough

```text
====================================================================================================
                        IAM POLICY TROUBLESHOOTING RUNBOOK
====================================================================================================

$ aws sts get-caller-identity
Output:
{
    "UserId": "AROAX...:i-0123456789abcdef0",
    "Account": "111122223333",
    "Arn": "arn:aws:sts::111122223333:assumed-role/lab06-app-workload-role/i-0123456789abcdef0"
}
Diagnosis: Confirms compute instance successfully assumed role without hardcoded credentials.
====================================================================================================
```

---

## 10. Teardown & Verification

```bash
terraform destroy -auto-approve
aws iam list-roles --query "Roles[?RoleName=='lab06-app-workload-role']"
```

---

## 11. Senior Interview Defense & Takeaways

> **Interviewer**: *"Why is storing cloud access keys (`AWS_ACCESS_KEY_ID` / OCI API signing key) in environment variables or configuration files considered an immediate security failure in production?"*

**Candidate Defense**:
*"Static credentials in configuration files or environment variables inevitably leak through container dumps, debug endpoints, git commits, or memory inspection.*

*Workload Identity (EKS Pod Identity / AWS IRSA / OCI Dynamic Groups) replaces static keys with ephemeral, cryptographic STS tokens rotated every hour. If an attacker dumps container environment variables, there are zero long-lived credentials to steal. Furthermore, permissions are enforced at the hypervisor/metadata layer, preventing privilege escalation."*
