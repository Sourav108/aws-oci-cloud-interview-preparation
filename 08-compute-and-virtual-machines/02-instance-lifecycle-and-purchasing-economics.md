# 02. Instance Lifecycle & Purchasing Economics

## 1. Problem
Compute infrastructure represents 60% to 80% of an enterprise cloud bill. Running all workloads on standard On-Demand pricing wastes hundreds of thousands of dollars annually. Conversely, naively deploying stateful databases or payment processing services on Spot or Preemptible instances triggers catastrophic data corruption when cloud providers reclaim compute capacity during peak regional demand. Furthermore, engineers frequently confuse the lifecycle mechanics of an OS reboot with an API Stop/Start, inadvertently wiping ephemeral drives or releasing vital public IP addresses.

## 2. Cloud Concept
### The Cloud Compute Lifecycle State Machine
A virtual machine moves through distinct operational states across its lifecycle:
1. **Pending / Provisioning**: The cloud control plane finds a physical server chassis with available CPU and RAM matching the shape, attaches virtual interfaces (ENIs/VNICs), and boots the guest OS. Billed only once `Running`.
2. **Running**: Fully operational; instance accumulates hourly compute and license charges.
3. **Stopping**: Graceful shutdown triggered. The OS flushes filesystem buffers and terminates processes.
4. **Stopped**:
   - The instance CPU and RAM allocations on the physical motherboard are **completely released**.
   - You **stop paying for compute (vCPU/OCPU)** `[Doc: AWS EC2 Pricing, checked 2026-09-03]`.
   - You **continue paying for persistent block storage (EBS / OCI Block Volumes)** attached to the instance.
   - *Crucial Network Behavior*: Non-elastic public IPv4 addresses are **immediately released back to the cloud pool**. Upon restart, the instance receives a brand new public IP.
5. **Shutting-down / Terminating**: Virtual disks are deleted (if `delete_on_termination = true`), network interfaces detached, and metadata wiped.
6. **Terminated**: Permanent state; instance cannot be restarted.

### Cloud Purchasing Models
| Model | AWS Equivalent | OCI Equivalent | Discount vs. On-Demand | Commitment / Trade-off |
| :--- | :--- | :--- | :---: | :--- |
| **On-Demand** | On-Demand | On-Demand (Pay-As-You-Go) | 0% (Base Price) | Zero commitment; maximum flexibility; highest cost |
| **Commitment-Based**| Compute Savings Plans / EC2 RIs | Annual Universal Credits (Commitment) | 30% to 72% | 1-year or 3-year financial commitment (\$X/hr spend) |
| **Excess / Interruptible**| **Spot Instances** | **Preemptible Instances** | **70% to 90%** | Cloud provider can reclaim capacity at any moment |

### Spot vs. Preemptible Interruption Warnings
- **AWS Spot Instances**: AWS gives a **2-minute interruption notice** via Instance Metadata Service (`http://169.254.169.254/latest/meta-data/spot/instance-action`) and Amazon EventBridge before forcefully terminating or stopping the instance `[Doc: AWS Spot Instance Interruptions, checked 2026-09-03]`.
- **OCI Preemptible Instances**: OCI gives a **30-second termination notice** via the Instance Metadata Service (`http://169.254.169.254/opc/v1/instance/`) before preemption `[Doc: OCI Preemptible Instances, checked 2026-09-03]`.

## 3. Mental Model
Think of cloud compute purchasing models as hotel room booking strategies:
- **On-Demand** is walking up to the front desk at 11 PM and paying the full rack rate for one night. You can leave whenever you want, but you pay maximum price.
- **Savings Plans / Commitments** is signing a long-term corporate lease guaranteeing you will rent 50 rooms every day for the next 2 years. The hotel grants you a 60% bulk discount.
- **Spot / Preemptible** is booking an unallocated standby room for a 90% discount, with the explicit contract agreement that if a full-paying guest arrives, the hotel can knock on your door and give you 2 minutes (or 30 seconds) to pack your bags and leave.

