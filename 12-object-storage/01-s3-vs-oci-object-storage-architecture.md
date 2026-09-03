# 01. S3 vs. OCI Object Storage Architecture

## 1. Problem
In traditional data storage architectures, scaling file systems to petabytes of unstructured data (videos, backups, analytical datasets, machine learning models) required expensive SAN/NAS appliances. These systems suffered from file-locking contention, inode limits, and rigid hierarchical directories that degraded under millions of files. Furthermore, naive web applications route massive file uploads (e.g., a 2 GB video) directly through application web servers, saturating CPU memory buffers, tying up web worker threads, and consuming expensive bandwidth. Hyperscale Object Storage solves this by providing a flat, infinitely scalable key-value storage fabric and delegated client-direct uploads.

## 2. Cloud Concept
### The Flat Key-Value Namespace
Unlike hierarchical file systems (POSIX) that organize data using folders, directories, and inodes, Object Storage is a flat **Key-Value Store**:
- **Bucket**: The top-level logical container for objects.
- **Key**: The unique string name of the object (e.g., `uploads/2026/09/invoice_4921.pdf`). While forward slashes (`/`) look like directories in the UI, to the storage engine, the slashes are simply literal characters in a flat alphanumeric string.
- **Value**: The raw binary byte payload (from 0 bytes up to 5 TB per object).
- **Metadata**: System metadata (Content-Type, ETag, last-modified) and custom user metadata key-values stored alongside the object.

### The Strong Consistency Model
Historically (prior to December 2020 in AWS), cloud object storage operated on an **Eventual Consistency** model for overwrite PUTs and DELETEs, requiring complex cache-invalidation logic in Big Data frameworks like Apache Spark and Hadoop.
- **Modern Strong Consistency (AWS S3 & OCI Object Storage)**:
  - Both AWS S3 and OCI Object Storage provide **Strong Read-after-Write Consistency** for `PUT` and `DELETE` requests of objects in all regions `[Doc: Amazon S3 Consistency Model, checked 2026-09-03]`.
  - The instant a `PUT` or `DELETE` request returns `HTTP 200 OK`, any subsequent `GET` or `LIST` request immediately returns the updated object or confirms its deletion. Zero lag, zero stale reads.

### Storage Tiers & Access Patterns
Cloud object storage provides differentiated storage tiers based on data access frequency and retrieval latency:

| Storage Tier | AWS S3 Tier | OCI Object Storage Tier | Retrieval Time | Minimum Duration | Best Use Case |
| :--- | :--- | :--- | :---: | :---: | :--- |
| **Hot (Active)** | S3 Standard | Standard | Milliseconds | None | Web assets, active analytics, mobile uploads |
| **Intelligent Auto** | S3 Intelligent-Tiering | Auto-Tiering | Milliseconds | None | Unpredictable or changing access patterns |
| **Warm (Infrequent)**| S3 Standard-IA | Infrequent Access | Milliseconds | 30 Days | Disaster recovery copies, monthly reports |
| **Cold (Archive)** | S3 Glacier Flexible | Archive Storage | Minutes to Hours | 90 Days | Regulatory compliance, historical audit logs |
| **Deep Freeze** | S3 Glacier Deep Archive | Archive Storage | 12 to 48 Hours | 180 Days | 7-to-10 year tape replacement backups |

### Delegated Authentication: Pre-Signed URLs vs. Pre-Authenticated Requests
Instead of proxying large file uploads through application compute instances, cloud providers allow generating temporary, cryptographically signed URLs:
- **AWS Pre-Signed URLs**:
  - The application backend generates a URL containing an IAM cryptographic signature (HMAC-SHA256) valid for a limited window (e.g., 15 minutes).
  - The client browser executes `HTTP PUT` directly to the S3 bucket endpoint using this URL. The backend server never handles the file payload.
