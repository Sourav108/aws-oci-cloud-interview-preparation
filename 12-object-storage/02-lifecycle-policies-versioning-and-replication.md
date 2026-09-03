# 02. Lifecycle Policies, Versioning & Cross-Region Replication

## 1. Problem
Without automated data lifecycle governance, enterprise cloud object storage costs grow monotonically into an uncontrollable financial liability. Organizations frequently store terabytes of temporary log files, ephemeral build artifacts, and abandoned multipart upload chunks for years at expensive Standard storage tier rates ($23 per TB/month). Furthermore, accidental file deletions or malicious ransomware attacks can permanently destroy business-critical datasets if versioning and replication controls are missing or improperly configured.

## 2. Cloud Concept
### Object Storage Governance Primitives
- **Object Versioning**:
  - Maintains multiple distinct variants of an object in the same bucket.
  - Generates a unique **Version ID** for every `PUT` operation on a key.
  - *Delete Marker Mechanics*: When an object is deleted from a version-enabled bucket, the object is **not permanently erased**. Instead, the storage engine inserts a **Delete Marker** as the current version. Previous versions remain fully intact and can be restored simply by deleting the delete marker.
  - *Permanent Deletion*: Only an explicit delete request specifying both the object key **and** the specific `VersionId` permanently purges the binary payload from disk.
- **Lifecycle Management Policies**:
  - Automated rules evaluated daily by the cloud storage engine to manage data transitions and expirations without application code changes:
    1. *Transition Actions*: Automatically demote aging objects to colder, cheaper storage tiers based on age (e.g., transition to Infrequent Access at 30 days, Glacier at 90 days, Deep Archive at 180 days).
    2. *Expiration Actions*: Permanently delete objects or expire noncurrent versions after a specified retention window (e.g., permanently delete previous versions after 90 days).
    3. *Abort Incomplete Multipart Uploads*: Automatically purges stranded, incomplete multipart upload chunks after $N$ days.
- **Cross-Region Replication (CRR)**:
  - Asynchronously copies objects, metadata, and tags from a source bucket in one region to a destination bucket in another region.
  - Provides geo-redundancy to survive regional cataclysms and satisfies strict compliance and disaster recovery mandates (RPO $< 15\text{ minutes}$).

## 3. Mental Model
Think of object storage lifecycle management as historical archive preservation:
- **Versioning** is keeping every draft of a legal contract in a physical filing folder. When someone scribbles "CANCELLED" across the top (Delete Marker), the previous signed drafts remain safe underneath in the folder.
- **Lifecycle Policies** is an automated office archivist: every morning, the archivist checks the dates on the files. Files 1 month old are moved from the desk to the basement storage (Infrequent Access). Files 1 year old are shipped to a salt mine underground vault (Deep Archive). Files 7 years old are thrown into the industrial shredder (Expiration).
- **Cross-Region Replication** is having every single signed document photocopied and flown via courier to a secondary vault on another continent every night.

## 4. Architecture Diagram
```text
OBJECT STORAGE LIFECYCLE DEMOTION & REPLICATION PIPELINE:

[Application Uploads: invoice.pdf (Day 0)]
              │
              ├──► Synchronous Write to PRIMARY BUCKET (us-east-1)
              │    (S3 Standard / OCI Standard: $0.023/GB)
              │
              ▼ Asynchronous Cross-Region Replication (CRR)
┌────────────────────────────────────────────────────────┐
│ DESTINATION REPLICATION BUCKET (eu-central-1)          │
│ (Survives complete US East Coast regional failure!)    │
└────────────────────────────────────────────────────────┘
              │
              ▼ Lifecycle Rule 1: At Day 30
┌────────────────────────────────────────────────────────┐
│ INFREQUENT ACCESS TIER (Standard-IA / OCI Infrequent)  │
│ (Cost: $0.0125/GB - 45% Savings! Millisecond retrieval)│
└─────────────────────────────┬──────────────────────────┘
                              │
                              ▼ Lifecycle Rule 2: At Day 90
┌────────────────────────────────────────────────────────┐
│ ARCHIVE TIER (S3 Glacier / OCI Archive)                │
│ (Cost: $0.0036/GB - 84% Savings! 3-5 hour retrieval)   │
└─────────────────────────────┬──────────────────────────┘
                              │
                              ▼ Lifecycle Rule 3: At Day 365
┌────────────────────────────────────────────────────────┐
│ DEEP ARCHIVE (S3 Glacier Deep Archive)                 │
│ (Cost: $0.00099/GB - 95% Savings! 12-hour retrieval)   │
└─────────────────────────────┬──────────────────────────┘
                              │
                              ▼ Lifecycle Rule 4: At Day 2555 (7 Years)
                        [PERMANENT PURGE]
```