## 4. Architecture Diagram
```text
COMPUTE LIFECYCLE & SPOT INTERRUPTION ARCHITECTURE:

[Pending / Provisioning]
          │
          ▼
    [RUNNING STATE] ◄──────────────────────┐
    (Billed Compute & Storage)             │
          │                                │ API Start
          ├──► Reboot (OS Level)           │ (Migrates to NEW Hardware Chassis!)
          │    * Stays on SAME chassis!    │
          │                                │
          ▼ API Stop                       │
    [STOPPED STATE] ───────────────────────┘
    * Compute Billing: $0.00
    * Storage Billing: Continues (EBS / BV)
    * Dynamic Public IP: RELEASED!

SPOT / PREEMPTIBLE INTERRUPTION WORKFLOW:
Cloud Engine reclaims capacity
          │
          ▼ (AWS: 2-min warning / OCI: 30-sec warning)
[Instance Metadata / EventBridge Interruption Signal]
          │
          ▼ Local Daemon catches signal
[Graceful Shutdown Script]
  ├──► 1. Deregisters from Load Balancer / Target Group
  ├──► 2. Pauses worker consumption from Kafka / SQS Queue
  ├──► 3. Checkpoints stateful task progress to S3 / Object Storage
  └──► 4. Exits cleanly before hypervisor hardware reclamation!
```

## 5. AWS Implementation
In AWS:
- **Savings Plans vs. Reserved Instances (RIs)**:
  - *Compute Savings Plans*: The modern standard. Requires committing to a dollar-per-hour spend (e.g., \$10/hr) for 1 or 3 years. Automatically applies across any EC2 instance family (regardless of region, OS, or instance size), AWS Fargate, and AWS Lambda. Maximum flexibility.
  - *EC2 Instance Savings Plans*: Commits to a specific instance family in a specific region (e.g., `m6i` in `us-east-1`). Higher discount (up to 72%), lower flexibility.
  - *Standard RIs*: Rigid legacy model; can be sold on the AWS RI Marketplace.
  - *Convertible RIs*: Allows exchanging for different instance families.
- **Spot Instances & Spot Fleets**:
  - Leverages spare EC2 capacity. Pricing fluctuates based on long-term supply and demand trends.
  - **EC2 Auto Scaling Mixed Instances Policy**: Best practice for production Spot adoption:
    - Set `OnDemandBaseCapacity` (e.g., 20% baseline On-Demand instances for core stability).
    - Distribute Spot across multiple instance types (e.g., `c6i.xlarge`, `c6a.xlarge`, `m6i.xlarge`) and multiple AZs using the `capacity-optimized` allocation strategy, dramatically minimizing simultaneous interruption risk.

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **OCI Universal Credits & Annual Commitments**:
  - OCI unifies cloud billing under **Universal Credits (UCM)** `[Doc: OCI Universal Credits, checked 2026-09-03]`.
  - Customers commit to an annual spend threshold (e.g., \$100,000/year).
  - *Massive Architectural Advantage*: Unlike AWS Reserved Instances which lock discounts to specific compute instances or regions, **OCI Universal Credits apply flexibly across ALL services and ALL global regions** (Compute, Autonomous DB, Object Storage, Networking).
- **OCI Preemptible Instances**:
  - Equivalent to AWS Spot. Delivers up to **50% to 80% discounts** compared to on-demand compute rates.
  - Available across VM shapes (including Flexible shapes) and Bare Metal shapes.
  - Reclaimed when capacity is required by standard on-demand workloads.
  - OCI sends an ACPI shutdown signal to the OS and publishes an interruption event to the metadata service 30 seconds prior to preemption.
- **Capacity Reservations**:
  - OCI allows creating **Reserved Capacity** across specific Availability Domains and Fault Domains without running active VMs, guaranteeing that compute capacity is available during disaster recovery failover events without paying full instance run costs.

