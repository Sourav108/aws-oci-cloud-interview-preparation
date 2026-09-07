# Reference Project 03: Global Media Streaming & Transcoding Pipeline

---

## 1. Executive Summary & Architecture Overview

This production reference architecture delivers a globally distributed, high-throughput media upload, automated malware scanning, hardware-accelerated video transcoding, and CDN streaming distribution pipeline capable of ingesting **500 TB/day** of high-definition video from content creators worldwide.

Key Architectural Capabilities:
- **Zero-Proxy Edge Ingestion**: Creators bypass application compute, uploading multi-part video chunks directly to cloud object storage via short-lived, cryptographically signed pre-signed URLs / Pre-Authenticated Requests (PAR).
- **Automated Quarantine & Malware Screening**: Raw uploaded media is isolated in an air-gapped quarantine storage bucket where automated container sandboxes (ClamAV / GuardDuty) verify file integrity before triggering downstream pipelines.
- **Hardware-Accelerated GPU Transcoding Fleet**: Amazon EKS and OCI Kubernetes Engine (OKE) worker pods equipped with NVIDIA Tensor Core GPUs execute parallelized FFmpeg transcoding into multi-bitrate HLS/DASH streams.
- **Global CDN Video Delivery**: Transcoded HLS segments and manifests are cached and distributed globally via Amazon CloudFront and OCI Content Delivery Network.

---

## 2. Dual-Cloud System Topology

```
========================================================================================================================
                                 GLOBAL MEDIA STREAMING PIPELINE TOPOLOGY
========================================================================================================================

                                       [ Content Creators (Worldwide) ]
                                                       │
                                   ┌───────────────────┴───────────────────┐
                                   │ (1. POST /uploads/init - Metadata)    │ (2. PUT 16MB Chunks via Presigned URL)
                                   ▼                                       ▼
             [ Edge DNS / Anycast: Route 53 / OCI DNS ]    [ Edge POP: S3 Transfer Acceleration / OCI CDN Edge ]
                                   │                                       │ (Cloud Private Backbone)
                                   ▼                                       ▼
             [ Regional Ingress API: ALB / OCI LB ]        [ QUARANTINE OBJECT STORAGE BUCKET ]
                                   │                         - AWS: S3 Standard (Hex Hash Sharded)
                                   ▼                         - OCI: Object Storage (Tiered Compartment)
             [ Control Plane Pods: EKS / OKE ]                             │
              - Generate Presigned Chunk URLs                              │ (ObjectCreated Notification)
              - Track Parts in DynamoDB / OCI NoSQL                        ▼
                                                           [ EVENT BUS: EventBridge / OCI Events ]
                                                                           │
                                                                           ▼
                                                   [ WORKER ORCHESTRATION: Step Functions / OKE ]
                                                                           │
                                           ┌───────────────────────────────┴───────────────────────────────┐
                                           │ (Malware & Content Safety)                                    │ (Adaptive Bitrate Transcoding)
                                           ▼                                                               ▼
                             [ ClamAV / GuardDuty Sandbox ]                                  [ GPU Transcode Fleet: EKS / OKE ]
                                           │                                                  - FFmpeg on NVIDIA Tensor Core
                                           ├── Malicious: Quarantine & Alert                  - Package HLS / MPEG-DASH
                                           │                                                               │
                                           └── Clean: Move to Production Bucket ───────────────────────────┘
                                                                           │
                                                                           ▼
                                                          [ PRODUCTION CDN DISTRIBUTION ]
                                                          - Amazon CloudFront / OCI CDN
                                                          - Global Streaming Video Delivery to Audiences
========================================================================================================================
```

---

## 3. Dual-Cloud Component Mapping Matrix

| Architectural Function | AWS Cloud Implementation | OCI Cloud Implementation | Engineering Justification |
| :--- | :--- | :--- | :--- |
| **Edge Ingress Acceleration**| S3 Transfer Acceleration `[Doc: S3, checked 2026]` | OCI CDN Edge with Pre-Authenticated Requests `[Doc: OCI Storage, checked 2026]` | Bypasses public internet routing; terminates TCP at local edge and routes over cloud backbone. |
| **Direct-to-Storage Token** | S3 Pre-Signed URLs (PartNumber PUT) | OCI Pre-Authenticated Requests (PAR) | Allows clients to write directly to storage without exposing IAM credentials or proxying via compute. |
| **Quarantine Storage** | Amazon S3 Standard (WORM Object Lock) | OCI Object Storage Standard (Retention Rules) | Enforces strict quarantine isolation on unverified incoming bytes. |
| **Event Routing Fabric** | Amazon EventBridge + S3 Event Notifications | OCI Events Service + OCI Streaming | Triggers decoupled asynchronous processing upon multi-part upload completion. |
| **GPU Transcoding Fleet** | Amazon EKS with G5 GPU Nodes (`g5.2xlarge`) | OKE with NVIDIA GPU Shapes (`VM.GPU.A10.1`) | Hardware-accelerated NVENC video transcoding at 60% lower unit cost using spot/preemptible instances. |
| **Upload State Store** | Amazon DynamoDB (On-Demand) | OCI NoSQL Database | Stores chunk upload manifests, ETags, and resumable session progress. |

---

## 4. Production Infrastructure as Code (Terraform HCL)

### 4.1 AWS Terraform Module (`aws_media_pipeline.tf`)

