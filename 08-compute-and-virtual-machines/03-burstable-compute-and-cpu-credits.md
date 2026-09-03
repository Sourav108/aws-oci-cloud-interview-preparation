# 03. Burstable Compute & CPU Credits

## 1. Problem
Many general-purpose web services, CI/CD runners, and staging environments do not consume 100% of their CPU capacity continuously. Their utilization profile consists of long periods of low idle usage (5%–15% CPU) punctuated by brief, intense traffic spikes (compiling code, processing a batch upload). Deploying fixed-compute instances for these workloads results in paying for 80% idle capacity. Burstable instances (AWS T3/T4g and OCI Burstable Shapes) solve this by providing low-cost baseline performance with the ability to burst to 100% CPU. However, if architects fail to understand the mathematical mechanics of **CPU credit exhaustion** or **burstable caps**, production systems experience sudden, catastrophic CPU throttling that freezes applications.

## 2. Cloud Concept: The CPU Credit Economy
- **Baseline Utilization**: The guaranteed percentage of physical CPU core performance allocated to the instance at all times.
- **CPU Credit Accrual**: When the instance operates **below** its baseline utilization, it earns CPU credits and deposits them into an internal credit balance.
  $$1\text{ CPU Credit} = 1\text{ vCPU running at }100\%\text{ utilization for }1\text{ minute}$$
- **Bursting**: When the workload spikes **above** baseline, the instance consumes accumulated credits from its balance to burst up to 100% CPU without performance degradation.
- **Credit Exhaustion**: Once the credit balance hits zero:
  - *Standard Mode (Throttled)*: The hypervisor forcibly throttles the instance CPU down to the baseline limit (e.g., exactly 20% CPU). The application freezes, dropping requests.
  - *Unlimited Mode (AWS T-Unlimited)*: The instance continues bursting at 100%, but AWS bills extra surcharges per vCPU-hour spent above baseline, triggering shocking end-of-month cloud bills.

## 3. AWS Implementation: T3 / T4g Credit Math
In AWS EC2:
- An instance type like `t3.medium` (2 vCPUs) has a baseline performance of **20% per vCPU** (40% aggregate) `[Doc: Burstable Performance Instances, checked 2026-09-03]`.
- Earns **24 CPU credits per hour** (maximum balance: 576 credits).
- **The "T-Unlimited" Billing Trap**:
  - AWS T3 instances launch in **Unlimited Mode** by default.
  - If a runaway process (e.g., cryptominer or infinite loop) pins CPU at 100% for 30 days, a \$30/month `t3.medium` can silently rack up over **\$200/month in surplus credit charges** (\$0.05 per vCPU-hour)!
  - *Best Practice*: For cost-controlled environments, explicitly set `credit_specification { cpu_credits = "standard" }`.

## 4. OCI Implementation: Burstable Shapes
In Oracle Cloud Infrastructure:
- OCI implements burstable compute directly on its Flexible Shapes architecture (`VM.Standard.E5.Flex` / `VM.Standard3.Flex`) `[Doc: OCI Burstable Instances, checked 2026-09-03]`.
- **Configurable Baseline Caps**:
  - When provisioning an OCI burstable instance, you select the explicit baseline utilization cap:
    - **12.5% Baseline**: Sized for near-idle services, staging nodes, and low-traffic test servers.
    - **50% Baseline**: Sized for active web servers that need periodic headroom.
- **Pricing & Simplicity**:
  - In OCI, pricing is pegged strictly to the baseline fraction (an instance with a 12.5% baseline costs a fraction of a full-core shape).
  - There are no opaque credit banks or surprise surplus charges; instances burst to 100% when spare host capacity is available, delivering transparent, predictable monthly costs.

## 5. Production Failure Modes: Silent CPU Throttling
- **The Staging-to-Production Replication Disaster**: A development team runs load testing on a `t3.large` instance. The test runs for 20 minutes and passes with flying colors because the instance started with a full bank of 576 accumulated credits. Confident in the performance, they launch production on `t3.large` (in Standard Mode). Under sustained 8-hour production traffic, the credit balance drains to zero. At 2:00 PM, the hypervisor throttles CPU to 30%, causing a complete application lockup.

## 6. Troubleshooting & CloudWatch Metrics
When investigating degraded performance on a burstable instance:
1. **Inspect CloudWatch Credit Metrics**:
   - `CPUCreditBalance`: If this graph reaches zero, the instance is in active CPU starvation.
   - `CPUSurplusCreditBalance`: If rising, you are actively burning money in T-Unlimited mode.
2. **Examine `dmesg` and OS Steal Time**:
   - Run `top` or `htop` inside Linux.
   - Look at the `%st` (**CPU Steal Time**) column:
     ```text
     %Cpu(s): 12.0 us, 5.0 sy, 0.0 ni, 0.0 id, 0.0 wa, 0.0 hi, 0.0 si, 83.0 st
     ```
   - An elevated `%st` value ($> 50\%$) proves the cloud hypervisor is actively stealing clock cycles and throttling the guest VM due to credit exhaustion.

## 7. Senior Interview Question & Defense
**Question**: *You observe an application on an EC2 `t3.xlarge` instance experiencing 5-second response latency spikes. `top` shows 30% CPU utilization, but user requests are timing out. What is happening at the hypervisor level, and how do you resolve it permanently?*

**Staff-Level Defense**:
> "This is the classic hallmark of **Burstable CPU Credit Starvation**:
>
> 1. **Root Cause Analysis**:
>    - A `t3.xlarge` (4 vCPUs) has a baseline utilization quota of **40% per vCPU**.
>    - When the application operated above 40% for an extended period, it completely exhausted its `CPUCreditBalance`.
>    - Because the instance was configured in **Standard Mode**, the AWS Nitro hypervisor forcefully capped CPU execution to exactly 40% baseline capacity.
>    - While the guest OS reports 30–40% CPU utilization, the CPU is running at its absolute physical throttle ceiling. The application thread pool deadlocks, queue depths spike, and response latency explodes.
>    - If we inspect `top`, the CPU steal metric (`%st`) will show high values, proving hypervisor-enforced clock cycle starvation.
>
> 2. **Permanent Resolution**:
>    - **Immediate Triage**: Enable **T3 Unlimited Mode** via AWS CLI to immediately lift the CPU throttle ceiling while paying surplus credit fees.
>    - **Architectural Fix**: Burstable instances must **never** be used for sustained production workloads with predictable load. I will immediately resize the instance to a non-burstable fixed-compute instance—specifically a **Compute-Optimized `c6i.xlarge`** or **OCI `VM.Standard.E5.Flex`** with dedicated OCPUs. Dedicated shapes eliminate credit math, eliminate hypervisor throttling, and guarantee 100% sustained CPU performance 24/7."