- **OCI Pre-Authenticated Requests (PAR)**:
  - OCI's equivalent native primitive `[Doc: OCI Pre-Authenticated Requests, checked 2026-09-03]`.
  - Can be scoped to a single object or an entire bucket for read, write, or read/write access with strict expiration timestamps.

## 3. Mental Model
Think of object storage as a commercial coat check room:
- **POSIX Filesystems** is a maze of labeled filing cabinets. To find a coat, you must walk through Hallway A, open Drawer B, and find Folder C. If the filing cabinet fills up, you must buy a bigger cabinet.
- **Object Storage** is a coat check desk. You hand the attendant your heavy winter coat. The attendant hands you a plastic claim ticket (`Key = invoice_4921`). The attendant hangs the coat on an automated, infinite motorized carousel in a football-field-sized warehouse. When you return, you show your ticket, and the carousel immediately delivers your exact coat.
- **Pre-Signed URLs / PARs** is handing a customer a one-time guest pass so they can walk up to the coat check counter and drop off their luggage directly without you standing in line with them.

## 4. Architecture Diagram
```text
DELEGATED DIRECT-TO-OBJECT-STORAGE INGESTION ARCHITECTURE:

[Client Browser / Mobile App]
        │
        ├──► 1. POST /api/v1/upload-request (Small JSON payload)
        │
        ▼
[Application Server / Lambda / OCI Function]
        │  * Validates user authentication & permissions
        │  * Generates Pre-Signed URL / Pre-Authenticated Request
        │  * Validates file size limit & Content-Type
        ▼
[Client Browser] ◄── 2. Returns Signed URL: https://bucket.s3.amazonaws.com/uuid.mp4?X-Amz-Signature=...
        │
        │
        ▼ 3. Direct Binary Upload (HTTP PUT 2 GB Video Payload)
┌────────────────────────────────────────────────────────────────────────┐
│ CLOUD OBJECT STORAGE FABRIC (Amazon S3 / OCI Object Storage)           │
│                                                                        │
│   * Cryptographic Signature Verified at Cloud Edge                     │
│   * Payload streamed directly into storage nodes                       │
│   * Replicated across 3 Availability Domains / Fault Domains (11 9s)   │
│   * Triggers ObjectCreated Event ──► Worker Function (Transcoding)     │
└────────────────────────────────────────────────────────────────────────┘
  * APPLICATION SERVER HANDLES 0 BYTES OF VIDEO TRAFFIC!
  * SAVES 100% OF APPLICATION CPU, MEMORY, AND EGRESS BANDWIDTH!
```

## 5. AWS Implementation
In AWS Amazon S3:
- **Bucket Naming & Namespace**:
  - **Globally Unique Namespace**: An S3 bucket name must be unique across **all AWS accounts in all global regions worldwide**. Once an account claims `my-company-data`, no other entity on earth can use that bucket name.
- **Durability & Availability Guarantees**:
  - Engineered for **99.999999999% (11 9s) of data durability** across all storage tiers `[Doc: Amazon S3 Storage Classes, checked 2026-09-03]`.
  - Objects are stored redundantly across multiple physically separated Availability Zones within the region.
- **S3 Intelligent-Tiering**:
  - The ultimate hands-off cost optimization engine.
  - Automatically moves objects between three access tiers (Frequent, Infrequent, and Archive Instant Access) based on access patterns without performance impact or retrieval fees. Charges a small monthly monitoring fee per 1,000 objects.
- **S3 Block Public Access**:
  - An account-level and bucket-level centralized control that overrides all bucket policies and ACLs, guaranteeing that objects can never be exposed publicly.

## 6. OCI Implementation
In Oracle Cloud Infrastructure Object Storage:
- **Bucket Naming & Tenancy Namespaces**:
  - **MAJOR ARCHITECTURAL DISTINCTION FROM AWS**:
  - In OCI, bucket names are **NOT globally unique across all customers**.
  - OCI assigns each tenancy a unique, immutable string called the **Tenancy Namespace** (e.g., `ax924bjk10`) `[Doc: OCI Object Storage Namespaces, checked 2026-09-03]`.
  - Bucket names need only be unique **within your own tenancy namespace**. Two different OCI customers can both create a bucket named `production-backups` without collision!
  - REST URL format: `https://objectstorage.<region>.oraclecloud.com/n/<namespace>/b/<bucket-name>/o/<object-name>`.
