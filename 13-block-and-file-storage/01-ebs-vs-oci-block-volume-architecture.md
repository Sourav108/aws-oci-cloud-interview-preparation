# 01. EBS vs. OCI Block Volume Architecture

## 1. Problem
High-performance relational databases (PostgreSQL, Oracle Database, MySQL) and latency-sensitive search engines (Elasticsearch, OpenSearch) require persistent, network-attached block devices that emulate raw physical hard drives or NVMe SSDs. When storage systems are poorly architected, applications hit severe I/O bottlenecks: write operations stall behind choked IOPS limits, backups lock database tables, or companies pay 10x more for storage capacity simply to acquire necessary I/O throughput. Understanding the architectural mechanics of Amazon EBS and OCI Block Volumes allows engineers to extract maximum performance at minimum cost.

## 2. Cloud Concept
### Network-Attached Block Storage
Cloud block volumes are **not** physical SSDs plugged into the motherboard of the compute instance. Instead, a block volume is a **distributed, replicated storage system** running on dedicated storage clusters connected to compute hypervisors across a high-speed optical network fabric:
- **AWS Amazon EBS**: Attaches to EC2 instances via the AWS Nitro Card for EBS over high-speed PCI-e NVMe or iSCSI emulation.
- **OCI Block Volume**: Attaches to OCI compute instances via the OCI SmartNIC over network-attached iSCSI or native paravirtualized controllers.

### The Decoupling of Capacity from Performance
- **The Legacy Problem (AWS gp2)**: On legacy `gp2` volumes, IOPS were strictly tied to disk size ($3\text{ IOPS per GB}$). To obtain 3,000 IOPS, you had to provision a 1 TB volume, even if the database only held 50 GB of data!
- **The Modern AWS gp3 Architecture**:
  - Completely decouples capacity from performance.
  - Baseline: Every `gp3` volume receives **3,000 baseline IOPS and 125 MB/s throughput for free**, regardless of volume size `[Doc: Amazon EBS Volume Types, checked 2026-09-03]`.
  - Architects can provision up to **16,000 IOPS and 1,000 MB/s** independently without paying for extra gigabytes of storage.
- **The OCI Volume Performance Unit (VPU) Architecture**:
  - OCI took performance decoupling to an even more elegant level using **Volume Performance Units (VPUs)** `[Doc: OCI Block Volume Performance, checked 2026-09-03]`:
    - *Lower Cost (0 VPU)*: Optimized for archival/sequential workloads (e.g., big data ETL).
    - *Balanced (10 VPU)*: Default tier delivering up to 25,000 IOPS and 480 MB/s.
    - *Higher Performance (20 VPU)*: Sized for enterprise databases (up to 50,000 IOPS).
    - *Ultra High Performance (30 to 120 VPU)*: Delivers up to **300,000 IOPS and 2,680 MB/s throughput** per volume with sub-millisecond p99 latency!
  - **Dynamic Runtime Tuning**: You can adjust the VPU slider on an active, running OCI Block Volume via API/Terraform in seconds with **zero downtime and zero detachment**.

## 3. Mental Model
Think of cloud block storage as utility water pressure:
- **Storage Capacity (GB)** is the size of the water tank in your backyard.
- **IOPS (Input/Output Operations per Second)** is how many times per second you can open and close the tap (crucial for small, random database record lookups).
- **Throughput (MB/s)** is the diameter of the pipe. Even if you open the tap 10,000 times a second, if the pipe is narrow, only a trickle of water comes out (crucial for sequential backups and table scans).
- **OCI VPUs** is an electric water pump dial: turn the dial to 20 during the daytime business rush for maximum water pressure, and turn it back to 10 at night to save money, all without shutting off the main water line.

