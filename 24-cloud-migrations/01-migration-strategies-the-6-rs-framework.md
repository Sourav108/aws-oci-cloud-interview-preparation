# Enterprise Cloud Migration Strategies: The 6 R's Framework & Server Migration (AWS vs. OCI)

---

## 1. Executive Summary & Core Definitions

Enterprise cloud migration is the strategic orchestration of transitioning digital business assets—including compute workloads, data storage, network topologies, and business applications—from on-premises legacy data centers or private co-locations into hyperscale public cloud environments. A successful migration is not measured merely by the technical transfer of gigabytes; it is governed by business agility, total cost of ownership (TCO) optimization, security posture enhancement, and operational resilience.

To manage the complexity of evaluating hundreds or thousands of heterogeneous enterprise applications, the cloud industry standardizes on **The 6 R's Migration Framework** originally popularized by AWS and adopted across modern cloud architecture.

```
+---------------------------------------------------------------------------------------------------+
|                                  THE 6 R's MIGRATION SPECTRUM                                     |
+---------------------------------------------------------------------------------------------------+
| LOW EFFORT / MINIMAL CLOUD BENEFIT  <=======================>  HIGH EFFORT / MAXIMUM CLOUD VALUE   |
|                                                                                                   |
| [ RETIRE ]     Decommission redundant or abandoned applications (Immediate cost savings)          |
| [ RETAIN ]     Keep on-premises due to compliance, latency, or unamortized hardware depreciation  |
| [ REHOST ]     Lift & Shift: Move VMs as-is via block-level replication (AWS MGN / OCI Migration)  |
| [ REPLATFORM ] Lift & Reshape: Adopt managed PaaS (RDS / Base DB, Container Instances)            |
| [ REPURCHASE ] Drop & Shop: Replace custom software with commercial SaaS (Salesforce, Workday)    |
| [ REFACTOR ]   Re-architect: Break monoliths into cloud-native microservices, serverless, & APIs  |
+---------------------------------------------------------------------------------------------------+
```

### Core Terminology
* **Total Cost of Ownership (TCO)**: The comprehensive financial calculation comparing on-premises physical data center capital expenditure (CapEx: servers, SAN storage, cooling, power, rack space, real estate, hardware maintenance contracts) against cloud operational expenditure (OpEx: compute, storage, egress bandwidth, managed service fees).
* **AWS Application Migration Service (AWS MGN)**: The primary, automated lift-and-shift service that executes continuous block-level data replication from physical, virtual, or cloud sources into an AWS staging area without interrupting production workloads [Doc: AWS MGN, checked 2026].
* **OCI Cloud Migration Service**: A fully managed OCI service that automates discovery, planning, and migration of on-premises virtual machines (VMware vSphere) directly to OCI Compute instances using an agentless replication appliance [Doc: OCI Cloud Migration, checked 2026].
* **Migration Wave**: A grouped batch of applications scheduled for concurrent migration, clustered by network dependency mapping, data affinities, and business impact risk.
* **Cutover Window**: The designated maintenance window during which production DNS and traffic are shifted from the on-premises source to the newly launched cloud target instances, preceded by final delta block replication.

---

## 2. Distributed Systems Theory & Architecture

### The Three Phases of Enterprise Cloud Migration

```
+-------------------+      +-------------------+      +-----------------------------------------+
| PHASE 1: ASSESS   | ---> | PHASE 2: MOBILIZE | ---> | PHASE 3: MIGRATE & MODERNIZE            |
+-------------------+      +-------------------+      +-----------------------------------------+
| - Portfolio Audit |      | - Landing Zone    |      | - Migration Wave Planning               |
| - TCO Modeling    |      | - DirectConnect / |      | - Block Replication (AWS MGN / OCI CMS) |
| - 6 R's Mapping   |      |   FastConnect     |      | - Non-Disruptive Testing                |
| - Discovery Tools |      | - IAM & Security  |      | - Production Cutover & Modernization    |
+-------------------+      +-------------------+      +-----------------------------------------+
```

### The 6 R's Architectural Deep Dive