## 5. AWS Implementation
In AWS Amazon S3:
- **Lifecycle Rule Structure**:
  - Rules filter by Prefix, Object Tags, or Object Size.
  - Can manage **Current Versions** and **Noncurrent Versions** independently:
    - Example: Keep Current Version in Standard; transition Noncurrent Versions to Glacier Flexible after 7 days; permanently expire Noncurrent Versions after 30 days.
  - **The Abort Incomplete Multipart Uploads Rule**:
    - When a multipart upload fails or is abandoned, the uploaded binary chunks remain stored in S3 indefinitely, accumulating storage charges while hidden from standard `s3 ls` commands!
    - *Mandatory AWS Rule*: Always configure `AbortIncompleteMultipartUpload` after 7 days on every bucket.
- **S3 Replication Features**:
  - Supports Cross-Region Replication (CRR) and Same-Region Replication (SRR).
  - **S3 Replication Time Control (S3 RTC)**: Backed by an SLA to replicate **99.99% of objects within 15 minutes** of upload `[Doc: Amazon S3 RTC, checked 2026-09-03]`.
  - Supports cross-account replication with **Ownership Overwrite** (transfers object ownership to the destination account, preventing the source account from retaining access).
- **MFA Delete & Object Lock**:
  - *MFA Delete*: Requires physical Multi-Factor Authentication hardware tokens to permanently delete an object version or alter bucket versioning state.
  - *S3 Object Lock (WORM - Write Once, Read Many)*: Enforces SEC Rule 17a-4 compliance. Objects cannot be deleted or overwritten by anyone—including the AWS root account—for the retention period!

## 6. OCI Implementation
In Oracle Cloud Infrastructure Object Storage:
- **OCI Object Versioning**:
  - Enabled at the bucket level via the `versioning` attribute `[Doc: OCI Object Storage Versioning, checked 2026-09-03]`.
  - Generates immutable Version IDs.
  - Supports Delete Markers to preserve historical data.
- **OCI Lifecycle Policy Rules**:
  - OCI provides declarative JSON-based lifecycle rules.
  - Evaluates rules on a 24-hour cycle.
  - Supports transition from `Standard` to `InfrequentAccess` or `Archive`, and automatic object expiration (deletion).
  - Rules can filter by object name prefixes and pattern matching.
- **OCI Cross-Region Replication (CRR)**:
  - Configured directly at the bucket level: select a destination region and target bucket.
  - Fully asynchronous; mirrors object creations, overwrites, and metadata changes across OCI realms and regions.
  - Zero custom scripting required: OCI manages underlying replication pipelines with end-to-end encryption.
- **OCI Retention Rules (WORM Compliance)**:
  - Direct equivalent to AWS Object Lock.
  - Locks objects in a bucket so they cannot be deleted or modified until the retention duration expires.
  - Supports **Locked Retention Rules**: once locked, even the OCI Tenancy Administrator cannot shorten the duration or delete the rule!

## 7. Configuration
Comparing lifecycle governance and replication in Terraform across AWS and OCI:

### AWS S3 Lifecycle Policy & Replication (Terraform)
```hcl
# AWS S3 Bucket with Versioning Enabled
resource "aws_s3_bucket" "prod_data" {
  bucket = "enterprise-records-2026-prod"
}

resource "aws_s3_bucket_versioning" "versioning" {
  bucket = aws_s3_bucket.prod_data.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Automated Lifecycle Policy: Tier Demotion & Multipart Abort
resource "aws_s3_bucket_lifecycle_configuration" "lifecycle" {
  bucket = aws_s3_bucket.prod_data.id

  rule {
    id     = "tiering-and-cleanup-rule"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    # Transition current versions to Standard-IA at 30 days
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    # Transition current versions to Glacier at 90 days
    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    # Expire noncurrent (old) versions after 60 days
    noncurrent_version_expiration {
      noncurrent_days = 60
    }

    # CRITICAL: Clean up aborted multipart upload chunks!
    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}
```