## 4. Architecture Diagram
```text
CLOUD BLOCK STORAGE ATTACHMENT & FABRIC:

AWS EC2 NITRO ATTACHMENT:
┌────────────────────────────────────────────────────────────────────────┐
│ EC2 NITRO INSTANCE (m6i.xlarge)                                        │
│   [Linux Kernel: /dev/nvme1n1]                                         │
│          │ Direct PCI-e DMA                                            │
│   [Nitro Card for EBS] ──► Encrypts block with KMS (AES-256)           │
├──────────┼─────────────────────────────────────────────────────────────┤
│          │ Dedicated NVMe-over-Fabrics Network Link                    │
│          ▼                                                             │
│   [Amazon EBS Storage Cluster (Dedicated Storage Fabric)]              │
│   * Replicated across independent storage nodes within the AZ          │
│   * gp3: 3,000–16,000 IOPS | io2 Block Express: Up to 256,000 IOPS     │
└────────────────────────────────────────────────────────────────────────┘

OCI SMARTNIC ATTACHMENT:
┌────────────────────────────────────────────────────────────────────────┐
│ OCI COMPUTE INSTANCE (VM or Bare Metal)                                │
│   [Linux Kernel: /dev/sdb (iSCSI or Paravirtualized)]                  │
│          │                                                             │
│   [Off-Box SmartNIC] ──► Encrypts & routes iSCSI packets               │
├──────────┼─────────────────────────────────────────────────────────────┤
│          │ High-Speed Non-Blocking Clos Network Link                   │
│          ▼                                                             │
│   [OCI Block Volume Cluster (Spans Fault Domains)]                     │
│   * Tuned via VPUs: Balanced (10 VPU) ──► Ultra High (up to 120 VPU)   │
│   * Delivers up to 300,000 IOPS with sub-millisecond latency           │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS Elastic Block Store (EBS):
- **Volume Types**:
  - `gp3 (General Purpose SSD)`: Default standard. Cost-effective, independently provisioned IOPS/throughput. Up to 16,000 IOPS / 1,000 MB/s.
  - `io2 Block Express (Provisioned IOPS SSD)`: Mission-critical tier. Delivers up to **256,000 IOPS**, **4,000 MB/s throughput**, and **99.999% durability** with sub-millisecond latency `[Doc: AWS io2 Block Express, checked 2026-09-03]`.
  - `st1 (Throughput Optimized HDD)`: Low-cost magnetic storage for streaming Big Data and logging ($500\text{ MB/s}$).
  - `sc1 (Cold HDD)`: Lowest-cost magnetic storage for infrequently accessed file servers.
- **EBS Multi-Attach**:
  - Allows attaching an `io2` or `io2 Block Express` volume to up to **16 EC2 instances simultaneously** within the same Availability Zone.
  - *Mandatory Requirement*: The guest OS must use a cluster-aware filesystem (e.g., GFS2, OCFS2) to coordinate write locks and prevent filesystem corruption.

## 6. OCI Implementation
In Oracle Cloud Infrastructure Block Volume:
- **The VPU-Driven Performance Tiers**:
  - OCI simplifies block storage into a single unified volume engine configured by **Volume Performance Units (VPUs)**:
    - *0 VPU (Lower Cost)*: \$0.0255/GB-month. Delivers 2 IOPS/GB. Ideal for sequential batch data.
    - *10 VPU (Balanced)*: \$0.0425/GB-month. Delivers 60 IOPS/GB (up to 25,000 IOPS per volume).
    - *20 VPU (Higher Performance)*: \$0.0595/GB-month. Delivers 75 IOPS/GB (up to 50,000 IOPS per volume).
    - *30–120 VPU (Ultra High Performance)*: Scales up to **300,000 IOPS and 2,680 MB/s per volume** `[Doc: OCI Block Volume Limits, checked 2026-09-03]`.
- **Dynamic Performance Auto-Tuning**:
  - OCI supports **Automated Performance Auto-Tuning**: scheduled policies that automatically adjust VPUs up during business hours (e.g., 20 VPU) and throttle them down to Lower Cost (0 VPU) on weekends, slashing block storage spend by 40% automatically!
- **OCI Read-Write Volume Sharing (Clustering)**:
  - Supports attaching a single Block Volume to up to **32 compute instances** in Read-Write mode (or up to 8 instances for Bare Metal shapes).
  - Used for clustered Oracle RAC databases and active-active file clusters.
- **Volume Groups**:
  - Allows grouping up to 32 volumes together.
  - Takes **crash-consistent snapshots across all volumes simultaneously** with zero I/O freezing, essential for databases spanning multiple data and log drives.

## 7. Configuration
Comparing block storage configuration in Terraform across AWS and OCI:

### AWS EBS gp3 Volume with Independent IOPS (Terraform)
```hcl
# AWS gp3 Volume with customized IOPS and Throughput
resource "aws_ebs_volume" "database_data" {
  availability_zone = "us-east-1a" # Zonal primitive!
  size              = 100          # 100 GB
  type              = "gp3"

  # Independent scaling beyond 3,000 baseline!
  iops       = 6000 # 6,000 IOPS
  throughput = 250  # 250 MB/s throughput

  encrypted  = true
  kms_key_id = var.kms_key_arn

  tags = { Name = "db-data-volume" }
}

