# Module 08: Compute & Virtual Machines (EC2 vs. OCI Compute Shapes)

> **Architectural Objective**: *Master cloud compute virtualization, hardware offloading architectures, instance lifecycle management, and financial capacity models. Deconstruct the mechanics of AWS Nitro and EC2 instance families against OCI Bare Metal, VM Shapes, and Flexible Shapes (`VM.Standard.E5.Flex`), evaluate purchasing economics (Savings Plans/RIs vs. OCI Universal Credits and Spot vs. Preemptible instances), and master burstable CPU credit math.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. EC2 vs. OCI Compute Shapes & Nitro Architecture](01-ec2-vs-oci-compute-shapes-and-nitro.md)** | Hardware Virtualization, Nitro Cards vs. OCI SmartNICs, Flexible Shapes, Tenancy Models (Multi-Tenant vs. Dedicated vs. Bare Metal) | Full 20-Section Deep Dive (~2,400 words) |
| **[02. Instance Lifecycle & Purchasing Economics](02-instance-lifecycle-and-purchasing-economics.md)** | Lifecycle States, Reboot vs. Stop/Start Mechanics, Spot (2-min warning) vs. Preemptible (30-sec warning), Savings Plans vs. Universal Credits | Full 20-Section Deep Dive (~2,200 words) |
| **[03. Burstable Compute & CPU Credits](03-burstable-compute-and-cpu-credits.md)** | AWS T3/T4g Credit Math (Accrual, Baseline, T-Unlimited Surprises) vs. OCI Burstable Shapes (12.5% & 50% Baselines), Throttling Debugging | Abbreviated Compute Guide (~900 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **Offloading Hypervisor Overhead**: How AWS Nitro and OCI SmartNIC off-box virtualization reclaim 100% of host CPU and memory for customer workloads, virtually eliminating the "noisy neighbor" hypervisor tax.
2. **Flexible Shapes vs. Rigid Instance Types**: Why OCI Flexible Shapes eliminate the architectural over-provisioning dilemma (e.g., needing 32 GB RAM but only 2 CPUs), and how to size workloads for maximum unit cost efficiency.
3. **Spot vs. Preemptible Interruption Handling**: How to design fault-tolerant, horizontally auto-scaling worker clusters that handle AWS 2-minute Spot termination notices and OCI 30-second Preemptible termination signals without data loss.
