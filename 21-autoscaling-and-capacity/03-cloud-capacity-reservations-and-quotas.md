# Cloud Capacity Reservations, Zonal Exhaustion & Service Quotas (AWS vs. OCI)

---

## 1. Executive Summary & Core Definitions

In hyperscale multi-tenant public clouds, physical infrastructure is finite. During major natural disasters, fiber cuts, or regional zone outages, thousands of tenants simultaneously trigger disaster recovery failovers and autoscaling expansion into surviving Availability Zones (AZs) or Availability Domains (ADs). Without pre-allocated compute reservations, workloads encounter **zonal capacity exhaustion** (`InsufficientInstanceCapacity` in AWS, `Out of host capacity` in OCI).

Capacity reservations guarantee that physical server capacity (CPU, memory, GPU, and localized hypervisor slots) is physically committed and held exclusively for your tenancy in a designated failure domain, completely eliminating the risk of launch failures during high-demand events.

```
       WITHOUT RESERVATIONS (On-Demand Spot/Autoscaling)
[ Regional Incident ] ---> 10,000 Nodes Fail Over ---> Surviving AZ
                                                            |
                                               [ Out of Host Capacity! ]
                                               Workload Remains Offline (Downtime)

       WITH CAPACITY RESERVATIONS
[ Regional Incident ] ---> 10,000 Nodes Fail Over ---> Surviving AZ
                                                            |
                                               [ Dedicated Hardware Slot ]
                                               Instance Launches Guaranteed (0 Downtime)
```

### Core Terminology
* **On-Demand Capacity Reservation (ODCR - AWS)**: A reservation that secures EC2 compute capacity in a specific AZ for any duration. Available immediately without long-term commitment. Incurs standard On-Demand hourly charges whether instances are running or idle [Doc: AWS EC2 ODCR, checked 2026].
* **Capacity Reservation (OCI)**: Secures compute capacity for specific compute shapes within an Availability Domain and optional Fault Domain. **Key Cost Differentiator**: OCI charges **zero fees** for reserved unlaunched capacity; billing only commences when instances are actively provisioned into the reservation [Doc: OCI Capacity Reservations, checked 2026].
* **Service Quotas / Service Limits**: Organizational soft and hard boundaries imposed by cloud providers to prevent runaway spend or API saturation (e.g., maximum vCPUs per region, maximum VPCs per tenancy).
* **Zonal Capacity Exhaustion**: The condition where a cloud provider's physical rack clusters in an AZ/AD have fully allocated all physical memory, CPU sockets, or network interfaces for a requested instance family/shape.

---

## 2. Architectural Deep Dive: AWS vs. OCI Mechanisms

### AWS On-Demand Capacity Reservations (ODCR)
AWS separates financial commitments from capacity commitments:
* **Financial Commitment**: Savings Plans and Standard Reserved Instances (RIs) provide discounts in exchange for 1- or 3-year commitments, but regional RIs do **not** reserve physical hardware.
* **Physical Guarantee**: ODCR provides physical capacity guarantees in a designated AZ.
* **Matching Criteria**: Reservations match instances based on instance type, platform (Linux/Windows), tenancy (default or dedicated), and AZ.
* **Instance Eligibility Types**:
  * `open`: Any newly launched instance matching the attributes automatically consumes the reservation.
  * `targeted`: Only instances that explicitly supply `--capacity-reservation-id` or match placement groups consume the reservation.
* **Capacity Reservation Groups & Resource Groups**: Multiple ODCRs can be aggregated into AWS Resource Groups to distribute capacity across multiple AZs and instance types.

### OCI Capacity Reservations
OCI treats capacity reservation as a first-class tenant reliability primitive:
* **Shape & Domain Specificity**: Capacity is reserved for a specific compute shape (e.g., `VM.Standard3.Flex` with exact OCPU and memory specs, or `BM.GPU4.8`) within a designated Availability Domain (AD) and Fault Domain (FD).
* **Billing Mechanics (Zero-Fee Advantage)**: Unlike AWS where idle reservations bill at 100% of On-Demand rates, OCI reserves capacity without idle holding charges. You only pay for active compute instances running within the reservation [Doc: OCI Compute Pricing, checked 2026].
* **Usage Types**:
  * Default: Instances matching the shape launched in the AD automatically consume reserved capacity.
  * Explicit / Target-Only: Instances must specifically reference the `reservation_id` in their launch specification.