```
+---------------+-------------------+---------------------------------------------------------------+
| Strategy      | Migration Vector  | Architectural Rationale & Trade-Offs                          |
+---------------+-------------------+---------------------------------------------------------------+
| **Rehost**    | Lift & Shift      | Exact bit-for-bit VM clone. Fastest time-to-market. Zero code |
|               | (Block-level)     | changes. Retains existing OS bugs and technical debt.          |
| **Replatform**| Lift & Reshape    | Swap self-hosted middleware for managed services (e.g., EC2   |
|               | (PaaS Migration)  | MySQL -> AWS RDS; VM WebLogic -> OCI WebLogic Cloud Service). |
| **Repurchase**| Drop & Shop       | Retire custom legacy application in favor of SaaS solution    |
|               | (SaaS Transition) | (e.g., in-house ticketing system -> ServiceNow or Jira).      |
| **Refactor**  | Re-architect      | Completely rewrite application into serverless functions,     |
|               | (Cloud-Native)    | microservices, and event-driven architecture. Highest ROI.    |
| **Retain**    | Do Nothing        | Keep application in on-prem data center. Defer migration due  |
|               | (Hybrid Keep)     | to sub-millisecond mainframe latency or regulatory mandates.  |
| **Retire**    | Decommission      | Identify obsolete, zombie, or redundant workloads during     |
|               | (Sunset Systems)  | discovery and terminate them. Frees 10-20% IT budget instantly.|
+---------------+-------------------+---------------------------------------------------------------+
```

---

## 3. Core Mechanics & Deep Dive

### Block-Level Replication Mechanics (AWS MGN)

AWS Application Migration Service utilizes continuous block-level asynchronous replication that operates transparently beneath the operating system filesystem:

```
[ On-Premises Host / VM ]                              [ AWS Staging Area (Dedicated VPC) ]
[ Application Writes I/O ]
           |
           v
+-----------------------+                              +-------------------------------+
| AWS Replication Agent |                              | Lightweight Replication Server|
| (Kernel Driver Hook)  | --- TLS 1.3 / TCP 1500 ----> | (t3.small EC2 Instance)       |
+-----------------------+                              +---------------+---------------+
           |                                                           |
  Intercepts Disk Blocks                                  Writes dirty blocks to
  (Continuous CBT)                                        Low-Cost Staging EBS Volumes
                                                                       |
                                                                       v
                                                       +-------------------------------+
                                                       | Staging EBS Volumes           |
                                                       | (Continuous Near-Real-Time)   |
                                                       +-------------------------------+
                                                                       |
                                                           [ Trigger Test / Cutover ]
                                                                       v
                                                       +-------------------------------+
                                                       | Target Production EC2 Instance|
                                                       | - Injected AWS Hypervisor     |
                                                       |   Drivers (ENA / NVMe)        |
                                                       | - Converted to Target Shape   |
                                                       +-------------------------------+
```

1. **Agent Installation**: A lightweight agent containing a kernel-level filter driver is installed on the source physical or virtual machine.
2. **Initial Block Sync**: The agent reads all physical storage sectors sequentially and streams them to an EC2 Replication Server residing in a low-cost staging VPC over port 1500 (encrypted via TLS 1.3).
3. **Continuous Changed Block Tracking (CBT)**: Once the initial sync completes, the agent captures memory-resident disk write deltas in real-time. Lag between source and staging EBS is typically **under 1 second**.
4. **Target Machine Conversion**: During cutover, AWS MGN creates point-in-time EBS snapshots from the staging volumes, attaches them to newly spawned target EC2 instances, and automatically injects AWS cloud drivers (ENA network drivers and NVMe storage drivers).

---

### OCI Cloud Migration Service Mechanics

OCI provides native, agentless migration tailored for VMware environments:

