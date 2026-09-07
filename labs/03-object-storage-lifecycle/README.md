# Lab 03: Object Storage Lifecycle & Customer-Managed Encryption

---

## 1. Goal & Core Concept

The objective of this hands-on lab is to configure, secure, and validate enterprise cloud object storage across **Amazon S3** and **OCI Object Storage**.

### Core Architectural Concepts Tested
- **Customer-Managed Key Encryption (CMEK)**: Enforcing hardware-backed envelope encryption using AWS KMS and OCI Vault.
- **Automated Lifecycle Tiering**: Transitioning objects from Standard (Hot) to Infrequent Access (Warm) to Glacier / Archive (Cold) storage tiers.
- **Secure Delegation via Temporary Tokens**: Generating short-lived S3 Pre-Signed URLs and OCI Pre-Authenticated Requests (PAR) for zero-proxy data uploads.
- **Bucket Policy Guardrails**: Enforcing TLS 1.3 transit encryption (`aws:SecureTransport`) and preventing accidental public bucket exposure.

---

## 2. Cost Warning & Resource Estimate

> [!NOTE]
> **ESTIMATED HOURLY RUN COST: `< $0.02 / hour`**
>
> | Cloud Provider | Resource Component | Quantity | Hourly Rate | Run Cost (2 Hours) |
> | :--- | :--- | :---: | :---: | :---: |
> | **AWS** | S3 Standard Storage (< 1 GB test data) `[Doc: S3, checked 2026]` | 1 Bucket | Negligible ($0.023/GB-mo) | $0.001 |
> | **AWS** | AWS KMS Customer Managed Key (CMK) `[Doc: KMS, checked 2026]` | 1 Key | $1.00 / month ($0.0014/hr) | $0.003 |
> | **OCI** | OCI Object Storage (< 1 GB test data) `[Doc: OCI Storage, checked 2026]` | 1 Bucket | Negligible ($0.0255/GB-mo)| $0.001 |
> | **OCI** | OCI Vault Master Encryption Key `[Doc: OCI Vault, checked 2026]` | 1 Key | $0.00 (Standard Tier Free)| $0.000 |
>
> *Total Estimated Lab Cost for a 2-hour session: **$0.005 USD**.*
> *In No-Live-Account Fallback Mode, cost is **$0.00**.*

---

## 3. Architecture Diagram

```
========================================================================================================================
                          OBJECT STORAGE LIFECYCLE & ENCRYPTION TOPOLOGY
========================================================================================================================

  [ Client Application / User ]
                │
                ├── 1. Request Upload Token ────────► [ Control Plane API (EKS / OKE) ]
                │                                                    │
                │◄── 2. Return Presigned URL / PAR ──────────────────┤ (Valid for 15 mins)
                │
                ▼ (3. Direct-to-Storage PUT over TLS 1.3)
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  OBJECT STORAGE BUCKET (AWS S3 / OCI Object Storage)
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   BUCKET POLICY GUARDRAILS:
   - Deny if aws:SecureTransport == false (Enforces HTTPS)
   - Deny if s3:x-amz-server-side-encryption != "aws:kms"
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   ENVELOPE ENCRYPTION TIER:
   - AWS KMS Customer Managed Key (CMK) / OCI Vault Master Encryption Key
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   LIFECYCLE TIERING RULES:
   - Day 0 to 30:   Standard Tier (Hot Access, Sub-millisecond latency)
   - Day 31 to 90:  Infrequent Access Tier (S3 Standard-IA / OCI Infrequent Access - 50% cost savings)
   - Day 91+:       Archive / Glacier Tier (S3 Glacier / OCI Archive - 85% cost savings)
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
```

---

## 4. Prerequisites

1. Terraform CLI installed.
2. Cloud credentials configured with permissions to create S3/OCI buckets and KMS/Vault keys.

---

## 5. Infrastructure Code (Terraform HCL)

### 5.1 AWS S3 & KMS Implementation (`aws_storage.tf`)

```hcl
# AWS Reference Implementation: S3 Bucket with KMS CMEK and Lifecycle Rules
resource "aws_kms_key" "s3_cmk" {
  description             = "Customer Managed Key for S3 Bucket"
  deletion_window_in_days = 7
  enable_key_rotation     = true
  tags                    = { Environment = "lab" }
}

resource "aws_s3_bucket" "secure_bucket" {
  bucket        = "lab03-enterprise-storage-${var.random_suffix}"
  force_destroy = true
}

resource "aws_s3_bucket_public_access_block" "block_public" {
  bucket = aws_s3_bucket.secure_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "kms_encryption" {
  bucket = aws_s3_bucket.secure_bucket.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.s3_cmk.arn
      sse_algorithm     = "aws:kms"
    }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "bucket_lifecycle" {
  bucket = aws_s3_bucket.secure_bucket.id

  rule {
    id     = "tier-to-ia-and-glacier"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }
}
```

