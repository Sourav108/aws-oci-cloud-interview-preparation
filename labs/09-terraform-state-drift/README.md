# Lab 09: Infrastructure as Code, Remote State Locking & Drift Detection

---

## 1. Goal & Core Concept

The objective of this hands-on lab is to configure enterprise-grade **Terraform Remote State Backends** with mutual-exclusion state locking (AWS S3 + DynamoDB vs. OCI Object Storage native state locking), inject manual out-of-band infrastructure drift, and execute automated state reconciliation.

### Core Architectural Concepts Tested
- **Remote State Storage & Security**: Storing state files in an encrypted, versioned object store with least-privilege IAM controls.
- **State Locking Mechanics**: Preventing concurrent executions and race conditions using Amazon DynamoDB lock tables and OCI native lock files.
- **Out-of-Band Configuration Drift**: Detecting when an operator manually modifies resources via the cloud console or CLI.
- **Reconciliation & `terraform refresh`**: Synchronizing state and safely planning changes to eliminate drift.

---

## 2. Cost Warning & Resource Estimate

> [!NOTE]
> **ESTIMATED HOURLY RUN COST: `< $0.05 / hour`**
>
> | Cloud Provider | Resource Component | Quantity | Hourly Rate | Run Cost (2 Hours) |
> | :--- | :--- | :---: | :---: | :---: |
> | **AWS** | S3 State Bucket + DynamoDB Lock Table `[Doc: DynamoDB, checked 2026]` | 1 S3 + 1 Table | Free Tier (25 RCU/WCU free)| $0.00 |
> | **AWS** | Test Security Group / VPC | 1 Resource | $0.00 (Free) | $0.00 |
> | **OCI** | OCI Object Storage Bucket (State Backend) `[Doc: OCI Storage, checked 2026]` | 1 Bucket | Free Tier | $0.00 |
>
> *Total Estimated Lab Cost for a 2-hour session: **$0.00 USD**.*
> *In No-Live-Account Fallback Mode, cost is **$0.00**.*

---

## 3. Architecture Diagram

```
========================================================================================================================
                          TERRAFORM REMOTE STATE & LOCKING ARCHITECTURE
========================================================================================================================

  [ Developer / CI Pipeline A ]                     [ Developer / CI Pipeline B (Concurrent) ]
                │                                                     │
                ├── 1. Acquire Lock (LockID) ───┐                     │
                │                               ▼                     │
  ══════════════════════════════════════════════════════════════════  │
  LOCK MANAGER (AWS DynamoDB Table / OCI Object Lock)                 │
  - Table: terraform-state-locks                                      │
  - LockID: "prod/terraform.tfstate-md5"                              │
  ══════════════════════════════════════════════════════════════════  │
                │                                                     ├── 2. Attempt Acquire Lock ──► [ REJECTED ]
                ▼                                                     │   "Error: Error acquiring the state lock!"
  STATE STORAGE (Amazon S3 Bucket / OCI Object Storage)               │
  - Bucket: prod-terraform-remote-state (Versioning Enabled)          │
  - Server-Side Encryption: AWS KMS / OCI Vault                       │
  ══════════════════════════════════════════════════════════════════
                │
                ▼
  [ 3. Read/Write State File & Release Lock ]
========================================================================================================================
```

---

## 4. Prerequisites

1. Terraform CLI v1.8+.
2. Cloud credentials with permissions to manage S3, DynamoDB, and OCI Object Storage.

---

## 5. Infrastructure Code (Terraform HCL)

### 5.1 AWS S3 Backend with DynamoDB Locking (`backend_aws.tf`)

```hcl
# AWS Reference Implementation: S3 State Bucket + DynamoDB Lock Table
resource "aws_s3_bucket" "state_bucket" {
  bucket        = "lab09-terraform-state-${var.account_id}"
  force_destroy = true
}

resource "aws_s3_bucket_versioning" "state_versioning" {
  bucket = aws_s3_bucket.state_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_dynamodb_table" "state_locks" {
  name         = "lab09-terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

# Example Backend Declaration in root module
terraform {
  backend "s3" {
    bucket         = "lab09-terraform-state-xxxx"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "lab09-terraform-locks"
    encrypt        = true
  }
}
```

### 5.2 OCI Object Storage Native HTTP Backend (`backend_oci.tf`)

```hcl
# OCI Reference Implementation: Object Storage Native HTTP State Backend
terraform {
  backend "http" {
    address       = "https://objectstorage.us-ashburn-1.oraclecloud.com/p/xxx/n/namespace/b/tf_state_bucket/o/terraform.tfstate"
    update_method = "PUT"
  }
}
```

---

## 6. Step-by-Step Deployment Guide

```bash
terraform init
terraform validate
terraform plan
```

---

## 7. Expected Validation Results

```text
[Statically validated — not applied to a live account]

Testing Concurrent Lock Acquisition:
$ terraform apply &
$ terraform apply
Error: Error acquiring the state lock

Error message: ConditionalCheckFailedException: The conditional request failed
Lock Info:
  ID:        9a12c48e-32b0-4f51-87a3-48e02d91bc21
  Path:      lab09-terraform-state-xxxx/global/s3/terraform.tfstate
  Operation: OperationTypeApply
  Who:       ci-runner-42@enterprise.com
```

---

## 8. Failure Injection Drill: Manual Security Group Drift

### The Scenario
An engineer during an outage manually edits a production Security Group via the cloud console, opening port 22 (SSH) to `0.0.0.0/0`.

### The Injection
```bash
aws ec2 authorize-security-group-ingress   --group-id sg-0123456789abcdef0   --protocol tcp --port 22 --cidr 0.0.0.0/0
```

### Manifested Symptoms
- The infrastructure is now in an unmanaged, dangerous security state.
- `terraform plan` detects configuration drift:
```text
~ resource "aws_security_group" "web_sg" {
    # (3 unchanged attributes hidden)

  - ingress {
      - cidr_blocks = ["0.0.0.0/0"]
      - from_port   = 22
      - protocol    = "tcp"
      - to_port     = 22
    }
}
Plan: 0 to add, 1 to change, 0 to destroy.
```

---

## 9. Debugging Walkthrough

```text
====================================================================================================
                        STATE RECONCILIATION & CORRUPTION RECOVERY
====================================================================================================

Step 1: Check Current Remote State
  $ terraform state show aws_security_group.web_sg

Step 2: Remediate Drift back to Desired State
  $ terraform apply -auto-approve
  Output:
  aws_security_group.web_sg: Modifying... [id=sg-xxxx]
  aws_security_group.web_sg: Modifications complete after 1s
  Result: Ingress rule for port 22 is automatically revoked!
====================================================================================================
```

---

## 10. Teardown & Verification

```bash
terraform destroy -auto-approve
```

---

## 11. Senior Interview Defense & Takeaways

> **Interviewer**: *"What happens if a Terraform process is abruptly killed (`kill -9`) mid-apply while holding a DynamoDB state lock?"*

**Candidate Defense**:
*"If a CI worker crashes or is abruptly terminated while applying, the DynamoDB lock record remains stuck in the table. Any subsequent run will abort immediately with `Error: Error acquiring the state lock`.*

*To safely recover without state corruption: first, verify that no actual processes are running against the state file. Next, inspect the lock ID reported in the error message, and execute `terraform force-unlock <LOCK_ID>`. Finally, run `terraform refresh` or `terraform plan` to audit whether partial resources were provisioned before the crash and reconcile state."*