```
[ On-Premises VMware vSphere ]                         [ OCI Tenancy (Target Compartment) ]
+------------------------------------+
| VMware vCenter / ESXi Hosts        |
| [ Source VMs: Win / Linux ]        |
+------------------+-----------------+
                   | Local vCenter API
                   v
+------------------------------------+                 +-------------------------------+
| OCI Migration Remote Appliance     | -- FastConnect/ | OCI Cloud Migration Control   |
| (Virtual Appliance OVA on vSphere) | -- IPSec VPN -> | Plane (Asset Inventory & Plans)|
+------------------------------------+                 +---------------+---------------+
                   | Snapshot Export                                   |
                   +--- Asynchronous Block Stream -------------------->+
                                                                       v
                                                       +-------------------------------+
                                                       | OCI Block Volumes (Staging)   |
                                                       +-------------------------------+
                                                                       |
                                                              [ Execute Cutover Plan ]
                                                                       v
                                                       +-------------------------------+
                                                       | OCI Compute Instances         |
                                                       | (Standard/Flexible Shapes)    |
                                                       +-------------------------------+
```

1. **Remote Appliance Deployment**: Deploy the pre-built OCI Remote Discovery and Replication Appliance (delivered as an OVA template) directly inside the on-premises VMware vSphere cluster.
2. **Agentless Discovery**: The appliance queries VMware vCenter APIs to catalog CPU, memory, OS versions, disk sizes, and network port groups for all running VMs.
3. **Asset Inventory & Cost Estimation**: OCI Cloud Migration automatically maps on-premises VM resource profiles to optimal OCI Compute Shapes (e.g., `VM.Standard3.Flex` matching exact OCPUs and memory), providing real-time cost forecasts.
4. **Snapshot Replication**: Uses VMware vSphere Changed Block Tracking (CBT) snapshots to asynchronously replicate VMDK virtual disks directly to OCI Block Volumes in the target compartment.

---

## 4. Architecture & Data Flow Diagrams

### End-to-End Enterprise Migration Wave Timeline

```
[ T-60 Days: Discovery & Wave Planning ]
Discovery agents deploy -> Network dependency graph constructed -> Wave 1 (30 VMs) approved
                           |
[ T-14 Days: Baseline Replication ]
AWS MGN / OCI Migration appliance initiated -> Initial baseline disk sync completes
Replication enters continuous delta replication mode (Lag < 5 seconds)
                           |
[ T-7 Days: Non-Disruptive Launch Testing ]
Launch test instances in isolated staging VPC/VCN (Zero disruption to live on-prem servers)
Smoke tests: Validate OS boot, network routing, database connection strings, application health
Test instances terminated and cleaned up
                           |
[ T-0: Cutover Window (Saturday 22:00 - Sunday 02:00) ]
22:00 - Quiesce on-prem services (Stop Apache / IIS / background crons)
22:15 - Allow final storage write flush (Replication lag reaches ZERO seconds)
22:30 - Disconnect replication; Launch production target instances in target cloud
23:00 - Verify database integrity; run smoke test suite
23:30 - Shift DNS records (Route 53 / OCI DNS) to point to Cloud Load Balancers
00:15 - Validate end-to-end user transactions
01:00 - Open production traffic to the world; On-prem servers decommissioned to cold standby
```

---

## 5. Side-by-Side Architectural Comparison: AWS vs. OCI

| Feature / Dimension | AWS Application Migration Service (MGN) | OCI Cloud Migration Service |
| :--- | :--- | :--- |
| **Primary Architecture** | Agent-based kernel block replication | **Agentless** (via vSphere Appliance) or Agent-based |
| **Source Platform Support** | Physical bare-metal, VMware, Hyper-V, Azure, GCP, OCI | VMware vSphere vCenter, Physical servers |
| **Replication Transport** | TCP port 1500 (TLS encrypted) to staging EC2 | Encrypted SSL over OCI FastConnect or IPSec VPN |
| **Automated Driver Injection**| ENA, AWS NVMe, PV drivers injected at launch | VirtIO drivers, OCI OS initialization agents |
| **Shape Rightsizing** | Manual template mapping or Compute Optimizer | Native automatic recommendation to **Flexible Compute Shapes** |
| **Pre-Cutover Testing** | Non-disruptive test launch mode into test VPC | Built-in migration plan testing mode |
| **Post-Launch Optimization** | AWS Systems Manager automation scripts | OCI Cloud-Init automation scripts |
| **Service Cost** | Free replication for 90 days per server [Doc: AWS MGN] | Free migration service (Pay only for storage/compute used) |

