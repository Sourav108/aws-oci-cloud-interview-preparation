# 03. Multipart Uploads & Object Storage Performance Optimization

## 1. Problem
When transferring multi-gigabyte or terabyte-scale objects (e.g., raw video footage, database backups, disk images) over the internet or across cloud networks, single-stream HTTP PUT uploads are fragile and slow. A single transient TCP connection drop at 98% completion forces the client to restart the entire multi-gigabyte upload from byte zero. Furthermore, applications attempting to execute tens of thousands of requests per second against an S3 bucket frequently hit severe `HTTP 503 Slow Down` rate-limiting bottlenecks if object keys are structured naively.

## 2. Cloud Concept: Multipart Uploads & Prefix Partitioning
```text
MULTIPART PARALLEL UPLOAD:
[Client Process]
       ├──► Thread 1: Upload Part 1 (100 MB) ──► S3 / OCI Storage Node A ─┐
       ├──► Thread 2: Upload Part 2 (100 MB) ──► S3 / OCI Storage Node B ─┼──► CompleteMultipartUpload
       └──► Thread 3: Upload Part 3 (100 MB) ──► S3 / OCI Storage Node C ─┘    (Reassembles payload)

S3 PREFIX PARTITIONING THROUGHPUT:
Bucket: my-high-throughput-bucket
Prefix 1: /2026/09/03/orders/ ──► Partition 1 (3,500 PUT / 5,500 GET per sec)
Prefix 2: /2026/09/03/users/  ──► Partition 2 (3,500 PUT / 5,500 GET per sec)
Aggregate Bucket Capacity: Automatically scales linearly across distinct prefixes!
```

### The Multipart Upload Protocol
- Breaks a large object (from 5 MB up to 5 TB) into distinct binary chunks (Parts 1 to 10,000, each between 5 MB and 5 GB).
- Three-phase protocol:
  1. `InitiateMultipartUpload`: Obtains an `UploadId`.
  2. `UploadPart`: Uploads parts independently and in **parallel** across multiple network sockets. Each part returns an `ETag` checksum. If Part 42 fails due to a network blip, **only Part 42 is retried**!
  3. `CompleteMultipartUpload`: Client provides the manifest of Part Numbers and ETags. The cloud storage engine reassembles the parts into a single atomic object.

### The S3 Prefix Partitioning Engine
- Amazon S3 automatically partitions bucket storage across underlying physical hardware nodes based on **Object Key Prefixes** (the string between the bucket name and the last delimiter).
- Each distinct prefix delivers:
  - **3,500 PUT / POST / DELETE requests per second** `[Doc: Amazon S3 Performance Guidelines, checked 2026-09-03]`.
  - **5,500 GET / HEAD requests per second**.
- If your application spreads keys across 20 distinct prefixes, the bucket automatically scales to support $20 \times 5,500 = \mathbf{110,000\text{ GET requests per second}}$!

## 3. AWS Implementation
In Amazon S3:
- AWS SDKs (TransferManager) automatically split files larger than 100 MB into multipart uploads using 8 MB chunks.
- **S3 Express One Zone**:
  - Purpose-built high-performance storage class designed for AI/ML training and financial modeling.
  - Delivers **consistent single-digit millisecond latency** ($< 10\text{ms}$) and up to 10x faster request processing than S3 Standard `[Doc: Amazon S3 Express One Zone, checked 2026-09-03]`.
- **S3 Byte-Range Fetches**: Allows downloading specific byte slices (e.g., `bytes=0-1048575` to fetch the first 1 MB) in parallel across multiple threads to maximize network throughput.

## 4. OCI Implementation
In OCI Object Storage:
- **OCI Multipart Uploads**:
  - Supports part sizes from 100 KiB up to 50 GiB per part, up to 10,000 parts per object (maximum object size: 10 TiB).
  - Supported natively by OCI CLI and OCI Java/Python SDKs.
- **Flat Scalability Architecture**:
  - OCI Object Storage utilizes a distributed metadata cluster that minimizes hot-partitioning constraints.
  - Native integration with OCI High Performance Computing (HPC) networks for multi-gigabit throughput.

## 5. Production Failure Modes: Sequential Prefix Hotspotting
- **The Timestamp Prefix Hotspot**: An application writes 20,000 log records per second using the date as a key prefix: `s3://bucket/2026-09-03-12-00-01-log.json`. Because every single request starts with the exact same prefix string, all traffic lands on a **single underlying S3 storage partition**. At 3,501 requests/second, S3 rejects requests with `HTTP 503 Slow Down`.
  - *Fix*: Prepend a high-entropy hash or reversed user ID to the key:
    `s3://bucket/{hash_prefix}/2026-09-03/log.json`.

## 6. Troubleshooting & Inspection Commands
1. **List Active Stranded Multipart Uploads**:
   ```bash
   aws s3api list-multipart-uploads --bucket <bucket-name>
   ```
2. **Abort an Abandoned Multipart Upload**:
   ```bash
   aws s3api abort-multipart-upload --bucket <bucket> --key <key> --upload-id <upload-id>
   ```

## 7. Senior Interview Question & Defense
**Question**: *Your application architecture needs to ingest 50,000 PUT requests per second of telemetry JSON files into a single Amazon S3 bucket. A junior engineer proposes naming the files `s3://telemetry-bucket/YYYY/MM/DD/HH/device_id.json`. Will this architecture survive? If not, how do you redesign the key naming convention?*

**Staff-Level Defense**:
> "The proposed architecture will **instantly fail** and collapse under `HTTP 503 Slow Down` throttling:
>
> 1. **The Partition Bottleneck**:
>    - Amazon S3 scales throughput by automatically partitioning underlying storage nodes based on **Object Key Prefixes**.
>    - A single prefix supports a maximum of **3,500 PUT requests per second**.
>    - Because all 50,000 devices will write to the exact same prefix at any given hour (e.g., `/2026/09/03/12/`), all 50,000 requests hit the **same physical S3 partition**. At 3,501 RPS, S3 throttles the remaining 46,500 requests!
>
> 2. **The High-Entropy Redesign**:
>    - To support 50,000 PUT requests per second, we must distribute traffic across at least $\frac{50,000}{3,500} \approx \mathbf{15\text{ independent S3 partitions}}$.
>    - We redesign the object key naming convention by placing a **high-entropy hash prefix** at the beginning of the key:
>      ```text
>      s3://telemetry-bucket/{MD5_Hash(device_id)[0:4]}/device_id/YYYY/MM/DD/HH/payload.json
>      ```
>    - Example:
>      - Device 1234: `s3://telemetry-bucket/a8f2/1234/2026/09/03/12/payload.json`
>      - Device 9876: `s3://telemetry-bucket/b1c4/9876/2026/09/03/12/payload.json`
>    - The 4-character hex hash prefix generates up to $16^4 = \mathbf{65,536\text{ distinct prefixes}}$. S3 partitions the keyspace horizontally across thousands of physical storage servers, effortlessly supporting 50,000+ PUT/s with zero throttling."
