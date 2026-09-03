# 02. EFS vs. OCI File Storage Service (FSS)

## 1. Problem
Containerized applications running on Kubernetes (EKS/OKE) or serverless container tasks (Fargate / Container Instances) frequently require shared persistent filesystems where hundreds of pods can concurrently read and write shared assets (e.g., WordPress media uploads, Git repository mirrors, shared machine learning datasets). Standard cloud block devices (EBS / OCI Block Volume) fail in this topology because they are single-attach devices locked to a single virtual machine in a specific Availability Zone. Building a self-managed NFS server on an EC2 instance creates a catastrophic single-point-of-failure with severe throughput bottlenecks. Managed cloud NFS services solve this by providing serverless, multi-AZ/multi-AD POSIX-compliant file systems.

## 2. Cloud Concept
### Managed Distributed Network File Systems (NFS v4.1)
- **POSIX-Compliant Shared Storage**:
  - Exposes standard Network File System (NFS v4.0 / v4.1) protocols.
  - Supports true **ReadWriteMany (RWX)** access: hundreds or thousands of independent compute instances, container pods, and serverless functions can mount the filesystem concurrently across multiple Availability Zones / Availability Domains.
  - Enforces standard POSIX permissions (users, groups, read/write/execute bits) and atomic file locking (`flock`, `fcntl`).
- **Elastic Auto-Growth**:
  - There is no need to pre-provision disk size.
  - A filesystem starts at 0 bytes and grows automatically as you write files, scaling elastically to petabytes of data. You pay strictly for the exact gigabytes stored.
- **Mount Targets & VCN/VPC Integration**:
  - Cloud file systems expose **Mount Targets**: virtual network interfaces placed inside your private subnets with private IP addresses.
  - Compute instances mount the filesystem using standard operating system NFS clients:
    ```bash
    mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576 <mount-target-ip>:/ /mnt/shared
    ```

## 3. Mental Model
Think of cloud storage paradigms as office document collaboration:
- **Block Storage (EBS / Block Volume)** is a physical paper notebook locked inside your desk drawer. Only one person sitting at that exact desk can write in the notebook.
- **Object Storage (S3 / OCI Object Storage)** is sending emails with document attachments. It is great for sharing completed documents, but you cannot edit paragraph 3 of an attachment in real-time; you must download it, edit it locally, and re-upload the entire file.
- **Managed File Storage (EFS / FSS)** is a shared Google Doc: 50 team members can open the document simultaneously, read different pages in parallel, and type edits into paragraph 4 with real-time atomic locking.

## 4. Architecture Diagram
```text
MANAGED MULTI-AZ DISTRIBUTED FILE SYSTEM ARCHITECTURE:

┌────────────────────────────────────────────────────────────────────────┐
│ MANAGED DISTRIBUTED NFS BACKBONE (Amazon EFS / OCI FSS)                │
│ * Petabyte-scale shared storage engine                                 │
│ * Replicated across 3 Availability Zones / Availability Domains        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         ▼                          ▼                          ▼
┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
│ MOUNT TARGET AZ1 │       │ MOUNT TARGET AZ2 │       │ MOUNT TARGET AZ3 │
│ (Private Subnet) │       │ (Private Subnet) │       │ (Private Subnet) │
│ IP: 10.0.1.20    │       │ IP: 10.0.2.20    │       │ IP: 10.0.3.20    │
└────────┬─────────┘       └────────┬─────────┘       └────────┬─────────┘
         │                          │                          │
         ▼ NFS v4.1 Mount           ▼ NFS v4.1 Mount           ▼ NFS v4.1 Mount
┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
│ K8s Worker Node  │       │ K8s Worker Node  │       │ Serverless Pod   │
│ [Pod 1]  [Pod 2] │       │ [Pod 3]  [Pod 4] │       │ [Fargate Task]   │
│ (Mount: /shared) │       │ (Mount: /shared) │       │ (Mount: /shared) │
└──────────────────┘       └──────────────────┘       └──────────────────┘
```

## 5. AWS Implementation
In AWS Amazon EFS:
- **Performance Modes**:
  - *General Purpose (Default)*: Optimized for latency-sensitive web serving, CMS, and general business applications. Lowest per-operation latency.
  - *Max I/O*: Sized for massive Big Data, machine learning, and media processing clusters where thousands of instances query files concurrently (trades slightly higher base latency for unlimited aggregate throughput).