---

## 6. Deep Implementation & Configuration (Terraform / CLI / SDK)

### AWS: MGN Launch Template Configuration (Terraform)

```hcl
# AWS Application Migration Service Launch Template
# Governs how target EC2 instances are provisioned during cutover
resource "aws_launch_template" "mgn_production_template" {
  name_prefix   = "mgn-prod-cutover-"
  image_id      = "ami-0abcdef1234567890" # Base placeholder; MGN overrides with replicated volume
  instance_type = "c6i.xlarge"

  network_interfaces {
    associate_public_ip_address = false
    subnet_id                   = var.target_private_subnet_id
    security_groups             = [var.production_app_sg_id]
  }

  iam_instance_profile {
    name = aws_iam_instance_profile.ssm_managed_profile.name
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Environment = "Production"
      MigratedBy  = "AWS-MGN"
      MigrationWave = "Wave-03"
    }
  }

  user_data = base64encode(<<-EOF
              #!/bin/bash
              echo "Executing post-migration driver validation..."
              systemctl enable amazon-ssm-agent
              systemctl start amazon-ssm-agent
              EOF
  )
}
```

---

### OCI: Cloud Migration Asset Source & Migration Plan (Terraform)

```hcl
# OCI Cloud Migration Discovery Asset Source (VMware vCenter)
resource "oci_cloud_migrations_migration_asset" "vmware_app_server" {
  migration_id = oci_cloud_migrations_migration.datacenter_migration.id
  display_name = "app-srv-01-vmware"
  inventory_asset_id = var.discovered_inventory_asset_id

  # Map directly to OCI Flexible Compute Shape
  snap_shot_bucket_name = oci_objectstorage_bucket.migration_staging_bucket.name

  depends_on = [oci_cloud_migrations_migration.datacenter_migration]
}

# OCI Cloud Migration Plan with Target Compartment Placement
resource "oci_cloud_migrations_migration_plan" "wave_one_plan" {
  compartment_id = var.compartment_ocid
  migration_id   = oci_cloud_migrations_migration.datacenter_migration.id
  display_name   = "production-wave-01-plan"

  # Target Environment Specification
  target_environments {
    target_environment_type = "VM_TARGET_ENV"
    compartment_id          = var.production_compartment_ocid
    vcn_id                  = oci_core_vcn.production_vcn.id
    subnet_id               = oci_core_subnet.app_subnet.id
    availability_domain     = "UItM:US-ASHBURN-AD-1"
    fault_domain            = "FAULT-DOMAIN-1"
  }

  # Strategy: Cost-Driven Optimization using Flex Shapes
  strategies {
    strategy_type = "AS_IS" # Preserves existing CPU/RAM allocation in OCI Flex
    metric_time_window = "1d"
  }
}
```

---

### Automated Source Agent Installation Script (Linux Bash)

```bash
#!/usr/bin/env bash
set -euo pipefail

# Enterprise automated installer for AWS Application Migration Service Agent
AWS_REGION="us-east-1"
INSTALLATION_TOKEN="AQICAHh...truncated...token"

echo "=== Commencing AWS MGN Replication Agent Deployment ==="

# Verify root privileges
if [[ $EUID -ne 0 ]]; then
   echo "Error: This script must be run as root" 1>&2
   exit 1
fi

# Download official AWS MGN installer
echo "Downloading AWS MGN installer from regional endpoint..."
wget -q -O ./aws-replication-installer-init.py \
    "https://aws-application-migration-service-${AWS_REGION}.s3.amazonaws.com/latest/linux/aws-replication-installer-init.py"

# Execute installation with non-interactive flags
echo "Executing agent installation and initiating block-level sync..."
python3 ./aws-replication-installer-init.py \
    --region "${AWS_REGION}" \
    --installer-token "${INSTALLATION_TOKEN}" \
    --no-prompt

echo "=== AWS MGN Agent successfully installed. CBT driver active. ==="
```

---

## 7. Failure Modes, Edge Cases & Blast Radius

