# System Design 03: Globally Distributed Low-Latency Media Upload Platform

---

## 1. Requirements & Constraints (R)

### 1.1 Business Context & Problem Statement
A global media streaming and creator platform requires a globally distributed, high-throughput ingestion platform capable of ingesting high-definition video, raw camera footage, and audio podcasts from content creators located worldwide (North America, Europe, Asia-Pacific, Latin America). Uploading multi-gigabyte media files across high-latency, lossy trans-oceanic public internet links results in slow transfer speeds, frequent socket disconnects, and poor creator experience.

The enterprise mandates a cloud architecture that leverages edge points of presence (PoPs), direct-to-object-storage multi-part uploads, resumable chunked transfers, automated anti-malware quarantine scanning, and an asynchronous transcoding pipeline.

### 1.2 Functional Requirements
1. **Direct-to-Storage Ingestion (Zero-Proxy Architecture)**: Creators must upload media chunks directly to cloud object storage via time-bounded, cryptographic pre-signed URLs or pre-authenticated requests. Application servers must never proxy raw video payloads.
2. **Chunked & Resumable Multipart Uploads**: Support file uploads from $50\text{ MB}$ up to $50\text{ GB}$. If a mobile or satellite connection drops mid-transfer, client can resume from the last successfully acknowledged chunk without restarting from byte zero.
3. **Automated Post-Upload Media Processing**:
   - Extract media technical metadata (codec, bitrate, resolution, audio channels).
   - Execute asynchronous malware and virus scanning in an isolated quarantine sandbox.
   - Trigger adaptive bitrate (ABR) transcoding into HLS/DASH streams (1080p, 720p, 480p, 360p) upon malware clearance.
4. **Creator Upload Status Notification**: Provide real-time upload and transcoding progress updates via WebSockets or Webhook callbacks.

### 1.3 Non-Functional Requirements & Quantitative SLAs
- **Global Concurrency & Data Volume**:
  - Baseline Active Uploads: **5,000 concurrent streams**.
  - Peak Active Uploads: **25,000 concurrent streams**.
  - Daily Ingestion Volume: $\approx \mathbf{500\text{ TB/day}}$ ($15\text{ PB/month}$).
  - Object Size Distribution: Mean $1.5\text{ GB}$, Max $50\text{ GB}$.
- **Latency & Speed SLAs**:
  - Upload Initialization API: $P95 < 25\text{ms}$, $P99 < 50\text{ms}$.
  - Network Ingress Throughput: Creators must achieve $\ge 80\%$ of their local uplink ISP bandwidth by terminating TCP at the nearest cloud edge PoP.
- **Durability & Availability SLAs**:
  - Storage Durability: $\mathbf{99.999999999\%}$ (11 Nines) annual object durability.
  - Ingestion Availability: **99.99%** annual availability.
  - Recovery Point Objective (RPO): $\mathbf{0}$ (Once a chunk is acknowledged, it is persisted across at least 3 physical storage facilities).

---

## 2. High-Level Architecture (A)

### 2.1 Dual-Cloud Architectural Topology

```
========================================================================================================================
                     GLOBALLY DISTRIBUTED MEDIA INGESTION & PROCESSING TOPOLOGY
========================================================================================================================

                                       [ Creator Clients (Global Edge) ]
                                                       │
                                   ┌───────────────────┴───────────────────┐
                                   │ (1. Initialize Upload: Metadata)      │ (2. Upload Chunks via Presigned URL)
                                   ▼                                       ▼
             [ Edge DNS / Anycast: Route 53 / OCI DNS ]    [ Edge POP: S3 Transfer Acceleration / OCI CDN Edge ]
                                   │                                       │ (Optimized Cloud Private Backbone)
                                   ▼                                       ▼
             [ Regional Ingress API: ALB / OCI LB ]        [ QUARANTINE OBJECT STORAGE BUCKET ]
                                   │                         - AWS: S3 Bucket (Prefix Hash Sharded)
                                   ▼                         - OCI: Object Storage (Tiered Compartment)
             [ Control Plane Pods: EKS / OKE ]                             │
              - Validate Content-Type / Size                               │ (ObjectCreated Notification)
              - Issue Presigned Chunk URLs                                 ▼
              - Track State in DynamoDB / OCI NoSQL        [ EVENT BUS: EventBridge / OCI Events ]
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
                                                          - Global Creator Streaming Playback
========================================================================================================================
```