- **Throughput Modes**:
  - *Elastic Throughput (Recommended)*: Automatically scales throughput dynamically based on workload demand (up to 3 GiB/s read, 1 GiB/s write per filesystem) `[Doc: Amazon EFS Performance, checked 2026-09-03]`. You pay strictly for the data read and written.
  - *Provisioned Throughput*: Pre-allocates dedicated throughput (e.g., 500 MB/s) regardless of stored volume size.
  - *Bursting Throughput*: Baseline throughput scales proportionally with stored data volume ($50\text{ KiB/s per GiB stored}$).
- **EFS Access Points**:
  - Enforces application directory sandboxing. Translates incoming NFS requests to a specific POSIX UID/GID (e.g., `uid: 1000`), completely eliminating root permission vulnerabilities.

## 6. OCI Implementation
In Oracle Cloud Infrastructure File Storage Service (FSS):
- **Enterprise-Grade High Performance**:
  - OCI FSS delivers high-throughput, low-latency file storage designed for enterprise Oracle Database dumps, HPC rendering, and OKE container persistent volumes `[Doc: OCI File Storage Service Overview, checked 2026-09-03]`.
  - Built directly on OCI's high-speed non-blocking network fabric with automated replication across Availability Domains and Fault Domains.
- **Export Paths & Export Options**:
  - A single OCI FSS **Mount Target** can export multiple independent filesystems via distinct **Export Paths** (e.g., `/finance`, `/media`, `/backups`).
  - **Export Options**: Granular NFS security controls configured directly in OCI:
    - Restrict access to specific client IP CIDR blocks (e.g., only `10.0.1.0/24`).
    - Enforce **Root Squashing** (`identity_squash = ROOT`): re-maps incoming root (`uid: 0`) requests to an unprivileged anonymous user (`nobody`), preventing malicious root escalation.
- **Native Snapshot & Clone Architecture**:
  - Supports creating read-only point-in-time snapshots of an entire filesystem in seconds using redirect-on-write pointers.
  - Allows instantaneous **Filesystem Clones**: provisions a writable copy of a petabyte-scale filesystem in under 5 seconds for dev/test environments with zero initial storage duplication.

## 7. Configuration
Comparing managed file system configuration in Terraform across AWS and OCI:

### AWS EFS with Access Point & Elastic Throughput (Terraform)
```hcl
# AWS EFS Filesystem with Elastic Throughput
resource "aws_efs_file_system" "shared_fs" {
  creation_token   = "app-shared-fs"
  performance_mode = "generalPurpose"
  throughput_mode  = "elastic" # Automatically scales throughput!
  encrypted        = true
  kms_key_id       = var.kms_key_arn

  tags = { Name = "production-efs" }
}

# Mount Targets across 3 AZs
resource "aws_efs_mount_target" "mount_az1" {
  file_system_id  = aws_efs_file_system.shared_fs.id
  subnet_id       = var.private_subnet_ids[0]
  security_groups = [var.efs_security_group_id]
}

# Sandboxed Access Point
resource "aws_efs_access_point" "app_ap" {
  file_system_id = aws_efs_file_system.shared_fs.id

  posix_user {
    gid = 1000
    uid = 1000
  }

  root_directory {
    path = "/app-data"
    creation_info {
      owner_gid   = 1000
      owner_uid   = 1000
      permissions = "0755"
    }
  }
}
```

### OCI File Storage Service (FSS) with Export Options (Terraform)
```hcl
# 1. OCI File System
resource "oci_file_storage_file_system" "fss" {
  availability_domain = var.availability_domain
  compartment_id      = var.compartment_id
  display_name        = "app-shared-fss"
  kms_key_id          = var.vault_key_id
}

# 2. Mount Target in Private Regional Subnet
resource "oci_file_storage_mount_target" "mount_target" {
  availability_domain = var.availability_domain
  compartment_id      = var.compartment_id
  display_name        = "fss-mount-target"
  subnet_id           = var.private_subnet_id
  nsg_ids             = [var.fss_nsg_id]
}

# 3. Export with Root Squashing Enabled
resource "oci_file_storage_export" "app_export" {
  export_set_id  = oci_file_storage_mount_target.mount_target.export_set_id
  file_system_id = oci_file_storage_file_system.fss.id
  path           = "/app-shared"

  export_options {
    source          = "10.0.0.0/16" # Restrict to VCN CIDR!
    access          = "READ_WRITE"
    identity_squash = "ROOT"        # Enforces root squash!
    anonymous_uid   = 65534
    anonymous_gid   = 65534
  }
}
```