| Failure Mode | Trigger Mechanism | Blast Radius | Mitigation Strategy |
| :--- | :--- | :--- | :--- |
| **Kernel Driver Incompatibility** | Source Linux kernel version is unsupported by the MGN/OCI replication agent | Agent crashes or triggers kernel panic on the source server | Validate kernel against official compatibility matrices; update kernel or switch to agentless vSphere replication. |
| **Replication Bandwidth Starvation** | Initial sync of 50 TB consumes 100% of company wide-area network (WAN) bandwidth | On-premises business applications experience network timeouts | Enforce bandwidth throttling limits in AWS MGN / OCI Migration settings; schedule bulk syncs during off-peak hours. |
| **Split-Brain Cutover Execution** | On-premises server is not cleanly powered down; DNS cutover occurs but background crons still write to on-prem DB | Data divergence: Cloud DB receives new orders; on-prem DB receives inventory updates | Enforce strict network isolation: revoke default gateway on on-prem VM before launching production cloud instance. |
| **Missing Cloud Hypervisor Drivers** | Source Windows VM lacks AWS ENA or OCI VirtIO network drivers | Cloud instance boots but has no network connectivity (`No IP assigned`) | Pre-install cloud driver packages on source machine prior to cutover; verify in test launch mode. |
| **Hidden Hardcoded IP Dependencies** | Legacy client software communicates via hardcoded IP `192.168.1.50` instead of DNS | Migrated cloud server unreachable by legacy on-prem client terminals | Implement AWS Overlay IP routing via Transit Gateway or configure OCI secondary private IPs matching the legacy subnet. |

---

## 8. Security, Compliance & Threat Modeling

### Migration Security Architecture & Data Protection

```
[ Source Data Center ]                                  [ Cloud Target Environment ]
+----------------------+                                +--------------------------+
| On-Prem VM Disk      |                                | Staging Storage          |
| Unencrypted on Disk  | --- TLS 1.3 / AES-256 -------->| EBS / Block Volumes      |
+----------------------+     Transit Encryption         | Encrypted via KMS CMK    |
                                                        +--------------------------+
```

1. **In-Transit Encryption**:
   * All replication streams must be encrypted using **TLS 1.3**. AWS MGN encapsulates all replication blocks in TLS frames routed over TCP port 1500. OCI Cloud Migration streams blocks over encrypted FastConnect MACsec or IPsec VPN tunnels.
2. **At-Rest Storage Encryption**:
   * Staging EBS volumes and OCI Block Volumes must be encrypted with Customer Managed Keys (CMKs) held in AWS KMS or OCI Vault.
3. **Least Privilege Migration IAM Roles**:
   * Avoid granting broad Administrator permissions to migration agents. Restrict installation tokens to temporary STS credentials with tight boundaries (`mgn:SendAgentMetrics`, `mgn:SendAgentLogs`, `mgn:GetAgentInstallationAssetsForMgn`).

---

## 9. Performance Tuning & Latency Engineering

### Accelerating Block Replication Throughput

1. **Replication Compression Tuning**:
   * Enable on-the-fly network compression on source replication agents:
     ```bash
     # AWS MGN Replication Configuration
     aws mgn update-replication-configuration \
         --source-server-id s-1234567890abcdef0 \
         --use-dedicated-replication-server \
         --ebs-encryption "CUSTOM" \
         --kms-key-arn "arn:aws:kms:..."
     ```
   * Compressing storage blocks on source hosts reduces WAN bandwidth consumption by up to **60%**, accelerating initial sync times.

2. **Replication Server Sizing**:
   * Default AWS MGN replication servers run on `t3.small` instances. For heavy write I/O databases generating $> 100 \text{ MB/sec}$ of dirty blocks, upgrade replication servers to compute-optimized or network-optimized instances (e.g., `c6i.xlarge`) to avoid TCP socket buffer exhaustion.

---

## 10. Observability, Telemetry & SRE Metrics

### Critical Migration Health Signals

```
[ Source Server ] ---> [ Replication Lag: 1.2s ] ---> [ Staging Storage ]
                       [ Daily Data Drift: 45 GB ]
                       [ ETA to Synchronized: 0m ]
```

