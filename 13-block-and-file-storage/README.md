# Module 13: Block & File Storage (EBS/EFS vs. Block Volume/FSS)

> **Architectural Objective**: *Master cloud network-attached block storage, distributed POSIX network filesystems, I/O performance tuning, and storage economics. Deconstruct Amazon Elastic Block Store (EBS) and Amazon EFS against OCI Block Volume and OCI File Storage Service (FSS), evaluate performance tiering (AWS gp3/io2 vs. OCI Volume Performance Units / VPUs), and master storage benchmarking with `fio`.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. EBS vs. OCI Block Volume Architecture](01-ebs-vs-oci-block-volume-architecture.md)** | Network Block Storage, Volume Types (gp3, io2 Block Express vs. OCI VPUs: Balanced, Higher, Ultra High Performance), Multi-Attach Clustering | Full 20-Section Deep Dive (~2,400 words) |
| **[02. EFS vs. OCI File Storage Service (FSS)](02-efs-vs-oci-file-storage-service.md)** | Managed NFS v4.1 Filesystems, Throughput Modes (Bursting vs. Provisioned vs. Elastic), Mount Targets, POSIX Locking & Kubernetes Integration | Full 20-Section Deep Dive (~2,200 words) |
| **[03. Storage Performance Tuning & Benchmarking](03-storage-performance-tuning-and-benchmarking.md)** | Benchmarking with `fio`, IOPS vs. Throughput vs. Latency Math ($Q = \text{IOPS} \times \text{Latency}$), EBS-Optimized Bottlenecks | Abbreviated Storage Engineering Guide (~900 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **The gp3 Independent Scaling Model**: How AWS gp3 decoupled storage capacity from IOPS/throughput (breaking the gp2 ratio constraint), and how OCI's Volume Performance Units (VPUs) provide elastic, runtime performance scaling with zero downtime.
2. **Block vs. File Storage Decision Matrix**: When to choose single-attach block devices (EBS / Block Volume) for high-IOPS relational databases versus multi-attach NFS filesystems (EFS / FSS) for container pods.
3. **The I/O Queue Depth Equation**: How to tune Linux disk queue depths and block sizes to hit advertised cloud storage IOPS and throughput ceilings without inducing latency spikes.