---

## 3. Side-by-Side Comparison

| Feature / Dimension | AWS (On-Demand Capacity Reservations) | OCI (Capacity Reservations) |
| :--- | :--- | :--- |
| **Reservation Scope** | Specific Availability Zone (AZ) | Specific Availability Domain (AD) & optional Fault Domain |
| **Idle Billing Cost** | **Billed at full On-Demand rate** while unused [Doc: AWS EC2 Pricing] | **Free / Zero fee** while unused (Pay only when launched) [Doc: OCI Compute] |
| **Financial Discount Coupling** | Disjoint; can be covered by Regional Savings Plans or RIs | Coupled with Universal Credits commitments |
| **Shape Flexibility** | Fixed instance type (e.g., `m6i.2xlarge`) | Supports fixed shapes and Flexible shapes (OCPU + RAM combos) |
| **Autoscaling Group Integration** | Native (`CapacityReservationTarget` in ASG launch templates) | Native (`InstancePool` placement configuration) |
| **Cluster Placement Groups** | Supported with ODCR for low-latency HPC/ML workloads | Supported with OCI HPC bare metal and cluster networks |
| **Cross-Account Sharing** | Supported via AWS Resource Access Manager (RAM) | Compartment inheritance and cross-tenancy policies |
| **Service Quota Enforcement** | AWS Service Quotas (Self-service console & API) | OCI Service Limits & Compartment Quota Policies |

---

## 4. Implementation & Configuration (Terraform / CLI)

### AWS: On-Demand Capacity Reservation (Terraform)

```hcl
# AWS EC2 Targeted Capacity Reservation in us-east-1a
resource "aws_ec2_capacity_reservation" "production_db_standby" {
  instance_type           = "r6i.4xlarge"
  instance_platform       = "Linux/UNIX"
  availability_zone       = "us-east-1a"
  instance_count          = 4
  instance_match_criteria = "targeted" # Requires explicit reservation ID reference
  tenancy                 = "default"

  # Can be cancelled at any time or set with an end date
  end_date_type           = "unlimited"

  tags = {
    Environment = "Production"
    Role        = "DisasterRecoveryGuarantee"
  }
}

# EC2 Instance consuming the targeted capacity reservation
resource "aws_instance" "dr_node" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "r6i.4xlarge"
  subnet_id     = "subnet-0123456789abcdef0" # Subnet residing in us-east-1a

  capacity_reservation_specification {
    capacity_reservation_preference = "capacity-reservations-only"
    capacity_reservation_target {
      capacity_reservation_id = aws_ec2_capacity_reservation.production_db_standby.id
    }
  }

  tags = {
    Name = "prod-dr-node-01"
  }
}
```

---

### OCI: Capacity Reservation (Terraform)

```hcl
# OCI Capacity Reservation in Availability Domain 1
resource "oci_core_compute_capacity_reservation" "dr_compute_reserve" {
  compartment_id = var.compartment_ocid
  display_name   = "dr-failover-capacity-reserve"
  availability_domain = "UItM:US-ASHBURN-AD-1"
  is_default_reservation = false

  instance_reservation_configs {
    instance_shape = "VM.Standard3.Flex"

    instance_shape_config {
      ocpus         = 8
      memory_in_gbs = 64
    }

    reserved_count = 10 # 10 instances reserved; 0 cost while idle
    fault_domain   = "FAULT-DOMAIN-1"
  }
}

# OCI Instance Pool targeting the Capacity Reservation
resource "oci_core_instance_pool" "app_pool" {
  compartment_id        = var.compartment_ocid
  instance_configuration_id = oci_core_instance_configuration.app_config.id
  size                  = 0 # Scaled to 0 in secondary region until DR failover

  placement_configurations {
    availability_domain    = "UItM:US-ASHBURN-AD-1"
    primary_vnic_subnets {
      subnet_id = oci_core_subnet.app_subnet.id
    }
    # Direct failover instances into guaranteed capacity
    capacity_reservation_id = oci_core_compute_capacity_reservation.dr_compute_reserve.id
  }
}
```