| Metric Name | Source | Description | SRE Alert Threshold |
| :--- | :--- | :--- | :--- |
| `ReplicationLagDuration` | AWS MGN / CloudWatch | Age of un-replicated data in seconds | > 60 seconds during cutover window |
| `DataReplicationState` | OCI Cloud Migration | State of block volume sync (DISCONNECTED/REPLICATING) | State == `DISCONNECTED` |
| `NetworkEgressUtilization` | On-Prem Router / SNMP | Bandwidth consumed by migration appliance | > 85% of total WAN pipe capacity |
| `LaunchTestStatus` | AWS MGN API | Health status of test instance launch execution | Status == `FAILED` |

---

## 11. Cost Modeling & Capacity Planning

### Comprehensive Migration Cost Analysis (100 Enterprise VMs)

| Cost Component | On-Premises Baseline (Monthly) | Migration Phase Cost (Temporary) | Post-Migration Cloud Steady State |
| :--- | :--- | :--- | :--- |
| **Server Hardware / Hosting** | $25,000 (Data center rack/power) | $25,000 (Still active) | $0 (Decommissioned) |
| **Replication Infrastructure**| $0 | $1,200 (Staging EC2/EBS / OCI Disks) | $0 (Terminated post-cutover) |
| **Network Egress Bandwidth** | $0 | $1,500 (Bulk initial sync transfer) | $400 (Standard cloud egress) |
| **Target Cloud Compute/Storage**| $0 | $0 (Until cutover) | $16,500 (Right-sized instances) |
| **Net Operational Spend** | **$25,000 / month** | **$27,700 / month (Bubble)** | **$16,900 / month (-32% TCO)** |

*Staff Insight: The "Migration Bubble"*:
During the 3–6 month migration execution, organizations experience a temporary financial "bubble" where they pay for on-premises hosting and cloud staging infrastructure simultaneously. Proper wave planning minimizes this bubble duration.

---

## 12. Operational Runbooks & Triage Playbooks

### Runbook: Emergency Production Cutover Rollback

```
[ Production Cutover Declared FAILED: High Latency / Critical Bugs ]
                                |
                                v
               Step 1: Declare Migration Rollback
          (Incident Commander authorizes 15-minute rollback protocol)
                                |
                                v
               Step 2: Revert Global DNS Records
          Re-point Route 53 / OCI DNS to On-Premises VIP:
          aws route53 change-resource-record-sets --hosted-zone-id ...
                                |
                                v
               Step 3: Re-enable On-Premises Network Ingress
          Re-enable default gateway and unquiesce on-prem services
                                |
                                v
               Step 4: Terminate Cloud Target Instances
          Stop newly launched cloud compute instances to prevent rogue writes
                                |
                                v
               Step 5: Resume Replication Stream
          Re-attach MGN / OCI replication agents to resume delta sync for next attempt
```

---

## 13. Edge Cases, Quirks & Gotchas

### Subtle Migration Traps

1. **Storage Sector Misalignment (512e vs 4Kn)**:
   * Modern enterprise SAN storage uses 4 KB native (4Kn) sectors, whereas standard AWS EBS volumes default to 512-byte emulation (512e). Direct sector copies from 4Kn sources can cause bootloader corruption (`GRUB error: unknown filesystem`). Ensure storage conversion flags are active in MGN launch templates.
2. **Static MAC Address Licensing**:
   * Legacy enterprise software (e.g., Siemens, FlexLM, SAP licenses) is frequently locked to the physical MAC address of the on-premises network card. When migrated to AWS/OCI, the VM receives a new virtual NIC and new MAC address, causing the software license to immediately invalidate.
   * *Mitigation*: Assign a static custom MAC address to the target cloud network interface (ENI / VNIC) via cloud CLI prior to instance boot.

---

## 14. Real-World Case Study / Postmortem

### Enterprise Outage Postmortem: The 36-Hour ERP Cutover Stall