### 5.2 OCI Object Storage & Vault Implementation (`oci_storage.tf`)

```hcl
# OCI Reference Implementation: Object Storage with Vault CMEK and Lifecycle
resource "oci_kms_vault" "lab_vault" {
  compartment_id = var.compartment_ocid
  display_name   = "lab03-security-vault"
  vault_type     = "DEFAULT"
}

resource "oci_kms_key" "storage_key" {
  compartment_id = var.compartment_ocid
  display_name   = "lab03-storage-key"
  vault_id       = oci_kms_vault.lab_vault.id

  key_shape {
    algorithm = "AES"
    length    = 32
  }
  management_endpoint = oci_kms_vault.lab_vault.management_endpoint
}

resource "oci_objectstorage_bucket" "secure_bucket" {
  compartment_id = var.compartment_ocid
  name           = "lab03_enterprise_storage_${var.random_suffix}"
  namespace      = var.object_storage_namespace
  storage_tier   = "Standard"
  kms_key_id     = oci_kms_key.storage_key.id
  auto_tiering   = "InfrequentAccess"
}

resource "oci_objectstorage_object_lifecycle_policy" "storage_lifecycle" {
  bucket    = oci_objectstorage_bucket.secure_bucket.name
  namespace = var.object_storage_namespace

  rules {
    name        = "ArchiveAfter90Days"
    action      = "ARCHIVE"
    time_amount = 90
    time_unit   = "DAYS"
    is_enabled  = true
    target      = "objects"
  }
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

Testing Direct Upload via Pre-Signed URL:
$ aws s3 presign s3://lab03-enterprise-storage-xxxx/test.txt --expires-in 900
https://lab03-enterprise-storage-xxxx.s3.amazonaws.com/test.txt?AWSAccessKeyId=...

$ curl -X PUT -T "payload.txt" "${PRESIGNED_URL}"
HTTP/1.1 200 OK
x-amz-server-side-encryption: aws:kms
x-amz-server-side-encryption-aws-kms-key-id: arn:aws:kms:us-east-1:111122223333:key/...
```

---

## 8. Failure Injection Drill: Plaintext HTTP Egress Attempt

### The Scenario
Attempt to upload an object to the bucket over insecure HTTP instead of HTTPS (TLS).

### The Injection
```bash
curl -X PUT http://${BUCKET_NAME}.s3.amazonaws.com/insecure.txt -d "test"
```

### Manifested Symptoms
```text
HTTP/1.1 403 Forbidden
<Error>
  <Code>AccessDenied</Code>
  <Message>Access Denied: EnforceTLSRequestsOnly</Message>
</Error>
```

---

## 9. Debugging Walkthrough

```text
====================================================================================================
                        KMS DECRYPT AUTHORIZATION DIAGNOSIS
====================================================================================================

Symptom: Downstream microservice receives 'AccessDeniedException' when reading S3 object.
Step 1: Check S3 Bucket Policy
  - Bucket policy allows 's3:GetObject' to the microservice IAM role.
Step 2: Inspect CloudTrail Event
  - EventName: 'kms:Decrypt'
  - ErrorCode: 'AccessDenied'
  - Message: 'User: arn:aws:iam::... is not authorized to perform: kms:Decrypt on resource: key-xxxx'
Root Cause: S3 permissions were granted, but the KMS Key Policy lacked the microservice IAM principal!
Fix: Add microservice IAM role ARN to the KMS Key Policy's 'Allow' statement.
====================================================================================================
```

---

## 10. Teardown & Verification

```bash
terraform destroy -auto-approve
aws s3 ls | grep lab03
```

---

## 11. Senior Interview Defense & Takeaways

> **Interviewer**: *"What is the minimum object size and duration penalty when tiering objects to S3 Standard-Infrequent Access or OCI Infrequent Access?"*

**Candidate Defense**:
*"Tiering small objects to Infrequent Access is an architectural cost trap. In AWS S3 Standard-IA, objects smaller than 128 KB are billed at the full 128 KB minimum storage rate, and objects are subject to a mandatory 30-day minimum billing duration.*

*If an application writes millions of 2 KB log files and tiers them to IA after 7 days, the storage bill actually multiplies by 64x due to the 128 KB padding! Best practice dictates compacting small records into larger files (e.g., 64 MB Parquet files) or using S3 Intelligent-Tiering / OCI Auto-Tiering to avoid small-object penalties."*