## 7. Configuration
Comparing Spot and Preemptible instance configuration in Terraform across AWS and OCI:

### AWS EC2 Auto Scaling Mixed Instances Policy (Terraform)
```hcl
# Launch Template for Worker Nodes
resource "aws_launch_template" "worker" {
  name_prefix   = "spot-worker-"
  image_id      = var.ami_id
  instance_type = "c6i.xlarge"
  tags          = { Role = "batch-worker" }
}

# Auto Scaling Group mixing On-Demand baseline with diverse Spot types
resource "aws_autoscaling_group" "spot_asg" {
  name                = "batch-processing-asg"
  vpc_zone_identifier = var.private_subnet_ids
  min_size            = 2
  max_size            = 20
  desired_capacity    = 4

  mixed_instances_policy {
    instances_distribution {
      on_demand_base_capacity                  = 1 # 1 guaranteed On-Demand
      on_demand_percentage_above_base_capacity = 0 # 100% Spot for the rest!
      spot_allocation_strategy                 = "capacity-optimized"
    }

    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.worker.id
        version            = "$Latest"
      }

      # Instance type diversification to survive spot pool reclaims
      override { instance_type = "c6i.xlarge" }
      override { instance_type = "c6a.xlarge" }
      override { instance_type = "c7g.xlarge" }
      override { instance_type = "m6i.xlarge" }
    }
  }
}
```

### OCI Preemptible Compute Instance (Terraform)
```hcl
# OCI Preemptible Instance Configuration
resource "oci_core_instance" "preemptible_worker" {
  compartment_id      = var.compartment_id
  availability_domain = var.availability_domain
  display_name        = "preemptible-batch-worker"
  shape               = "VM.Standard.E5.Flex"

  shape_config {
    ocpus         = 4
    memory_in_gbs = 16
  }

  # Mark instance as Preemptible (Up to 80% discount!)
  preemptible_instance_config {
    preemption_action {
      type                    = "TERMINATE"
      preserve_boot_volume    = false
    }
  }

  source_details {
    source_type = "image"
    source_id   = var.image_id
  }
}
```

## 8. Data Flow
```text
Graceful Interruption Handler Flow on Spot / Preemptible Instance:
1. Cloud Control Plane triggers preemption signal.
2. Local Daemon polls Metadata Service every 5s:
   curl -s http://169.254.169.254/latest/meta-data/spot/instance-action (AWS)
   curl -s http://169.254.169.254/opc/v1/instance/ (OCI)
3. HTTP 200 returned with JSON: {"action": "terminate", "time": "2026-09-03T12:02:00Z"}
4. Daemon triggers local script:
   - Signals container runtime: docker stop -t 20 (SIGTERM sent to app)
   - App flushes in-memory queue offsets to Kafka.
   - App commits uncompleted batch job status to database as 'interrupted'.
5. 30–120 seconds elapse: Hypervisor terminates instance safely with ZERO corrupt state.
```

## 9. Security
- **Data Remanence & Boot Volume Erasure**:
  - When an instance is terminated, cloud providers cryptographically wipe physical flash blocks before reallocating storage to other tenants.
  - Enforce EBS and OCI Block Volume KMS encryption with Customer Managed Keys (CMK). When an instance is terminated, revoking the KMS key renders all historical snapshots permanently unreadable.

## 10. Reliability
- **The Spot Interruption Golden Rule**:
  - Never run single-instance stateful workloads (PostgreSQL primary, MongoDB master, Consul leader) on Spot or Preemptible instances.
  - Restrict Spot/Preemptible to **stateless, idempotent, horizontally scalable workloads**:
    1. Asynchronous batch processing workers (SQS/Kafka consumers).
    2. CI/CD test runners and build agents.
    3. Machine learning training checkpoints and distributed rendering.
    4. Stateless web microservices behind load balancers with active connection draining.