### 2.2 Dual-Cloud Component Mapping

| Architectural Function | AWS Native Primitive | OCI Native Primitive | Architecture Rationale |
| :--- | :--- | :--- | :--- |
| **Global Ingress Steering** | Route 53 Geolocation / Latency Routing `[Doc: Route 53, checked 2026]` | OCI DNS Traffic Management Steering Policies `[Doc: OCI DNS, checked 2026]` | Routes client DNS requests to geographically closest edge ingestion location within $< 15\text{ms}$. |
| **Edge Ingress Acceleration**| Amazon S3 Transfer Acceleration (CloudFront Edge PoPs) `[Doc: S3, checked 2026]` | OCI FastConnect / OCI CDN Edge with Pre-Authenticated Requests `[Doc: OCI Storage, checked 2026]` | Terminates client TLS handshake at the local edge; routes payload over private cloud backbone to origin bucket. |
| **Direct-to-Storage Token** | S3 Pre-Signed URLs (Chunked `UploadPart` API) | OCI Pre-Authenticated Requests (PAR) with Object Put Permission | Grants scoped, time-limited write access to a specific object chunk without exposing IAM credentials. |
| **Quarantine Storage** | Amazon S3 Standard (Quarantine Bucket with WORM retention) | OCI Object Storage Standard (Isolated Security Compartment) | Isolates untrusted incoming creator bytes from production streaming buckets until security scanning clears. |
| **Event Routing Fabric** | Amazon EventBridge + S3 Event Notifications | OCI Events Service + OCI Streaming | Dispatches decoupled, sub-second notifications when multi-part upload assembly completes. |
| **Malware Sandbox** | AWS GuardDuty Malware Protection for S3 + ClamAV ECS Task | OCI Vulnerability Scanning Service + Container Instance AV Worker | Inspects file signatures and executable headers in a sterile sandbox before triggering transcoding. |
| **Transcoding Engine** | AWS Batch / EKS with GPU Instances (`g5.2xlarge`) | OKE with NVIDIA GPU Shapes (`VM.GPU.A10.1` / `BM.GPU.A100`) | Hardware-accelerated NVENC FFmpeg pipelines execute parallel transcoding into 4K/1080p/720p HLS profiles. |
| **Upload State Ledger** | Amazon DynamoDB (On-Demand Capacity) | OCI NoSQL Database | Stores chunk index, ETag checksums, upload part status, and resumable session state. |

---

## 3. Traffic Flow & Ingress Path (T)

### 3.1 Step-by-Step Multipart Ingress Lifecycle

```text
====================================================================================================
                             MULTIPART UPLOAD LIFECYCLE SEQUENCE
====================================================================================================

Client                  Control API (EKS/OKE)          DynamoDB / NoSQL           Storage (S3/OCI)
  │                            │                             │                          │
  ├─ 1. POST /uploads/init ───►│                             │                          │
  │  (file_name, size, SHA256) │                             │                          │
  │                            ├─ 2. Create Upload Session ─►│                          │
  │                            │                             │                          │
  │                            ├─ 3. Initiate Multipart ───────────────────────────────►│
  │                            │◄── Return UploadId ────────────────────────────────────┤
  │                            │                                                        │
  │                            ├─ 4. Generate Presigned URLs (Parts 1..N)               │
  │◄─ 5. Return UploadId ──────┤                                                        │
  │   + Presigned URLs List    │                                                        │
  │                            │                                                        │
  ├─ 6. PUT Chunk 1 (16MB) ────────────────────────────────────────────────────────────►│ (Direct Edge)
  │◄── ACK Chunk 1 (ETag_1) ────────────────────────────────────────────────────────────┤
  │                                                                                     │
  ├─ 7. PUT Chunk 2..N ────────────────────────────────────────────────────────────────►│ (Parallel Chunks)
  │◄── ACK Chunk 2..N (ETag_N) ─────────────────────────────────────────────────────────┤
  │                            │                                                        │
  ├─ 8. POST /uploads/complete ┤                                                        │
  │  (UploadId, Parts List)    ├─ 9. Complete Multipart ───────────────────────────────►│
  │                            │◄── Assemble Object & Verify Checksum ──────────────────┤
  │                            │                                                        │
  │◄─ 10. HTTP 200 OK ─────────┤                                                        │
  │   (Upload Complete)        │                                                        │
====================================================================================================
```