```hcl
# AWS Reference Implementation: S3 Ingestion Bucket with Transfer Acceleration
resource "aws_s3_bucket" "media_quarantine" {
  bucket        = "prod-enterprise-media-quarantine"
  force_destroy = false

  tags = {
    Environment = "production"
    Tier        = "quarantine"
  }
}

resource "aws_s3_bucket_accelerate_configuration" "s3_accel" {
  bucket = aws_s3_bucket.media_quarantine.id
  status = "Enabled" # Enables S3 Transfer Acceleration globally
}

resource "aws_s3_bucket_server_side_encryption_configuration" "s3_enc" {
  bucket = aws_s3_bucket.media_quarantine.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.media_key.arn
      sse_algorithm     = "aws:kms"
    }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "media_lifecycle" {
  bucket = aws_s3_bucket.media_quarantine.id

  rule {
    id     = "auto-tiering-and-cleanup"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    abort_incomplete_multipart_upload {
      days_after_initiation = 3
    }
  }
}
```

### 4.2 OCI Terraform Module (`oci_media_pipeline.tf`)

```hcl
# OCI Reference Implementation: Object Storage Bucket with Auto-Tiering
resource "oci_objectstorage_bucket" "media_quarantine" {
  compartment_id = var.compartment_ocid
  name           = "prod_enterprise_media_quarantine"
  namespace      = var.object_storage_namespace
  storage_tier   = "Standard"
  auto_tiering   = "InfrequentAccess" # Automated transition to cold storage

  kms_key_id     = oci_kms_key.vault_media_key.id

  versioning     = "Enabled"
}

resource "oci_objectstorage_object_lifecycle_policy" "quarantine_lifecycle" {
  bucket    = oci_objectstorage_bucket.media_quarantine.name
  namespace = var.object_storage_namespace

  rules {
    name        = "ArchiveOldFootage"
    action      = "ARCHIVE"
    time_amount = 90
    time_unit   = "DAYS"
    is_enabled  = true
    target      = "objects"
  }
}
```

---

## 5. Security, Workload Identity & Encryption

### 5.1 Pre-Signed Token Security Constraints
- S3 Pre-Signed URLs and OCI PAR tokens enforce a maximum **15-minute time-to-live (TTL)** per chunk.
- Tokens enforce mandatory MD5 checksum verification headers (`Content-MD5`) and restrict uploads to designated object key paths.
- IAM execution roles used by transcoding worker pods possess read-only permissions on the quarantine bucket and write-only permissions on the production distribution bucket.

---

## 6. Observability, SLIs/SLOs & Alerting Runbook

1. **Ingress Upload Throughput**: Ingest bandwidth in Mbps. Alert if regional median drops $< 50\text{ Mbps}$.
2. **Chunk 5xx Error Rate**: Rate of S3/Object Storage failures. Alert if $> 0.1\%$.
3. **Transcode Queue Delay**: Oldest video pending transcoding. Alert if $> 15\text{ minutes}$.

---

## 7. Deployment & Verification Runbook

```bash
# 1. Initialize Multipart Upload
INIT_RESPONSE=$(curl -s -X POST https://${API_URL}/v1/uploads/init   -H "Authorization: Bearer ${JWT_TOKEN}"   -H "Content-Type: application/json"   -d '{"filename": "video_raw.mp4", "size_bytes": 104857600, "checksum_sha256": "..."}')

# 2. Extract Pre-Signed Chunk URL and Upload Directly to S3/OCI
UPLOAD_URL=$(echo $INIT_RESPONSE | jq -r '.part_urls[0]')
curl -X PUT "${UPLOAD_URL}"   -H "Content-Type: video/mp4"   --data-binary "@chunk_001.part"

# 3. Finalize Multipart Upload
curl -X POST https://${API_URL}/v1/uploads/complete   -H "Authorization: Bearer ${JWT_TOKEN}"   -d '{"upload_id": "...", "parts": [{"part_number": 1, "etag": "..."}]}'
```

---

## 8. FinOps Cost Breakdown & Sizing Economics

```text
====================================================================================================
                        FINOPS COST BREAKDOWN (500 TB/DAY INGESTION)
====================================================================================================

TIER                             AWS MONTHLY COST        OCI MONTHLY COST
----------------------------------------------------------------------------------------------------
Ingress Acceleration             $6,000                  $0.00 (Standard edge included)
Hot Object Storage (15 PB)       $345,000                $382,500
Lifecycle Tiering Reductions     -$160,000               -$185,000
GPU Transcoding Fleet (Spot)     $48,000                 $26,000
Control Plane & Telemetry        $4,200                  $2,800
----------------------------------------------------------------------------------------------------
TOTAL RUN-RATE                   $243,200 / month        $226,300 / month
STORAGE COST PER RAW GB          ~$0.016 / GB            ~$0.015 / GB
====================================================================================================
```

---

## 9. Failure Mode Drills & Chaos Engineering Runbook

### 9.1 Game Day Drill: Storage Throttling 503 Mitigation
1. **Action**: Simulate 20,000 concurrent client chunk PUTs against a single storage prefix.
2. **Verification**:
   - Client upload SDK intercepts HTTP 503 SlowDown and initiates full jitter exponential backoff.
   - Dynamic hash-prefixing automatically disperses traffic across partition shards.
   - Upload success rate remains $> 99.9\%$.