### OCI Object Storage Lifecycle Policy (Terraform)
```hcl
# OCI Object Storage Bucket with Versioning
resource "oci_objectstorage_bucket" "prod_data" {
  compartment_id = var.compartment_id
  name           = "enterprise-records-prod"
  namespace      = var.tenancy_namespace
  versioning     = "Enabled"
  storage_tier   = "Standard"
}

# OCI Lifecycle Policy: Transition to Infrequent Access, then Archive
resource "oci_objectstorage_object_lifecycle_policy" "lifecycle" {
  namespace = var.tenancy_namespace
  bucket    = oci_objectstorage_bucket.prod_data.name

  rules {
    name        = "archive-old-logs"
    action      = "ARCHIVE"
    target      = "objects"
    time_amount = 90
    time_unit   = "DAYS"
    is_enabled  = true
    object_name_filter {
      inclusion_prefixes = ["logs/"]
    }
  }

  rules {
    name        = "expire-logs"
    action      = "DELETE"
    target      = "objects"
    time_amount = 365
    time_unit   = "DAYS"
    is_enabled  = true
  }
}
```

## 8. Data Flow
```text
The Delete Marker Lifecycle Flow:
1. User uploads document.pdf (VersionId: v1_initial).
2. User updates document.pdf (VersionId: v2_edited).
3. Current version is now v2_edited.
4. User executes: DELETE document.pdf (Without specifying VersionId!)
5. S3 / OCI Storage Engine Intercept:
   - Does NOT delete v1 or v2!
   - Injects a DELETE MARKER (VersionId: v3_del_marker) as the current version.
6. User executes GET document.pdf:
   - Storage engine reads current version (v3_del_marker).
   - Returns HTTP 404 Not Found.
7. Disaster Recovery / Accidental Restore:
   - Administrator executes: DELETE document.pdf?versionId=v3_del_marker
   - Delete marker is permanently removed.
   - document.pdf (v2_edited) immediately becomes current version again!
```

## 9. Security
- **Ransomware Mitigation via S3 Object Lock / OCI Retention Rules**:
  - If ransomware compromises administrative credentials, attackers attempt to delete all backups.
  - With **Object Lock in Compliance Mode** (WORM), the cloud storage engine cryptographically rejects all delete commands until the retention date, providing absolute ransomware immunity.
- **CRR KMS Key Management**:
  - Replicating encrypted objects requires granting the replication IAM role permissions to decrypt with the source KMS key and re-encrypt with the destination KMS key.

## 10. Reliability
- **Replication Loop Prevention**:
  - Cloud storage engines automatically attach metadata tracking replica origin.
  - If Bucket A replicates to Bucket B, Bucket B **will not replicate those objects back to Bucket A**, mathematically preventing infinite ping-pong replication storms.

## 11. Scaling
- **Massive Batch Processing via S3 Batch Operations**:
  - Applying lifecycle changes or encrypting billions of pre-existing objects can take weeks via standard API scripts.
  - **S3 Batch Operations** / OCI Bulk Object Operations executes asynchronous batch jobs across billions of objects in parallel with automated progress tracking and completion reports.

## 12. Observability
- **Storage Metrics in CloudWatch / OCI Monitoring**:
  - `BucketSizeBytes`: Total bytes stored across each storage tier.
  - `NumberOfObjects`: Total object count (track noncurrent versions count!).
  - `ReplicationLatency`: Tracks replication backlog between regions.

## 13. Cost
- **The Hidden Cost of Stranded Multipart Uploads**:
  - A 100 GB file upload fails halfway through. The 50 GB of uploaded chunks remain invisible on disk.
  - In a busy engineering org, **10 to 50 TB of abandoned chunks** can accumulate annually, costing **\$230 to \$1,150/month in pure waste**!
  - The `AbortIncompleteMultipartUpload` lifecycle rule completely eliminates this waste.
- **Small Object Glacier Warning**:
  - Glacier Flexible and Deep Archive add 32 KB of metadata overhead per object.
  - Transitioning millions of tiny 1 KB files to Glacier actually **increases** storage costs! Keep small files in Standard or combine them into archives (tar/zip) before uploading.

## 14. Failure Modes
- **The Accidental Massive Versioning Cost Explosion**: An application repeatedly writes a 100 MB state file to S3 every 30 seconds (`s3.putObject("state.json")`). Versioning is enabled on the bucket, but **no lifecycle rule exists to expire noncurrent versions**. Within 30 days, S3 accumulates $86,400\text{ versions} \times 100\text{ MB} = \mathbf{8.6\text{ TB}}$ of hidden noncurrent data, triggering an unexpected \$200/month bill for a single file!
- **The Non-Replicated Historical Data Blindspot**: Enabling Cross-Region Replication on an existing bucket containing 100 TB of data. **CRR only replicates NEW objects uploaded AFTER replication is enabled**! Existing 100 TB of historical objects are not copied until an explicit S3 Batch Replication job is executed.