1. **Upload Initialization Phase**:
   - Creator client issues `POST /api/v1/uploads/initialize` containing `{ filename: "raw_footage.mov", size_bytes: 10737418240, content_type: "video/quicktime", checksum_sha256: "e3b0c442..." }`.
   - Control Plane API validates that the user possesses a valid subscription, and checks that file extension and MIME type match allowed whitelists.
   - API calls S3 `CreateMultipartUpload` / OCI Object Storage `CreateMultipartUpload`, obtaining a cluster-unique `UploadId`.
   - The file is divided into deterministic $16\text{ MB}$ chunks (e.g., a $10\text{ GB}$ file yields $640$ chunks).
   - The API generates a cryptographic **Pre-Signed PUT URL** (AWS) or **Pre-Authenticated Request (PAR)** (OCI) for each part, valid for 60 minutes.
   - The session record is persisted in DynamoDB / OCI NoSQL with status `IN_PROGRESS`.

2. **Parallel Direct-to-Storage Transfer**:
   - The client application executes up to 8 concurrent HTTP `PUT` requests directly to the edge storage endpoint:
     - **AWS**: `https://{bucket}.s3-accelerate.amazonaws.com/{prefix}/{object}?partNumber=1&uploadId={id}`
     - **OCI**: `https://objectstorage.{region}.oraclecloud.com/p/{par_token}/n/{namespace}/b/{bucket}/u/{uploadId}/1`
   - TCP connections terminate at the nearest AWS CloudFront / OCI Edge POP. Chunks are routed over the dedicated low-jitter cloud provider backbone directly to the target storage facility.
   - For every chunk persisted, the storage engine validates the `Content-MD5` header and returns an `ETag`.

3. **Resumable State Reconciliation**:
   - If chunk 42 fails due to a local WiFi drop, the client catches the socket timeout, queries `GET /api/v1/uploads/{id}/status`, retrieves the list of acknowledged parts, and re-transmits only chunk 42.

4. **Object Assembly & Finalization**:
   - Once all parts are uploaded, the client calls `POST /api/v1/uploads/{id}/complete` with the manifest of part numbers and ETags.
   - Storage service stitches parts into a contiguous object and recalculates the holistic SHA-256 checksum.

---

## 4. Data Flow & Storage Engine (D)

### 4.1 Storage Prefix Sharding & Namespace Partitioning
Under extreme scale (25,000 concurrent uploads $\times$ multi-part chunks), high request rates against an object storage bucket can encounter throttling if all keys share a common lexical prefix.
- **AWS S3 Partitioning Rule**: S3 scales automatically to handle **3,500 PUT/POST/DELETE requests/second** per partitioned prefix `[Doc: S3 Performance, checked 2026]`.
- **OCI Object Storage Rule**: OCI scales partition buckets horizontally based on namespace and bucket sharding algorithms.
- **Architectural Implementation**: Prepend a deterministic MD5 hash prefix to all object keys:
  ```text
  s3://quarantine-media-bucket/{md5_hash(creator_id)[0:4]}/{creator_id}/{upload_id}/master.mov
  Example:
  s3://quarantine-media-bucket/a8f1/creator-9942/upl-77182/master.mov
  s3://quarantine-media-bucket/3c9e/creator-1102/upl-88219/master.mov
  ```
  This guarantees that uploads are uniformly distributed across hundreds of underlying physical storage partitions.