---

## 5. Failure Modes, Edge Cases & Zonal Exhaustion

```
[ Failure Timeline: Regional Az Outage ]
T = 00:00   AZ-a suffers catastrophic fiber cut / substation trip.
T = 00:02   Global DNS (Route 53 / OCI DNS) steers 100% traffic to AZ-b.
T = 00:03   Autoscaling Groups in AZ-b request +500 instances simultaneously.
T = 00:04   AWS returns: "InsufficientInstanceCapacity" (Surviving AZ is drained).
T = 00:05   Unreserved workloads fail. Workloads with ODCR launch successfully.
```

### Critical Edge Cases
1. **The "Surviving Zone" Runaway Stampede**:
   * When an entire cloud data center fails, thousands of enterprise tenants try to launch nodes in the remaining zones. The cloud provider's available hardware buffer evaporates within 90–180 seconds.
   * *Mitigation*: Reserve failover capacity in advance using ODCR / OCI Capacity Reservations, or architect multi-region active-active clusters.
2. **Quota Throttling vs. Physical Exhaustion**:
   * *Physical Exhaustion*: Cloud provider has zero physical servers left (Error 500 / `InsufficientInstanceCapacity`). Cannot be resolved by cloud support immediately.
   * *Service Quota Breach*: Account policy ceiling hit (e.g., `ClientError: RequestLimitExceeded` or `VcpuLimitExceeded`). Preventable via proactive Service Quota alerts.
3. **Capacity Reservation Drift**:
   * Application teams upgrade base instance shapes from `m5.xlarge` to `m6i.xlarge`, but fail to update the capacity reservation. During failover, the system requests `m6i` instances, ignoring the idle `m5` reservations and failing due to lack of `m6i` capacity.

---

## 6. Real-World Case Study / Incident Scenario

* **Company**: Tier-1 FinTech Payment Clearinghouse.
* **Setup**: Primary payment API running in AWS `us-east-1` across 3 AZs. Cold standby replica in `us-east-2`.
* **The Incident**: A massive lightning storm knocked out utility power to a major AWS data center in `us-east-1a`. Autoscaling policies triggered a massive scale-up in `us-east-1b` and `us-east-1c`.
* **The Consequence**: Because other large enterprises (Netflix, banks, SaaS providers) were also autoscaling into `us-east-1b` and `us-east-1c`, AWS ran completely out of `c5.2xlarge` and `r5.4xlarge` capacity. The payment API could not scale to handle redirected traffic, causing transaction drops for 47 minutes.
* **The Fix**:
  * Implemented AWS On-Demand Capacity Reservations for minimum viable payment processing capacity (200 instances) in `us-east-1b` and `us-east-1c`.
  * Configured OCI secondary warm standby with OCI Capacity Reservations—leveraging OCI's zero-cost idle reservation model to hold 150 bare-metal database and compute shapes at zero idle cost.

---

## 7. Interview Defense & Technical Trade-Offs

### Scenario: Defending DR Budget vs. Capacity Guarantees

* **Interviewer**: "Our executive team wants zero downtime during an AZ outage, but CFO refuses to pay double for idle compute in surviving zones. How do you design this?"
* **Staff Candidate Response**:
  1. *AWS Strategy with Mixed ODCR*: Do not reserve 100% peak capacity. Reserve only **Minimum Viable Scale (MVS)** (e.g., 30–40% capacity) with targeted ODCR. Cover these ODCRs with Regional Compute Savings Plans to obtain 30–40% financial discounts on the reservation holding fees.
  2. *Instance Flexibility*: Configure EC2 Auto Scaling Launch Templates to use **Instance Attribute-Based Selection** (e.g., allow `m6i`, `m5`, `c6i`, `c5` simultaneously across surviving AZs) to avoid pinning failover to a single hardware cluster.
  3. *Multi-Cloud Leverage (OCI Strategy)*: If multi-cloud is an option, place the DR standby cluster in OCI using **OCI Capacity Reservations**. Because OCI does not charge holding fees for unallocated reserved capacity, we guarantee physical capacity in Ashburn or Phoenix with $0/month compute idle spend until a failover is declared.
