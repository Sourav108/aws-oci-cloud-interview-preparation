# Module 12: Object Storage (Amazon S3 vs. OCI Object Storage)

> **Architectural Objective**: *Master hyperscale object storage architectures, strong consistency models, storage tiering economics, lifecycle transitions, and high-throughput data transfer. Deconstruct the mechanics of Amazon S3 against OCI Object Storage, evaluate time-bound delegated access (Pre-Signed URLs vs. Pre-Authenticated Requests), and master multipart uploads and prefix partitioning performance.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. S3 vs. OCI Object Storage Architecture](01-s3-vs-oci-object-storage-architecture.md)** | Flat Key-Value Namespace, Read-after-Write Strong Consistency, Storage Tiers (Standard, Intelligent-Tiering, Glacier / Archive), Pre-Signed URLs vs. Pre-Authenticated Requests (PAR) | Full 20-Section Deep Dive (~2,400 words) |
| **[02. Lifecycle Policies, Versioning & Replication](02-lifecycle-policies-versioning-and-replication.md)** | Object Versioning, Delete Markers, Aborting Incomplete Multipart Uploads, Tier Demotion Rules, Cross-Region Replication (CRR) | Full 20-Section Deep Dive (~2,200 words) |
| **[03. Multipart Uploads & Performance Optimization](03-multipart-uploads-and-performance-optimization.md)** | Multipart Upload Protocols (5 MB to 5 TB), S3 Prefix Partitioning (3,500 PUT / 5,500 GET per prefix) vs. OCI Flat Scaling, Checksums | Abbreviated Storage Guide (~900 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **Strong Consistency Mechanics**: How Amazon S3 and OCI Object Storage enforce atomic read-after-write consistency for PUTs and DELETEs, and the architectural implications for distributed data pipelines.
2. **Direct Client Uploads via Delegated Authentication**: How to design secure, high-throughput architectures using AWS Pre-Signed URLs and OCI Pre-Authenticated Requests (PAR) to offload gigabytes of file ingestion away from application servers directly to object storage.
3. **Prefix Partitioning & S3 Performance**: How to engineer S3 object key naming conventions to eliminate single-partition I/O bottlenecks and achieve 100,000+ requests per second across horizontally scaled prefixes.