### 4.2 Lifecycle Management & Automated Tiering Policy

```text
====================================================================================================
                        MEDIA STORAGE LIFECYCLE TIERING TIMELINE
====================================================================================================

[ Day 0: Ingestion ] ──────► S3 Standard / OCI Object Storage Standard (Hot Access Tier)
                                │
                                ▼ (Day 14: Processing & Initial Creator Edits Complete)
[ Day 14: Warm Tier ] ─────► S3 Standard-Infrequent Access (S3 Standard-IA) / OCI Infrequent Access
                                │ - 50% storage cost reduction
                                │ - Immediate retrieval latency
                                ▼ (Day 90: Inactive Raw Footage)
[ Day 90: Cold Archive ] ──► S3 Glacier Flexible Retrieval / OCI Archive Storage
                                │ - 85% storage cost reduction
                                │ - Retrieval time: 3-5 hours
                                ▼ (Day 365: Regulatory Long-Term Compliance)
[ Day 365+: Deep Cold ] ───► S3 Glacier Deep Archive / OCI Archive Retention Rules
                                - Cost: $0.00099 / GB / month ($1.00 / TB / month)
====================================================================================================
```

---

## 5. Security Architecture (S)

### 5.1 Pre-Signed Token Least-Privilege Hardening
Pre-signed URLs and PARs represent temporary delegations of storage authority. If intercepted, an attacker could potentially overwrite critical media or upload arbitrary content.
- **Strict Scope Controls**:
  - The pre-signed URL is restricted to the exact object key: `quarantine/{creator_id}/{upload_id}/part_042`.
  - HTTP method is locked to `PUT` only.
  - The URL enforces mandatory headers: `x-amz-content-sha256` or `Content-MD5`.
  - Expiration window is capped to **15 minutes** per chunk.
  - Content-Length range is bounded in the S3 bucket policy:
    ```json
    {
      "Sid": "EnforceChunkSizeRange",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::quarantine-media-bucket/*",
      "Condition": {
        "NumericGreaterThan": {"s3:content-length-range": 67108864}
      }
    }
    ```

### 5.2 Quarantine Sandbox & Anti-Malware Pipeline

```text
[ Raw Upload Complete in Quarantine Bucket ]
                   │
                   ▼
     [ S3 ObjectCreated / OCI Event ]
                   │
                   ▼
     [ Malware Scanning Worker (ClamAV + GuardDuty) ]
                   │
         ┌─────────┴─────────┐
         │                   │
         ▼ (Clean)           ▼ (Infected / Malicious)
  [ Move to Staging ]   [ 1. Quarantine Hard Lock ]
         │              [ 2. Alert Security Operations ]
         ▼              [ 3. Hard Delete Object ]
  [ Trigger Transcode ] [ 4. Suspend Creator Account ]
```

---

## 6. Reliability & High Availability (R)

### 6.1 Multi-Region Disaster Recovery & Durability Guarantees
- **Cross-Region Replication (CRR)**:
  - High-value master video assets must survive catastrophic regional cloud disasters.
  - **AWS**: S3 Cross-Region Replication with **Replication Time Control (S3 RTC)** guarantees that $99.99\%$ of newly uploaded media is replicated to a secondary region (e.g., `us-east-1` $\to$ `us-west-2`) within **15 minutes** `[Doc: S3 RTC, checked 2026]`.
  - **OCI**: OCI Object Storage Cross-Region Replication automatically synchronizes objects across geographically remote regions (e.g., Ashburn $\to$ Phoenix) asynchronously.

### 6.2 Resilient Client Upload State Machine
Mobile creators frequently pass through tunnels or experience cellular dead zones. The upload client architecture enforces:
1. **Exponential Backoff with Full Jitter**:
   $$T_{\text{sleep}} = \text{random}(0, \min(M, T_{\text{base}} \times 2^{\text{attempt}}))$$