## 8. Data Flow
```text
Multi-Pod Concurrent NFS Write Flow:
1. Pod 1 (AZ-1) writes: echo "order_100" >> /mnt/shared/ledger.log
2. Linux VFS layer issues standard POSIX file write.
3. In-kernel NFS client translates write to NFS v4.1 COMPOUND RPC call.
4. Transmitted over TCP port 2049 across private subnet to Mount Target AZ-1.
5. Storage Backbone:
   - Acquires distributed write lock.
   - Synchronously replicates block write across 3 AZs/ADs.
   - Releases lock, returns NFS write ACK.
6. Pod 2 (AZ-2) immediately executes: tail -n 1 /mnt/shared/ledger.log
   - Reads "order_100" with zero propagation delay!
```

## 9. Security
- **Defense-in-Depth Network Isolation**:
  - The Mount Target security group / NSG must allow inbound traffic on **TCP/UDP port 2049 (NFS)** strictly from the Application Tier security group.
  - In-Transit TLS Encryption: Enforce `amazon-efs-utils` or stunnel TLS wrappers to encrypt all NFS payloads traversing the internal cloud network.

## 10. Reliability
- **Multi-AZ Durability Advantage**:
  - Unlike EBS volumes which are strictly single-AZ, **Amazon EFS and OCI FSS are multi-AZ/multi-AD by design**.
  - If AZ-1 suffers a total power outage, compute instances in AZ-2 and AZ-3 continue accessing the exact same filesystem through their local Mount Targets with zero data loss and zero manual failover.

## 11. Scaling
- **Throughput Bottlenecks on Small Files**:
  - NFS is a metadata-heavy protocol.
  - Reading 100,000 tiny 2 KB files over NFS requires 100,000 round-trip RPC calls (`LOOKUP`, `GETATTR`, `OPEN`, `READ`, `CLOSE`), resulting in sluggish performance regardless of aggregate network bandwidth.
  - *Optimization*: Cache small metadata files locally in memory or on local SSDs, reserving EFS/FSS for large assets or true shared state.

## 12. Observability
- **Key EFS CloudWatch Metrics**:
  - `PercentIOLimit`: Percentage of I/O limit consumed in General Purpose mode. If hitting 100%, migrate to Max I/O mode.
  - `MeteredIOBytes`: Volume of data read and written.
- **OCI FSS Metrics**: Track `ClientReadBytes`, `ClientWriteBytes`, and `ReadLatency` in OCI Monitoring.

## 13. Cost
- **File Storage Pricing Comparison**:
  - AWS EFS: **\$0.30 / GB-month** for Standard storage tier `[Doc: Amazon EFS Pricing, checked 2026-09-03]`. EFS One Zone costs \$0.16/GB-month.
  - EFS Infrequent Access (IA): **\$0.025 / GB-month** (92% savings with automated lifecycle tiering).
  - OCI File Storage Service (FSS): **\$0.30 / GB-month** for base storage.
  - *FinOps Rule*: Managed file storage is approximately **3x to 4x more expensive than block storage (EBS gp3 at \$0.08/GB)**. Only use EFS/FSS when concurrent multi-instance ReadWriteMany access is an absolute requirement!

## 14. Failure Modes
- **The Bursting Throughput Starvation Trap**: A team provisions a new EFS filesystem containing only 10 GB of data in Bursting Throughput mode. The baseline throughput is a pathetic $10\text{ GB} \times 50\text{ KiB/s} = \mathbf{500\text{ KiB/s}}$! During deployment, 20 container pods attempt to pull shared configuration files simultaneously, exhausting the burst credit balance in 5 minutes. The filesystem throttles to 500 KiB/s, causing all pods to crash on startup. *Remediation: Always use Elastic Throughput.*
- **The Mount Target Deadlock During AZ Outage**: Hardcoding the IP address of the AZ-1 Mount Target in application mount scripts across all AZs. When AZ-1 experiences failure, instances in AZ-2 and AZ-3 lose storage access despite the filesystem being healthy. Mount using the **DNS hostname** (`fs-12345.efs.us-east-1.amazonaws.com`), which automatically resolves to the local AZ's mount target IP.

## 15. Troubleshooting
When compute instances fail to mount EFS or FSS:
1. **Test Port 2049 Reachability**:
   ```bash
   nc -zvw3 <mount-target-ip> 2049
   ```
   If connection times out, the Mount Target security group / NSG is blocking NFS ingress from the client.