## 11. Scaling
- **Spot Diversification Strategy**:
  - Spot capacity pools are divided by **Region $\times$ AZ $\times$ Instance Type**.
  - If you request 100 instances of purely `c6i.xlarge` in `us-east-1a`, AWS may exhaust that specific pool and reclaim all 100 instances simultaneously.
  - By configuring an ASG across 3 AZs and 4 different instance types (e.g., `c6i`, `c6a`, `c5`, `m6i`), you draw from $3 \times 4 = \mathbf{12\text{ independent Spot pools}}$, reducing simultaneous interruption probability to near zero.

## 12. Observability
- **Tracking Spot Interruptions**:
  - AWS EventBridge rule capturing `EC2 Spot Instance Interruption Warning` events. Trigger Lambda to update Slack/PagerDuty.
  - CloudWatch metric: `InstanceInterruptionCount`.
  - OCI Audit & Events: Capture `com.oraclecloud.computeapi.instancepreempt` events.

## 13. Cost
- **Annual Financial Comparison (100 Compute Instances)**:
  - 100 x `m6i.xlarge` On-Demand: $100 \times \$0.192/\text{hr} \times 8,760\text{ hrs} = \mathbf{\$168,192/\text{year}}$.
  - 100 x `m6i.xlarge` 3-Year Compute Savings Plan (55% off): $\mathbf{\$75,686/\text{year}}$ (Savings: \$92,506).
  - 100 x `m6i.xlarge` Spot / Preemptible (80% off): $\mathbf{\$33,638/\text{year}}$ (Savings: \$134,554).
- Staff architects blend these models: 30% Savings Plans (baseline steady-state load) + 70% Spot/Preemptible (burst and asynchronous batch).

## 14. Failure Modes
- **The Ephemeral Public IP Loss Trap**: An administrator launches a critical web server without an Elastic IP (AWS) or Reserved Public IP (OCI). During routine maintenance, they execute an API Stop and Start. The instance re-boots with a brand new, random public IP. External DNS records now point to an unallocated IP, breaking global customer traffic. *Remediation: Always bind Elastic / Reserved IPs to public services.*
- **The 30-Second OCI Preemption Race Condition**: A container process running on an OCI Preemptible instance requires 45 seconds to checkpoint data to Object Storage. OCI terminates the instance at second 30, truncating the upload and leaving corrupt, unrecoverable partial checkpoint data.

## 15. Troubleshooting
When Spot instances fail to launch or terminate unexpectedly:
1. **Inspect Scaling Activity History**:
   ```bash
   aws autoscaling describe-scaling-activities --autoscaling-group-name <asg-name>
   ```
   Look for `StatusMessage`: `InsufficientInstanceCapacity` indicates the specific Spot pool is completely exhausted in that AZ.
2. **Review EventBridge History**: Verify whether the termination was an intentional scale-in event or an involuntary cloud spot reclaim.
3. **Check Spot Allocation Strategy**: If set to `lowest-price`, the ASG aggressively pools all instances into the cheapest type, maximizing interruption blast radius. Switch immediately to `capacity-optimized`.

## 16. Common Mistakes
- **Using 1-Year RIs for Rapidly Evolving Workloads**: Committing to 3-year Standard RIs for older generation instances (e.g., `m5.large`). When AWS releases newer, faster, and cheaper generations (`m6i`, `m7g`), the company is locked into paying for obsolete hardware for another two years. Use **Compute Savings Plans**.
- **Forgetting that Stopped Instances Incur Storage Fees**: Believing that stopping 50 unused development instances eliminates all costs. The attached 500 GB EBS / Block Volumes continue accumulating storage charges every hour until permanently terminated.