- **OCI Storage Tiers**:
  - *Standard*: High-performance, low-latency tier for active access.
  - *Auto-Tiering*: Automatically transitions objects between Standard and Infrequent Access tiers based on consumption.
  - *Archive Storage*: Cold storage for compliance and disaster recovery. Minimum retention period: 90 days. Retrieval takes up to 4 hours.
- **Pre-Authenticated Requests (PAR)**:
  - Supports generating PARs via OCI Console, CLI, or SDK.
  - Can be generated for an individual object (Read, Write, or ReadWrite) or for an entire bucket (Write-only for upload dropboxes).
  - Can be revoked instantly via API at any time prior to expiration.

## 7. Configuration
Comparing secure bucket and delegated upload configuration in Terraform across AWS and OCI:

### AWS S3 Bucket with Block Public Access & KMS (Terraform)
```hcl
# Encrypted AWS S3 Bucket
resource "aws_s3_bucket" "secure_vault" {
  bucket = "enterprise-finance-data-vault-2026" # Globally unique!
}

# Enforce S3 Block Public Access (Account-level protection)
resource "aws_s3_bucket_public_access_block" "block_public" {
  bucket = aws_s3_bucket.secure_vault.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# KMS Server-Side Encryption (SSE-KMS)
resource "aws_s3_bucket_server_side_encryption_configuration" "kms_enc" {
  bucket = aws_s3_bucket.secure_vault.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = var.kms_key_arn
      sse_algorithm     = "aws:kms"
    }
  }
}
```

### OCI Object Storage Bucket with KMS Encryption (Terraform)
```hcl
# Fetch Tenancy Namespace
data "oci_objectstorage_namespace" "ns" {
  compartment_id = var.compartment_id
}

# OCI Object Storage Bucket
resource "oci_objectstorage_bucket" "secure_vault" {
  compartment_id = var.compartment_id
  name           = "finance-data-vault" # Unique within tenancy namespace only!
  namespace      = data.oci_objectstorage_namespace.ns.value
  storage_tier   = "Standard"

  # Enable Auto-Tiering for automatic cost optimization
  auto_tiering = "InfrequentAccess"

  # Customer Managed Key (CMK) Encryption via OCI Vault
  kms_key_id   = var.oci_vault_key_id
  access_type  = "NoPublicAccess"
}
```

## 8. Data Flow
```text
Direct-to-S3 Pre-Signed Upload Flow:
1. User clicks 'Upload Video' (2 GB) in web UI.
2. Web UI calls backend: POST /api/v1/uploads/generate-presigned-url
3. Backend authenticates user session:
   * Generates object key: uploads/user_99/video_123.mp4
   * Calls s3Client.getSignedUrl('putObject', { Expires: 900, ContentType: 'video/mp4' })
4. Backend returns signed URL to browser.
5. Browser executes native AJAX / fetch PUT to signed URL:
   PUT https://my-bucket.s3.amazonaws.com/uploads/user_99/video_123.mp4
   Headers: Content-Type: video/mp4
6. S3 validates HMAC signature and expiration:
   * Streams 2 GB payload directly to storage cluster.
   * Emits S3:ObjectCreated event to Amazon SQS / EventBridge.
7. Worker picks up event and initiates asynchronous video transcoding.
```

## 9. Security
- **Bucket Policies vs. IAM Policies**:
  - *IAM Policies*: Attached to users/roles; dictates what cloud identities can do across storage resources.
  - *Bucket Policies*: Attached directly to the bucket resource; dictates who (including anonymous users, cross-account identities, or specific CIDR blocks) can access that specific bucket.