* **Context**: Fortune 500 manufacturing company migrating 250 enterprise Linux servers from a Chicago data center to AWS `us-east-1`.
* **The Incident**: During weekend cutover, the team shut down on-prem servers and launched 250 EC2 instances via AWS MGN.
* **The Failure**:
  1. The staging replication had performed flawlessly for 3 weeks.
  2. However, the engineering team had never executed a **non-disruptive test launch** on the core ERP database server.
  3. When the database instance booted on AWS, it failed with kernel panic: the on-premises Red Hat kernel had been compiled with proprietary local SAN multi-path drivers that hung indefinitely when attempting to find the legacy SAN hardware.
  4. The team spent 14 hours attempting manual kernel rescues in single-user mode before exceeding their 24-hour maintenance window.
* **The Rollback**: The cutover was aborted; DNS was reverted to on-premises servers.
* **The Remediation**: The SRE team mandated that no server is eligible for production cutover without passing an automated test launch in an isolated staging VPC at least 7 days prior to cutover.

---

## 15. Architectural Trade-Off Analysis

| Strategy | Migration Velocity | Technical Debt Removed | Cloud Cost Optimization | Risk Level |
| :--- | :--- | :--- | :--- | :--- |
| **Rehost (Lift & Shift)** | **Fastest** (Days/Weeks) | Zero (Bugs ported to cloud) | Minimal (Runs identical sizes) | Lowest (Known baseline) |
| **Replatform (PaaS)** | Moderate (Weeks/Months)| Moderate (DB managed by cloud) | High (Auto-scaling, managed PaaS)| Moderate |
| **Refactor (Cloud-Native)**| Slowest (Months/Years) | **Maximum** (Full modernization)| **Maximum** (Pay-per-use Serverless) | High (Requires code rewrites) |

---

## 16. Cloud Migration & Dual-Cloud Considerations

### Cloud-to-Cloud Migration: AWS to OCI

Enterprises frequently execute secondary migrations—migrating compute and database workloads from AWS to OCI to take advantage of OCI's high-performance bare metal, flexible shapes, and Oracle Database licensing economies:

```
[ AWS Source Tenancy ]                                  [ OCI Target Tenancy ]
- AWS EC2 Instance                                      - OCI Compute (VM.Standard3.Flex)
- Export EBS Volume to RAW / QCOW2                       - Import Custom Image
          |                                                      ^
          +--- Encrypted Megaport Cloud Router / FastConnect ----+
```

* **Image Export & Conversion**: AWS EC2 instances can be exported via `aws ec2 create-instance-export-task` into Amazon S3 as raw VMDK or QCOW2 images, streamed across private interconnects, and imported into OCI Custom Images with VirtIO driver enablement.

---

## 17. Automated Verification & Testing

### Migration Readiness Audit Script (Python / Boto3)

```python
import boto3
import sys

def audit_mgn_migration_readiness():
    """
    Scans all source servers in AWS Application Migration Service.
    Verifies that replication lag is under 5 seconds and test launches succeeded
    before authorizing a production cutover window.
    """
    mgn = boto3.client('mgn', region_name='us-east-1')
    servers = mgn.describe_source_servers()['items']

    print(f"Auditing {len(servers)} migration source servers...")
    ready_count = 0

    for s in servers:
        server_id = s['sourceServerID']
        data_state = s['dataReplicationInfo']['dataReplicationState']
        lag_duration = s['dataReplicationInfo']['dataReplicationLagDuration']
        test_status = s['lifeCycle']['state']

        print(f"\nServer: {server_id} | State: {data_state} | Lag: {lag_duration} | Lifecycle: {test_status}")

        if data_state != "CONTINUOUS":
            print(f"FAILED: Server {server_id} is not in CONTINUOUS replication mode.")
            continue

        if test_status not in ["TEST_LAUNCH_SUCCEEDED", "READY_FOR_CUTOVER"]:
            print(f"FAILED: Server {server_id} has not passed test launch validation.")
            continue

        ready_count += 1

    print(f"\nAudit Summary: {ready_count}/{len(servers)} servers verified ready for cutover.")
    if ready_count != len(servers):
        sys.exit(1)
    print("SUCCESS: All servers approved for production migration wave.")

if __name__ == "__main__":
    audit_mgn_migration_readiness()
```