2. **Inspect Kernel Mount Errors**:
   - `mount.nfs: access denied by server`: The client IP is not authorized in OCI FSS Export Options, or the AWS IAM Identity Center policy rejects the connection.
   - `mount.nfs: Connection timed out`: Subnet routing issue or missing security group rule.

## 16. Common Mistakes
- **Using EFS/FSS as a Primary Database Disk**: Attempting to host PostgreSQL or MySQL data directories directly on an NFS mount. Database engines rely on low-latency fsync operations. Network NFS latency introduces 2–5ms per transaction, degrading database performance by 90%. Use **EBS / OCI Block Volumes** for relational databases.
- **Forgetting EFS Lifecycle Tiering**: Leaving cold, archived files in EFS Standard tier at \$0.30/GB instead of enabling EFS Lifecycle Management to demote files to EFS Infrequent Access (\$0.025/GB).

## 17. Trade-offs
| Dimension | Managed File Storage (EFS / FSS) | Managed Block Storage (EBS / BV) |
| :--- | :--- | :--- |
| **Concurrent Access**| **Multi-Attach (Thousands of Nodes - RWX)**| Single-Attach (One Instance - RWO) |
| **Latency** | 2–5ms (Network NFS protocol overhead) | Sub-millisecond (< 1ms direct NVMe) |
| **Throughput / IOPS**| Scaled dynamically via elastic fabric | Explicitly provisioned (Up to 300,000 IOPS)|
| **Cost** | \$0.30 / GB-month (High) | \$0.04 – \$0.08 / GB-month (Low) |
| **Blast Radius** | Regional / Multi-AZ resilient | Bound to a single Availability Zone |

## 18. Interview Questions
1. *Why should a relational database (e.g., PostgreSQL or Oracle Database) NEVER be deployed on Amazon EFS or OCI File Storage Service? What exact storage primitive must be used instead?*
2. *Explain the architectural difference between EFS Bursting Throughput and Elastic Throughput. Why does a small filesystem suffer in Bursting mode?*
3. *What are EFS Access Points and OCI Export Options, and how do they enforce container security in multi-tenant Kubernetes clusters?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "Deploying a relational database (like PostgreSQL, MySQL, or Oracle Database) directly onto Amazon EFS or OCI File Storage Service is a severe architectural anti-pattern:
>
> 1. **Latency & fsync Contention**:
>    - Relational database engines guarantee ACID compliance by executing synchronous write operations (`fsync`) to the Write-Ahead Log (WAL) or Redo Log on every committed transaction.
>    - On a network-attached block device (AWS EBS `gp3` or OCI Block Volume), block writes execute over dedicated hardware PCI-e/NVMe controllers (Nitro/SmartNIC) with **sub-millisecond latency ($< 1\text{ms}$)**.
>    - On EFS or FSS, every single write command is translated into an NFS v4.1 network RPC call over TCP port 2049, verified against POSIX metadata locks, and acknowledged across multiple Availability Zones. This introduces **2 to 5 milliseconds of network round-trip latency per transaction**, slashing database write throughput by 80%–95%.
>
> 2. **File Locking & In-Memory Cache Coherency**:
>    - Relational databases are single-writer engines that manage complex internal buffer pools in RAM. They do not require multi-instance ReadWriteMany sharing and can suffer from file locking contention over distributed NFS locks.
>
> 3. **The Mandatory Architecture**:
>    - Production databases must **always be deployed on dedicated Block Storage (AWS EBS `gp3` / `io2` or OCI Block Volume with Balanced/Higher VPUs)**.
>    - EFS and OCI FSS should be reserved strictly for shared, unstructured, multi-pod workloads—such as CMS media uploads, shared scripts, or container persistent assets requiring ReadWriteMany access."

## 20. Hands-on Exercise
**Objective**: Mount an EFS or OCI FSS filesystem inside an EC2/OCI VM and verify concurrent POSIX file locking.

### Verification Steps
1. Provision an EFS or OCI FSS filesystem with an active Mount Target in a private subnet.
2. Connect to a Linux VM in that subnet and install the NFS client: `sudo yum install -y nfs-utils`.
3. Mount the filesystem:
   ```bash
   sudo mkdir -p /mnt/shared
   sudo mount -t nfs4 -o nfsvers=4.1 <mount-target-ip>:/ /mnt/shared
   ```
4. Verify mount and write a test file:
   ```bash
   df -h /mnt/shared
   echo "Distributed Storage Verification: 2026-09-03" | sudo tee /mnt/shared/test.txt
   ```
5. Confirm that the file is created with standard POSIX permissions and readable across multiple concurrent instances.
