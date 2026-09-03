# 03. Storage Performance Tuning & Benchmarking

## 1. Problem
Cloud infrastructure vendors market impressive storage numbers: *"up to 256,000 IOPS"* or *"up to 300,000 IOPS"*. However, when engineers attach these volumes to compute instances and run simple file transfer scripts, they frequently achieve less than 10% of advertised performance. Achieving true high-throughput, low-latency storage performance in the cloud is not merely a matter of provisioning a high-tier volume; it requires tuning operating system I/O queue depths, calibrating I/O block sizes, and avoiding host-level compute throughput bottlenecks.

## 2. Cloud Concept: The Storage Performance Equation
Storage performance is governed by three inextricably linked variables:
1. **IOPS (Input/Output Operations per Second)**: The number of discrete read or write operations processed per second. Dominated by small, random I/O (e.g., 4 KB database index lookups).
2. **Throughput (MB/s)**: The total volume of data moved per second:
   $$\text{Throughput (MB/s)} = \frac{\text{IOPS} \times \text{I/O Block Size (Bytes)}}{1,048,576}$$
   - Pushing 10,000 IOPS with a **4 KB** block size yields: $\frac{10,000 \times 4,096}{1,048,576} = \mathbf{39.06\text{ MB/s}}$.
   - Pushing 10,000 IOPS with a **256 KB** block size yields: $\frac{10,000 \times 262,144}{1,048,576} = \mathbf{2,500\text{ MB/s}}$!
3. **Latency & Little's Law for I/O (Queue Depth)**:
   $$\text{Optimal Queue Depth} = \text{Target IOPS} \times \text{Average Latency (Seconds)}$$
   To hit **100,000 IOPS** at **1 millisecond ($0.001\text{s}$)** latency:
   $$\text{Required Queue Depth} = 100,000 \times 0.001 = \mathbf{100\text{ concurrent in-flight I/O requests}}$$
   If an application executes single-threaded, synchronous I/O (`queue depth = 1`), it can **never** exceed $1 / 0.001 = \mathbf{1,000\text{ IOPS}}$, regardless of whether the volume is provisioned for 250,000 IOPS!

## 3. Benchmarking with `fio` (Flexible I/O Tester)
Never benchmark cloud storage with `dd`. `dd` is single-threaded, synchronous, and measures operating system page cache rather than true disk performance. The industry standard tool for cloud storage benchmarking is **`fio`**.

### Standard `fio` Benchmark Configurations:
```bash
# 1. Random Read IOPS Benchmark (Small 4KB Blocks, High Concurrency)
fio --name=randread_iops --filename=/dev/nvme1n1 --ioengine=libaio --direct=1 \
    --bs=4k --iodepth=64 --rw=randread --numjobs=4 --runtime=60 --group_reporting

# 2. Sequential Throughput Benchmark (Large 1MB Blocks)
fio --name=seqwrite_throughput --filename=/dev/nvme1n1 --ioengine=libaio --direct=1 \
    --bs=1024k --iodepth=16 --rw=write --numjobs=2 --runtime=60 --group_reporting
```
- `--direct=1`: Bypasses operating system page cache, forcing I/O directly to the hardware controller.
- `--ioengine=libaio`: Uses Linux asynchronous I/O to keep the hardware queue saturated.

## 4. AWS EBS Performance Tuning
- **Verify EBS-Optimized Ceilings**: Ensure the EC2 instance type has dedicated EBS bandwidth exceeding your volume's provisioned performance.
- **NVMe Driver Tuning in Linux**: Modern Nitro instances expose EBS as NVMe devices (`/dev/nvmeXn1`). Verify that the `nvme_core.default_ps_max_latency_us=0` kernel parameter is set to prevent CPU power-saving states from inducing microsecond I/O latency spikes.

## 5. OCI Block Volume Tuning & Bare Metal NVMe Pass-Through
- **VPU Calibration**: Verify that the volume's VPU setting matches performance goals (e.g., 20 VPUs for 50,000 IOPS).
- **Direct NVMe Pass-Through on DenseIO Bare Metal**:
  - For extreme workloads (e.g., high-frequency trading, multi-terabyte Cassandra/ScyllaDB), use OCI **DenseIO Bare Metal shapes** (`BM.DenseIO.E5`).
  - Provides direct physical PCI-e pass-through to local NVMe drives, delivering **millions of IOPS at $< 20\mu\text{s}$ latency**, completely bypassing network storage virtualization.

## 6. Production Failure Modes: Queue Starvation & Block Size Mismatches
- **The Block Size Truncation Trap**: A data warehouse queries an EBS `gp3` volume provisioned for 125 MB/s throughput using 4 KB block sizes. To achieve 125 MB/s, the volume would need:
  $$\text{Required IOPS} = \frac{125 \times 1,048,576}{4,096} = \mathbf{32,000\text{ IOPS}}$$
  Because `gp3` caps IOPS at 16,000, throughput is physically capped at **62.5 MB/s**!
  - *Remediation*: Increase application read block sizes to 64 KB or 128 KB.

## 7. Senior Interview Question & Defense
**Question**: *You provision an AWS EBS io2 Block Express volume for 50,000 IOPS, but your benchmarking script only achieves 1,000 IOPS with 1ms latency. What is wrong with the benchmark or application, and how do you mathematically prove it?*

**Staff-Level Defense**:
> "The bottleneck is caused by an **I/O Queue Depth Deficit** due to single-threaded synchronous I/O:
>
> 1. **The Mathematical Proof via Little's Law for I/O**:
>    - The fundamental relationship governing storage I/O concurrency is:
>      $$\text{Concurrency (Queue Depth)} = \text{IOPS} \times \text{Latency}$$
>    - If the benchmark script issues one read, waits for completion, and then issues the next read, its operating system **Queue Depth is exactly 1**.
>    - If the volume latency is **1 millisecond ($0.001\text{s}$)**, the maximum mathematically possible IOPS for a single synchronous thread is:
>      $$\text{Max IOPS} = \frac{\text{Queue Depth}}{\text{Latency}} = \frac{1}{0.001\text{s}} = \mathbf{1,000\text{ IOPS}}$$
>    - The remaining 49,000 provisioned IOPS sit completely idle because the application is not submitting enough concurrent in-flight requests to fill the hardware pipeline.
>
> 2. **The Remediation Protocol**:
>    - To achieve the full 50,000 IOPS at 1ms latency, the application or benchmark must maintain an aggregate Queue Depth of:
>      $$\text{Required Queue Depth} = 50,000 \times 0.001\text{s} = \mathbf{50\text{ concurrent in-flight requests}}$$
>    - In `fio`, I re-run the test configuring asynchronous I/O (`--ioengine=libaio`) with `--iodepth=16` across `--numjobs=4` ($16 \times 4 = 64\text{ queue depth}$).
>    - In production application code, this is resolved by using **asynchronous non-blocking I/O frameworks** or increasing database worker thread pools to saturate the storage controller."