## 15. Troubleshooting
When objects fail to replicate across regions:
1. **Inspect Replication Status**:
   ```bash
   aws s3api head-object --bucket <source-bucket> --key <key> --query "ReplicationStatus"
   ```
   - `FAILED`: Check IAM role permissions for KMS decrypt/encrypt.
   - `PENDING`: Object is queued in the asynchronous replication pipeline.
2. **Verify Versioning Status**:
   - Replication **requires** Object Versioning to be enabled on both the source and destination buckets!
3. **Verify Destination KMS Key Policy**:
   - The destination KMS key policy must explicitly allow the source replication IAM role to call `kms:GenerateDataKey` and `kms:Encrypt`.

## 16. Common Mistakes
- **Assuming Deleting an Object in a Versioned Bucket Frees Storage Space**: Executing `aws s3 rm s3://bucket/file.zip` and expecting your storage bill to drop. This merely adds a Delete Marker; the underlying 10 GB file continues consuming storage until its specific `VersionId` is purged.
- **Failing to Enable Versioning on Destination Replication Buckets**: Attempting to enable CRR without versioning enabled on the destination bucket.

## 17. Trade-offs
| Feature | Architectural Benefit | Financial / Operational Trade-off |
| :--- | :--- | :--- |
| **Object Versioning** | Protection against accidental deletes & corruption | Doubles or triples storage consumption without lifecycle rules |
| **Glacier Deep Archive**| 95% storage cost savings (\$0.00099/GB) | 12-hour retrieval latency; 180-day minimum storage commitment |
| **Cross-Region Replication**| Sub-15 minute disaster recovery geo-resilience | Doubles storage costs (2 regions) + cross-region data transfer fees |
| **S3 Object Lock (WORM)**| Absolute regulatory compliance & ransomware immunity | Immutable; objects cannot be deleted even by administrators |

## 18. Interview Questions
1. *A developer executes `aws s3 rm s3://my-bucket/database.sql` on a version-enabled S3 bucket, and panics thinking production backups were destroyed. Explain what actually occurred at the storage engine level and how to restore the file in 10 seconds.*
2. *Why does failing to configure an 'Abort Incomplete Multipart Uploads' lifecycle rule result in thousands of dollars in invisible cloud waste?*
3. *You enable Cross-Region Replication (CRR) on a production bucket containing 500 TB of data. Two days later, an audit reveals the disaster recovery bucket has only 10 GB. What is the root cause, and how do you backfill the replication?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "The production backups are 100% safe and have not been destroyed:
>
> 1. **The Delete Marker Mechanism**:
>    - Because **Object Versioning** was enabled on the bucket, executing a simple `DELETE` request without passing a specific `VersionId` does **not** purge the underlying data.
>    - Instead, the Amazon S3 / OCI storage engine creates a new, zero-byte record called a **Delete Marker** and assigns it a new Version ID (e.g., `del_marker_99`), placing it at the head of the object's version stack.
>    - When standard clients or the AWS Console attempt to read `database.sql`, S3 inspects the current head version, sees the Delete Marker, and returns `HTTP 404 Not Found`.
>
> 2. **The 10-Second Restoration Protocol**:
>    - To restore the file immediately, we simply list the object versions to find the Version ID of the Delete Marker:
>      ```bash
>      aws s3api list-object-versions --bucket my-bucket --prefix database.sql
>      ```
>    - We then issue an explicit `delete-object` command targeting the **Delete Marker's Version ID**:
>      ```bash
>      aws s3api delete-object --bucket my-bucket --key database.sql --version-id del_marker_99
>      ```
>    - Removing the Delete Marker instantly restores the previous active version (`VersionId: v1_production`) as the current head of the object. The database backup is immediately readable again with zero data loss."

## 20. Hands-on Exercise
**Objective**: Demonstrate Delete Marker creation and restoration using the AWS CLI.

### Verification Steps
1. Create a test bucket with versioning enabled:
   ```bash
   aws s3api put-bucket-versioning --bucket test-version-bucket --versioning-configuration Status=Enabled
   ```
2. Upload a file: `aws s3 cp file.txt s3://test-version-bucket/file.txt`.
3. Delete the file: `aws s3 rm s3://test-version-bucket/file.txt`.
4. Attempt to download: `aws s3 cp s3://test-version-bucket/file.txt .` $\longrightarrow$ Verify `404 Not Found`.
5. List versions: observe that `DeleteMarker` is present alongside the original `VersionId`.
6. Delete the `DeleteMarker` specifically: verify that `file.txt` immediately reappears in standard `s3 ls` listings.