---

## 18. Staff+ Engineering Wisdom & Insights

### Enterprise Migration Truths

1. **Discovery is 80% of the Battle**: You cannot migrate what you do not know exists. The vast majority of migration failures are caused not by cloud replication bugs, but by forgotten dependencies: an ancient batch script running on a server under an engineer's desk that copies files via FTP every midnight. Run automated discovery tools for at least **30 days** to capture monthly financial close and quarterly batch cycles.
2. **Lift and Shift is a Phase, Not a Destination**: Rehosting to the cloud without a follow-up replatforming roadmap simply results in running expensive, oversized VMs in someone else's data center. Every Rehost project must have an approved Phase 2 modernization charter.
3. **Always Have a Two-Way Door**: Never execute a cutover that cannot be rolled back within 30 minutes. If the business stakes require zero data loss, deploy reverse replication before declaring victory.

---

## 19. Interview Defense & Technical Discussion Scenarios

### Scenario 1: Rehost vs. Refactor Strategy Defense

* **Interviewer**: "We have 500 legacy applications in a leased data center that expires in 6 months. Should we refactor them to Kubernetes and serverless before migrating?"
* **Staff Candidate Response**:
  1. *Immediate Reality Check*: Refactoring 500 applications in 6 months is an architectural impossibility. It guarantees missed data center evacuation deadlines and multi-million dollar lease penalty extensions.
  2. *Two-Stage Strategy*:
     * **Stage 1: Rehost (Lift & Shift)** using **AWS Application Migration Service (MGN)** or **OCI Cloud Migration**. Continuous block replication allows moving 500 VMs with zero code changes within 3 to 4 months, successfully beating the data center lease termination.
     * **Stage 2: Targeted Replatforming & Refactoring**. Once stabilized in the cloud, prioritize applications by business value. Refactor core revenue-generating applications to cloud-native microservices while keeping low-touch legacy workloads as right-sized VMs.

### Scenario 2: Handling Network Latency in Hybrid Migration Waves

* **Interviewer**: "During a 6-month migration, the database has been moved to AWS, but the legacy reporting application must remain on-premises for another 3 months. How do you mitigate the latency impact?"
* **Staff Candidate Response**:
  1. *Dedicated Private Interconnect*: Provision **AWS Direct Connect** or **OCI FastConnect** with sub-5ms deterministic latency and jumbo frames (9000 MTU) to prevent WAN jitter.
  2. *Local Read Caching*: Deploy a read replica or local Redis cache on-premises to service high-frequency read queries locally, shielding the cross-premises network link.
  3. *Re-evaluate Wave Grouping*: High-affinity application-database pairs should **never be separated across the WAN**. Re-group the reporting tool into the same migration wave as the database to migrate them concurrently in a single cutover window.

---

## 20. Summary Cheat Sheet & Reference Matrix

```
+---------------------------------------------------------------------------------------------------+
|                            CLOUD MIGRATION STRATEGIES CHEAT SHEET                                 |
+--------------------------+------------------------------------+-----------------------------------+
| Dimension                | AWS Migration Stack                | OCI Migration Stack               |
+--------------------------+------------------------------------+-----------------------------------+
| Portfolio Discovery      | Application Discovery Service      | OCI Cloud Migration Discovery     |
| Centralized Tracking     | AWS Migration Hub                  | OCI Migration Plans Dashboard     |
| Server Migration Engine  | Application Migration Service (MGN)| OCI Cloud Migration Appliance     |
| Replication Level        | Block-level asynchronous (Agent)   | Block-level agentless (vSphere)   |
| Staging Cost             | Lightweight EC2 + Staging EBS      | Staging Block Volumes             |
| Driver Injection         | Automatic ENA/NVMe hypervisor      | Automatic VirtIO drivers          |
| Shape Recommendation     | AWS Compute Optimizer              | Native OCI Flexible Shapes        |
| Max Free Replication     | 90 days per server [Doc: AWS MGN]  | Free service (Pay used storage)   |
+--------------------------+------------------------------------+-----------------------------------+
```