2. **Deterministic Chunk Resumption**: Client maintains a local SQLite state database on the mobile device recording `{upload_id, chunk_index, etag, status}`. On reconnect, it requests state synchronization from the API before resuming data flow.

---

## 7. Scaling & Capacity Planning (S)

### 7.1 GPU Worker Autoscaling via Event Backlog
Transcoding raw 4K video is highly compute-intensive. Worker pools cannot scale on CPU metrics alone because a single FFmpeg task will peg CPU/GPU to 100% immediately.
- **Scaling Metric**: SQS Queue Depth / OCI Streaming Lag ($L$).
- **Autoscaling Logic**:
  $$N_{\text{workers}} = \left\lceil \frac{\text{ApproximateNumberOfMessagesVisible}}{\text{TargetTasksPerWorker}} \right\rceil$$
- **AWS Implementation**: KEDA ScaledObject provisions Amazon EC2 G5 GPU instances (`g5.2xlarge` with NVIDIA A10G Tensor Core GPUs) via Karpenter.
- **OCI Implementation**: OKE Cluster Autoscaler provisions `VM.GPU.A10.1` shapes dynamically from pre-allocated spot/preemptible shape capacity pools.

---

## 8. Observability & Production Telemetry (O)

### 8.1 Critical Upload Telemetry Metrics
1. **Ingress Transfer Goodput (Mbps)**:
   - Measured per creator geographic region.
   - Baseline: P50 $> 150\text{ Mbps}$, P90 $> 400\text{ Mbps}$.
2. **Chunk Retry & Error Ratio**:
   - $\text{Chunk Error Rate} = \frac{\text{Failed Chunk PUTs}}{\text{Total Chunk PUTs}}$.
   - Alert threshold: $> 0.5\%$ over 5 minutes indicates edge PoP degradation or ISP routing anomaly.
3. **Quarantine Scan Duration**:
   - Duration from `ObjectCreated` to security clearance. P95 Target: $< 8\text{ seconds}$.
4. **Transcoding Queue Backlog Age**:
   - Oldest un-transcoded video in queue. Alert if $> 15\text{ minutes}$.

---

## 9. Cloud Cost & Unit Economics (C)

### 9.1 Monthly FinOps BOM (500 TB/Day Ingestion)

```text
====================================================================================================
               MONTHLY FINOPS ESTIMATE: 500 TB/DAY INGESTION & PROCESSING
====================================================================================================

COMPONENT                         AWS NATIVE IMPLEMENTATION       OCI NATIVE IMPLEMENTATION
----------------------------------------------------------------------------------------------------
Data Ingress (Internet -> Cloud)  $0.00 (Free Ingress)            $0.00 (Free Ingress)
S3 Transfer Acceleration          $6,000 ($0.04/GB edge ingress)  $0.00 (Standard edge included)
Hot Object Storage (15 PB)        $345,000 (S3 Standard baseline) $382,500 (OCI Standard baseline)
Automated Storage Lifecycle Tier  -$160,000 (Tiering savings)     -$185,000 (OCI Infrequent savings)
GPU Transcode Fleet (Spot/Preempt)$48,000 (EC2 G5 Spot instances) $26,000 (OCI A10 Preemptible)
Control Plane (EKS/OKE + DB)      $4,200 (EKS + DynamoDB)         $2,800 (OKE + OCI NoSQL)
----------------------------------------------------------------------------------------------------
TOTAL ESTIMATED MONTHLY SPEND     $243,200                        $226,300
STORAGE COST PER RAW GB           ~$0.016 / GB                    ~$0.015 / GB
====================================================================================================
```

### 9.2 FinOps Cost Optimization Levers
1. **Direct-to-Storage Architecture**: Eliminating intermediate proxy EC2/VM servers saves over **\$45,000/month** in compute and NAT Gateway data processing fees.
2. **Preemptible / Spot GPU Instances**: Video transcoding is an asynchronous, stateless workload. Utilizing AWS Spot instances and OCI Preemptible GPU instances yields a **60% to 70% discount** compared to on-demand compute rates.