resource "aws_volume_attachment" "db_attach" {
  device_name = "/dev/sdf"
  volume_id   = aws_ebs_volume.database_data.id
  instance_id = aws_instance.db_server.id
}
```

### OCI Block Volume with VPU Performance Tuning (Terraform)
```hcl
# OCI Block Volume with Higher Performance (20 VPU)
resource "oci_core_volume" "database_data" {
  compartment_id      = var.compartment_id
  availability_domain = var.availability_domain
  display_name        = "db-data-volume"
  size_in_gbs         = 100

  # Performance tuned via VPUs (Higher Performance = 20 VPUs)
  vpus_per_gb = 20

  kms_key_id = var.oci_vault_key_id
}

# Attach volume using Paravirtualized controller
resource "oci_core_volume_attachment" "db_attach" {
  attachment_type = "paravirtualized" # High-speed off-box attachment!
  instance_id     = oci_core_instance.db_server.id
  volume_id       = oci_core_volume.database_data.id
}
```

## 8. Data Flow
```text
I/O Execution and Latency Path:
1. Application executes: pwrite(fd, buffer, 8192, offset)
2. Linux filesystem issues block write to /dev/nvme1n1.
3. Hardware Intercept:
   - AWS: Nitro Card for EBS intercepts command over PCI-e.
   - OCI: SmartNIC intercepts command over paravirtualized bus.
4. Cryptographic Encryption: Block payload encrypted in hardware with KMS key.
5. Network Transmission: Encrypted packet transmitted over dedicated storage fiber.
6. Replicated Storage Acknowledgment:
   - Written synchronously to multiple storage nodes.
   - Storage cluster returns ACK to Nitro / SmartNIC.