## 17. Trade-offs
| Purchasing Model | Cost Reduction | Availability Risk | Operational Overhead | Best Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **On-Demand** | None (0%) | None (Highest availability) | Lowest | New unprofiled apps; short-term testing |
| **Savings Plans / UCM** | 30% – 72% | None (Reserved capacity) | Low (Financial tracking only) | Core production databases, steady-state web fleets |
| **Spot / Preemptible** | 70% – 90% | High (Reclaimed on 30–120s notice)| High (Requires automated fault tolerance) | Asynchronous batch queues, CI/CD pipelines, stateless workers |

## 18. Interview Questions
1. *You are tasked with slashing your organization's \$2,000,000 annual EC2/Compute spend by 50%. Walk me through your portfolio allocation strategy across On-Demand, Savings Plans, and Spot instances.*
2. *What is the exact technical difference between an OS-level reboot and an API Stop followed by Start? Trace what happens to the physical hardware, public IP, and ephemeral disk storage.*
3. *How do AWS Spot Instances and OCI Preemptible Instances differ in their interruption warning mechanisms, and how must application shutdown code be architected to handle both?*

## 19. Interview Answer
**Exemplary Answer to Question 2**:
> "The architectural difference between an **OS-level reboot** and an **API Stop/Start** fundamentally impacts the underlying physical chassis, network addressing, and storage persistence:
>
> 1. **OS-Level Reboot (`sudo reboot` / `aws ec2 reboot-instances`)**:
>    - **Physical Chassis**: The VM remains hosted on the **exact same physical motherboard and hypervisor**. The OS kernel simply resets.
>    - **Network Addressing**: Both private and public IP addresses (including dynamic public IPs) are **fully preserved**.
>    - **Storage**: Ephemeral instance store volumes (local NVMe SSDs) and in-memory caches remain fully intact and operational.
>    - **Billing**: Compute billing continues uninterrupted.
>
> 2. **API Stop followed by Start (`aws ec2 stop-instances` then `start-instances`)**:
>    - **Physical Chassis**: The instance is completely de-provisioned from the physical motherboard. CPU cores and RAM are released back to the cloud pool. When started, the cloud placement engine allocates a **completely new, healthy physical server chassis** in the same Availability Zone.
>    - **Network Addressing**: Private IP addresses are preserved. However, **dynamic (non-elastic) public IPv4 addresses are permanently released** back to the cloud pool. The instance receives a brand new public IP upon reboot (unless bound to an Elastic/Reserved IP).
>    - **Storage**:
>      - Network block storage (EBS / OCI Block Volume) detaches gracefully and re-attaches to the new chassis with zero data loss.
>      - **Ephemeral Instance Store Drives are completely wiped and cryptographically erased**. Any data stored on local NVMe drives is permanently lost.
>    - **Billing**: Compute billing drops to \$0.00 while stopped; persistent block storage charges continue.
>
> In production, an OS reboot is used for OS kernel patches, while an API Stop/Start is mandatory to recover from underlying physical hardware degradation (`StatusCheckFailed_System`)."

## 20. Hands-on Exercise
**Objective**: Deploy a Spot / Preemptible instance and simulate graceful interruption handling using local metadata polling.

### Verification Steps
1. Launch an EC2 Spot instance or OCI Preemptible instance with a user-data startup script.
2. Run a background Python daemon that polls the local metadata service every 2 seconds:
   ```python
   import urllib.request, json, os, time
   while True:
       try:
           # Poll AWS Spot Interruption endpoint
           req = urllib.request.urlopen("http://169.254.169.254/latest/meta-data/spot/instance-action", timeout=1)
           data = json.loads(req.read().decode())
           print("INTERRUPTION SIGNAL RECEIVED! Saving checkpoint...")
           os.system("touch /tmp/checkpoint_saved && sync")
           break
       except:
           time.sleep(2)
   ```
3. In the cloud console, initiate a Spot termination.
4. Verify that the daemon captures the HTTP 200 event, executes the checkpoint save, and exits cleanly before the hypervisor terminates the VM.