---

## 10. Failure Modes & Cascades (F)

### 10.1 Scenario A: S3 / Object Storage 503 SlowDown Under Flash Crowd
- **Failure Condition**: 10,000 creators upload video chunks simultaneously targeting un-hashed keys in a single bucket prefix (`/uploads/...`), causing S3 to throw `503 SlowDown` throttling responses.
- **Mitigation Flow**:
  1. Client upload SDK detects HTTP 503 and immediately applies exponential backoff with full jitter.
  2. The ingestion control plane enforces **hexadecimal prefix sharding** (`/a4b2/uploads/...`), immediately dispersing requests across hundreds of internal storage partitions.
  3. S3/OCI partition heat monitors detect the elevated traffic and split storage partitions automatically.
  4. 503 errors dissipate to 0% within 60 seconds.

### 10.2 Scenario B: Corrupt Chunk Stitches into Defective Master File
- **Failure Condition**: A creator's network hardware suffers packet corruption, flipping bits in chunk 15.
- **Mitigation Flow**:
  1. The client pre-computes an MD5 checksum for chunk 15 before transmission, including it in the `Content-MD5` header.
  2. Cloud Object Storage receives the chunk, computes the MD5 on arrival, and compares it to the header.
  3. Checksums do not match: Storage engine immediately discards the chunk and returns `HTTP 400 BadDigest`.
  4. The corrupted chunk is never accepted; the client automatically re-reads the chunk from disk and re-transmits.

---

## 11. Trade-offs & Defense (T)

### 11.1 Key Architectural Compromises

```text
====================================================================================================
                                  ARCHITECTURAL TRADE-OFF MATRIX
====================================================================================================

DESIGN CHOICE                   CHOSEN OPTION              REJECTED ALTERNATIVE       TECHNICAL JUSTIFICATION
----------------------------------------------------------------------------------------------------
Ingress Data Path               Direct-to-Storage via      API Gateway / Microservice Proxying 500 TB/day through
                                Pre-Signed URLs            Proxy Server               compute generates massive
                                                                                      server memory pressure and
                                                                                      doubles network egress costs.
----------------------------------------------------------------------------------------------------
Transcoding Infrastructure      Self-Managed GPU Workers   Managed Transcode Service  Managed services charge up
                                (FFmpeg on EKS / OKE)      (AWS MediaConvert)         to $0.015/minute of video;
                                                                                      self-managed GPU spot nodes
                                                                                      reduce cost by 78% at scale.
----------------------------------------------------------------------------------------------------
Upload Coordination             Client-Side Chunking       Server-Side Chunking       Client-side chunking allows
                                (16 MB Multi-part)         (Streaming Chunking)       fine-grained resumability
                                                                                      after network disconnects.
====================================================================================================
```

### 11.2 Bar-Raiser Defense Script

> **Interviewer**: *"Why not just route all upload traffic through an API Gateway or Reverse Proxy cluster so you can perform authentication, validation, and virus scanning in real-time as the bytes fly through?"*

**Candidate Defense**:
*"Streaming multi-gigabyte files through an API Gateway or compute proxy layer is a catastrophic anti-pattern at our scale of 500 TB/day. First, managed API Gateways enforce rigid payload limits—AWS API Gateway hard-caps payloads at 10 MB, making multi-gigabyte uploads impossible without breaking them into thousands of tiny API calls.*

*Second, proxying raw media through worker instances forces compute nodes to hold open idle TCP sockets, saturating memory and ephemeral disk buffers. It also doubles data transfer costs, as traffic pays to enter the compute tier and then pays again to write to object storage.*

*Our zero-proxy architecture uses the control plane strictly for metadata and security authorization, issuing time-bounded cryptographic pre-signed URLs. The heavy data payload flows directly from client edge to object storage. We achieve security validation asynchronously using event-driven quarantine buckets and sandbox scanners before releasing assets to production, achieving optimal throughput, fault isolation, and minimal operational cost."*