- **Enforcing TLS In-Transit Encryption**:
  - Enforce SSL/TLS in-transit encryption using a bucket policy condition denying insecure HTTP requests:
    ```json
    "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    ```

## 10. Reliability
- **11 9s Durability Math**:
  - 99.999999999% durability means that if you store 10,000,000 objects in S3 or OCI Object Storage, you can statistically expect to lose **a single object once every 10,000 years**!
  - Achieved via Erasure Coding across independent hardware storage servers and multiple data centers.

## 11. Scaling
- **Massive Horizontal Scale**:
  - Object storage has no overall storage limit: customers store exabytes of data across billions of objects.
  - Automatic horizontal partitioning distributes request load across underlying storage clusters without customer intervention.

## 12. Observability
- **Storage Metrics & Access Logs**:
  - S3 Storage Lens: Provides organizational-wide visibility into object storage consumption, cost-optimization opportunities, and security posture.
  - Server Access Logging / S3 Data Events in CloudTrail: Records every individual `GetObject`, `PutObject`, and `DeleteObject` API call.

## 13. Cost
- **Storage Tier Pricing Comparison (per GB-month)**:
  - S3 Standard: **\$0.023 / GB** `[Doc: AWS S3 Pricing, checked 2026-09-03]`.
  - S3 Standard-IA: **\$0.0125 / GB** (45% savings, but adds \$0.01/GB retrieval fee).
  - S3 Glacier Flexible: **\$0.0036 / GB** (84% savings).
  - S3 Glacier Deep Archive: **\$0.00099 / GB** (95% savings!).
  - OCI Object Storage: **\$0.0255 / GB** for Standard; **\$0.0026 / GB** for Archive.
- *FinOps Rule*: Moving 100 TB of aging backups from Standard to Glacier Deep Archive slashes monthly storage costs from **\$2,300 to \$99**!

## 14. Failure Modes
- **The Accidental Public Data Leak (Misconfigured Bucket Policy)**: An administrator applies a bucket policy intended to allow cross-account access, but uses `"Principal": "*"` without strict condition keys. The entire bucket is indexed by public web scanners, exposing sensitive customer records to the internet. *Remediation: Enforce S3 Block Public Access globally.*
- **The Pre-Signed URL Content-Type Mismatch**: The backend generates a pre-signed URL with `ContentType: "image/jpeg"`. The client browser executes the PUT request with `Content-Type: "application/octet-stream"`. S3 rejects the request with `HTTP 403 Forbidden` (`SignatureDoesNotMatch`) because the HTTP header was included in the cryptographic signature calculation.

## 15. Troubleshooting
When pre-signed uploads fail with HTTP 403:
1. **Verify CORS Configuration**:
   - If uploading from a web browser, S3 / OCI Object Storage must have a **CORS (Cross-Origin Resource Sharing)** policy allowing `PUT` methods from the frontend domain (`https://app.mycompany.com`).
2. **Inspect Exact HTTP Headers**:
   - Any header included during URL signing (e.g., `x-amz-server-side-encryption`, `Content-Type`) must be sent by the client with the exact same value.
3. **Verify Generator IAM Permissions**:
   - A pre-signed URL can only perform actions that the **generating IAM identity** possesses at the time of the request! If the backend role lacks `s3:PutObject`, the pre-signed URL will fail, even if the URL itself is cryptographically valid.

## 16. Common Mistakes
- **Proxying Uploads Through Application Servers**: Accepting multipart form uploads on an EC2/Compute instance and uploading them to S3 in serial. This wastes compute RAM, bottlenecks network bandwidth, and caps maximum file sizes. Always use **Pre-Signed URLs / PARs**.
- **Forgetting Retrieval Fees on Infrequent Access Tiers**: Moving highly active data to S3 Standard-IA to "save money". Every read operation incurs per-GB retrieval fees, doubling total storage costs.