7. Linux kernel receives write completion interrupt (< 1ms elapsed time!).
```

## 9. Security
- **Default Encryption at Rest**:
  - Both EBS and OCI Block Volumes enforce AES-256 encryption.
  - When creating volume snapshots, the snapshots inherit the encryption keys automatically.
  - Revoking KMS key permissions immediately unmounts the volume and halts all I/O.

## 10. Reliability
- **Zonal Availability Constraints**:
  - **AWS EBS volumes are strictly Zonal**. A volume created in `us-east-1a` cannot be attached to an instance in `us-east-1b`.
  - To move data across AZs in AWS, you must snapshot the EBS volume and restore a new volume from that snapshot into the target AZ.
  - In OCI, Block Volumes belong to an Availability Domain. In single-AD regions, volumes can be attached to any instance across all 3 Fault Domains.

## 11. Scaling
- **The EBS-Optimized Instance Bottleneck**:
  - Provisioning a 16,000 IOPS `gp3` volume does not guarantee the instance can process 16,000 IOPS!
  - Every EC2 instance type has an **EBS-Optimized Throughput and IOPS Limit**.
  - A small `t3.medium` caps EBS throughput at **187.5 MB/s and 3,000 IOPS** `[Doc: EBS-Optimized Instances, checked 2026-09-03]`. Attaching a 10,000 IOPS volume to a `t3.medium` results in the host hypervisor dropping 70% of the I/O!
  - *Sizing Rule*: Always verify the EC2 instance's dedicated EBS bandwidth ceiling.

## 12. Observability
- **Volume CloudWatch Metrics**:
  - `VolumeReadOps` / `VolumeWriteOps`: Used to calculate active IOPS.
  - `VolumeThroughputPercentage`: Measures if the volume is hitting its provisioned throughput limit.
  - `VolumeQueueLength`: Number of read/write requests waiting in the operating system queue. A queue length $> 10$ indicates severe disk starvation.

## 13. Cost
- **EBS vs. OCI Block Volume Pricing**:
  - AWS `gp3`: **\$0.08 / GB-month** (includes 3,000 IOPS + 125 MB/s). Additional IOPS cost \$0.005/IOPS-month; additional throughput costs \$0.04/MB/s-month `[Doc: AWS EBS Pricing, checked 2026-09-03]`.
  - AWS `io2`: **\$0.125 / GB-month** + **\$0.065 / provisioned IOPS-month**. A 10,000 IOPS `io2` volume costs over **\$750/month**!
  - OCI Block Volume (Balanced 10 VPU): **\$0.0425 / GB-month**. A 500 GB volume delivering 25,000 IOPS costs **\$21.25/month** in OCI vs. hundreds of dollars in AWS!

## 14. Failure Modes
- **The Multi-Attach Ext4 Filesystem Corruption**: An engineer attaches an `io2` EBS volume to two EC2 instances simultaneously using Multi-Attach, formatting the drive with standard `ext4` or `xfs`. Because standard filesystems maintain uncoordinated in-memory block allocation caches, both instances write to the same disk blocks simultaneously, resulting in catastrophic, irreversible filesystem corruption. *Remediation: Always use cluster-aware filesystems like GFS2 or OCFS2.*
- **The Snapshot Freezing Performance Dip**: Initializing a volume from a large EBS snapshot without **Fast Snapshot Restore (FSR)**. The volume is created instantly, but blocks are lazy-loaded from Amazon S3 on first read, causing database queries to suffer 100ms+ disk read latency.

## 15. Troubleshooting
When a database suffers from slow disk I/O:
1. **Calculate Active IOPS and Throughput**:
   ```bash
   iostat -xz 1
   ```
   Look at `r/s + w/s` (current IOPS) and `rMB/s + wMB/s` (current throughput).
2. **Inspect `%util` and `await`**:
   - If `%util` is **100%** and `await` (I/O latency) is $> 10\text{ms}$, the volume has reached its provisioned IOPS or throughput ceiling.
3. **Verify Host Instance Limit**: Check if the EC2 instance has triggered `EBSIOBalance%` exhaustion on burstable EBS shapes.

## 16. Common Mistakes
- **Paying for io2 When gp3 Suffices**: Over-provisioning expensive `io2` volumes (\$0.065 per IOPS) for workloads that require less than 16,000 IOPS. A `gp3` volume provides identical performance up to 16,000 IOPS at 80% lower cost.
- **Forgetting to Expand Filesystem After Volume Resize**: Resizing an EBS or OCI Block Volume from 100 GB to 200 GB via the cloud console and expecting `df -h` to show the new space. You must run `growpart` and `resize2fs` (or `xfs_growfs`) inside the OS to expand the partition.

## 17. Trade-offs
| Feature | AWS EBS (gp3) | AWS EBS (io2 Block Express) | OCI Block Volume (Ultra High) |
| :--- | :--- | :--- | :--- |
| **Max IOPS** | 16,000 IOPS | 256,000 IOPS | 300,000 IOPS |
| **Max Throughput**| 1,000 MB/s | 4,000 MB/s | 2,680 MB/s |
| **Latency** | Single-digit ms (1–2ms)| Sub-millisecond (< 1ms) | Sub-millisecond (< 1ms) |
| **Pricing Model**| Baseline free; billable add-ons | High per-IOPS charge | Extremely low VPU rate |
| **Dynamic Tuning**| Can increase; cannot decrease for 6 hrs | Can increase; cannot decrease for 6 hrs | **Instant increase/decrease in seconds** |

## 18. Interview Questions
1. *A PostgreSQL database on an EC2 instance with a 10,000 IOPS gp3 volume experiences 20ms I/O latency spikes. CloudWatch shows the volume is only pushing 3,000 IOPS. What is the bottleneck, and how do you diagnose it?*
2. *How do OCI Volume Performance Units (VPUs) fundamentally differ from AWS EBS provisioned IOPS in terms of architectural flexibility and billing economics?*
3. *Why does attaching an EBS volume to two EC2 instances using Multi-Attach corrupt standard ext4 filesystems? What cluster architecture is mandatory?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "If an EBS `gp3` volume provisioned for 10,000 IOPS is bottlenecked at 3,000 IOPS with elevated latency, the constraint is not the EBS volume—the bottleneck is the **EC2 Instance EBS-Optimized Throughput / IOPS Limit**:
>
> 1. **The Architectural Bottleneck**:
>    - EBS traffic flows across a dedicated network bus between the EC2 hypervisor (Nitro Card) and the EBS storage cluster.
>    - Every EC2 instance size enforces a hard, physical hardware ceiling on dedicated EBS bandwidth and IOPS.
>    - For example, an `m5.large` or `c5.large` instance has an EBS-optimized limit capped at **3,600 baseline IOPS** and **187.5 MB/s throughput**, regardless of how much performance was provisioned on the attached disk!
>    - When the database attempts to push 10,000 IOPS, the Nitro Card throttles I/O at the host boundary, causing read/write queues to build up in operating system buffers and driving `await` latency to 20ms+.
>
> 2. **Diagnosis & Verification**:
>    - I check CloudWatch EC2 instance metrics: specifically `EBSIOBalance%` and `EBSByteBalance%`. If these burst-bucket metrics hit 0%, the instance is actively throttled.
>    - On Linux, running `iostat -xz 1` will show `avgqu-sz > 15` while IOPS plateau exactly at the instance's documented EBS ceiling.
>
> 3. **The Remediation**:
>    - I immediately resize the EC2 instance to an instance size supporting at least 10,000 IOPS (e.g., `m6i.2xlarge`, which provides up to 40,000 EBS IOPS and 1,250 MB/s dedicated bandwidth).
>    - This lifts the host I/O throttle ceiling, allowing the volume to achieve its full 10,000 IOPS and returning disk latency to $< 1\text{ms}$."

## 20. Hands-on Exercise
**Objective**: Provision an OCI Block Volume with customized VPUs and dynamically adjust performance without unmounting.

### Verification Steps
1. Provision an OCI Block Volume with `vpus_per_gb = 10` (Balanced) and attach to a running VM.
2. Inside the VM, benchmark baseline IOPS using `fio`:
   ```bash
   fio --name=randwrite --ioengine=libaio --iodepth=16 --rw=randwrite --bs=4k \
       --direct=1 --size=1G --numjobs=4 --runtime=30 --group_reporting --filename=/dev/sdb
   ```
3. Update the volume via OCI CLI to `vpus_per_gb = 20` (Higher Performance):
   ```bash
   oci bv volume update --volume-id <vol-id> --vpus-per-gb 20
   ```
4. Re-run `fio`: observe immediate 50%+ increase in random write IOPS with zero unmounting or OS restarts!