## 17. Trade-offs
| Storage Paradigm | Latency / Throughput | Cost per GB | Scalability | Access Protocol |
| :--- | :--- | :--- | :--- | :--- |
| **Object Storage (S3 / OCI)** | 50–100ms latency; Gigabytes/s | Ultra-Low (\$0.001–\$0.023) | Virtually Infinite | REST API (HTTP GET/PUT) |
| **Block Storage (EBS / BV)** | Sub-millisecond; 256,000 IOPS | Moderate (\$0.08–\$0.12) | Up to 64 TB per volume | Direct OS block device |
| **File Storage (EFS / FSS)** | 1–3ms; Concurrent multi-mount | High (\$0.30) | Elastic petabytes | NFS v4.1 network mount |

## 18. Interview Questions
1. *Your organization operates a video-sharing platform where users upload 500 MB to 5 GB video files. Why is routing these uploads through your web application servers an anti-pattern, and how do you redesign the architecture using S3 Pre-Signed URLs or OCI Pre-Authenticated Requests?*
2. *What is the fundamental architectural difference between bucket namespaces in AWS S3 versus OCI Object Storage?*
3. *Explain the Strong Consistency model implemented in Amazon S3 and OCI Object Storage. What was the legacy eventual consistency behavior, and why was it problematic for data engineering pipelines?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "Routing large video uploads directly through web application servers is a critical architectural anti-pattern for three major reasons:
>
> 1. **The Web Server Bottleneck**:
>    - Large file uploads consume web server thread pools and socket buffers for extended periods. A client on a slow mobile connection uploading a 2 GB video holds a web worker thread hostage for 10 minutes.
>    - This leads to web thread exhaustion, requiring organizations to over-provision expensive compute clusters simply to buffer network packets.
>    - Furthermore, data is transferred twice: once from client to server, and once from server to S3, incurring double bandwidth processing costs.
>
> 2. **The Direct-to-Object-Storage Redesign**:
>    - **Step 1 (Token Request)**: The client browser sends a lightweight JSON request (`POST /api/v1/uploads`) containing metadata (file name, file size, mime type) to our backend API.
>    - **Step 2 (Delegated Authorization)**: The backend authenticates the user, verifies upload quotas, and generates an **Amazon S3 Pre-Signed URL** (or **OCI Pre-Authenticated Request**) using the cloud SDK with a strict 15-minute expiration and locked `Content-Type: video/mp4`.
>    - **Step 3 (Direct S3 Ingest)**: The backend returns the signed URL to the browser. The client browser uses JavaScript to execute an HTTP `PUT` directly to the S3 bucket endpoint, streaming the 2 GB video payload across AWS edge infrastructure directly into S3.
>    - **Step 4 (Asynchronous Processing)**: Upon completion, S3 emits an `s3:ObjectCreated` event to Amazon EventBridge / SQS. A serverless worker task picks up the message and triggers an AWS Elemental MediaConvert or container transcoding pipeline.
>
> 3. **The Architectural Benefit**:
>    - Application compute handles **0 bytes** of heavy video traffic.
>    - Web servers remain 100% available to serve interactive API calls, slashing compute costs by 80% while scaling to millions of concurrent uploads seamlessly."

## 20. Hands-on Exercise
**Objective**: Generate a Pre-Signed URL using AWS CLI and upload a file directly to Amazon S3.

### Verification Steps
1. Create an S3 bucket with Block Public Access enabled.
2. Generate a Pre-Signed URL for an object upload valid for 300 seconds:
   ```bash
   aws s3 presign s3://<bucket-name>/direct-upload.txt --expires-in 300
   ```
3. From an external terminal without AWS credentials configured, upload a file via `curl`:
   ```bash
   curl -X PUT -T local_file.txt "<presigned-url>"
   ```
4. Verify via AWS CLI: `aws s3 ls s3://<bucket-name>/`, confirming that the object was uploaded successfully through delegated authentication.
