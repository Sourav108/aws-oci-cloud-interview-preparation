# 02. OCI Regions, Availability Domains & Fault Domains

## 1. Problem
When cloud engineers design systems on Oracle Cloud Infrastructure (OCI) using assumptions imported directly from AWS, they frequently misarchitect their network and compute topology. Engineers often assume that every OCI region has 3 physically isolated Availability Domains (like AWS has at least 3 AZs), or conversely, they assume that a single-AD region lacks hardware fault isolation and treat it as a single point of failure. Understanding OCI's **Realm**, **Region**, **Availability Domain**, and **Fault Domain** hierarchy is essential for designing resilient, high-throughput enterprise systems.

## 2. Cloud Concept
OCI structures its physical and logical infrastructure into four distinct containment tiers:

1. **Realm**: A completely independent, isolated collection of OCI regions that share zero identity, zero metadata, and zero infrastructure with other realms.
   - `OC1`: Commercial / Global Realm.
   - `OC2`, `OC3`: US Government and Defense Realms.
   - `OC4`: UK Dedicated and Sovereign Cloud Realms.
2. **Region**: A localized geographic area within a realm (e.g., `us-ashburn-1`, `eu-frankfurt-1`, `ap-tokyo-1`).
3. **Availability Domain (AD)**: One or more data centers located within a region. Availability Domains are physically isolated from one another, with independent power and cooling infrastructure, connected by low-latency optical fiber.
   - *Multi-AD Regions*: Regions containing 3 physically separate Availability Domains (e.g., Ashburn, Phoenix, Frankfurt, London) `[Doc: OCI Data Regions, checked 2026-09-03]`.
   - *Single-AD Regions*: Cost-optimized regions containing 1 Availability Domain (e.g., Zurich, Madrid, Melbourne, Hyderabad).
4. **Fault Domain (FD)**: Inside **every** Availability Domain, OCI engineers exactly **3 Fault Domains** (`FAULT-DOMAIN-1`, `FAULT-DOMAIN-2`, `FAULT-DOMAIN-3`).
   - A Fault Domain represents a distinct physical hardware grouping: separate server racks, redundant power distribution units (PDUs), and independent top-of-rack (ToR) switches within the data center hall `[Doc: OCI Fault Domains, checked 2026-09-03]`.
   - Hardware maintenance, hypervisor patching, and physical rack maintenance are conducted on one Fault Domain at a time, ensuring that resources distributed across different FDs never undergo concurrent maintenance downtime.

## 3. Mental Model
Think of OCI infrastructure as a secure corporate campus:
- **The Realm** is the country's sovereign legal jurisdiction. Data and identities in Realm A cannot be seen or queried from Realm B.
- **The Region** is a corporate office campus in a city.
- **The Availability Domain** is a separate physical office building on that campus.
- **The Fault Domain** is a separate, fire-walled wing within each office building, powered by its own electrical circuit breaker and network switchboard. Even if an office building has only one main lobby (single-AD region), a power surge in Wing 1 cannot take down computers in Wing 2 or Wing 3.

## 4. Architecture Diagram
```text
┌────────────────────────────────────────────────────────────────────────┐
│                   OCI COMMERCIAL REALM (OC1)                           │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    OCI REGION: us-ashburn-1                      │  │
│  │                                                                  │  │
│  │  ┌───────────────────────┐             ┌───────────────────────┐ │  │
│  │  │ AVAILABILITY DOMAIN 1 │             │ AVAILABILITY DOMAIN 2 │ │  │
│  │  │                       │             │                       │ │  │
│  │  │ ┌───────────────────┐ │  Low-       │ ┌───────────────────┐ │  │
│  │  │ │  FAULT DOMAIN 1   │ │  Latency    │ │  FAULT DOMAIN 1   │ │  │
│  │  │ │ (Rack A / PDU 1)  │ │  Optical    │ │ (Rack D / PDU 4)  │ │  │
│  │  │ ├───────────────────┤ │  Inter-     │ ├───────────────────┤ │  │
│  │  │ │  FAULT DOMAIN 2   │ │  connect    │ │  FAULT DOMAIN 2   │ │  │
│  │  │ │ (Rack B / PDU 2)  │ │◄───────────►│ │ (Rack E / PDU 5)  │ │  │
│  │  │ ├───────────────────┤ │             │ ├───────────────────┤ │  │
│  │  │ │  FAULT DOMAIN 3   │ │             │ │  FAULT DOMAIN 3   │ │  │
│  │  │ │ (Rack C / PDU 3)  │ │             │ │ (Rack F / PDU 6)  │ │  │
│  │  │ └───────────────────┘ │             │ └───────────────────┘ │  │
│  │  └───────────────────────┘             └───────────────────────┘ │  │
│  │                                                                  │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │
│  │  │ REGIONAL SUBNET (10.0.1.0/24) — SPANS ALL ADs & FDs        │  │  │
│  │  └────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS:
- AWS requires every region to have a minimum of 3 Availability Zones `[Doc: AWS Global Infrastructure, checked 2026-09-03]`.
- AWS exposes no native equivalent to "Fault Domains" inside a single AZ for standard EC2 instances. To guarantee rack-level anti-affinity in AWS, engineers must explicitly provision a **Spread Placement Group** (which caps instances at 7 per AZ) or a **Partition Placement Group**.
- Subnets in AWS are strictly **Zonal**. A subnet cannot span across multiple AZs. Creating a multi-AZ application requires provisioning at least 3 distinct subnets with non-overlapping CIDRs and managing route tables across all 3 subnets.

## 6. OCI Implementation
In OCI:
- **Native Fault Domain Allocation**: When creating any compute instance or database node in OCI, you can explicitly select the target Fault Domain (`FAULT-DOMAIN-1`, `FAULT-DOMAIN-2`, `FAULT-DOMAIN-3`), or allow OCI's scheduler to balance instances across FDs automatically `[Doc: OCI Compute Launch, checked 2026-09-03]`.
- **Anti-Affinity Without Placement Groups**: High availability across racks within a data center requires zero extra configuration fees or instance limits; it is a first-class citizen of every compute launch request.
- **Regional Subnets by Default**: OCI Virtual Cloud Networks (VCNs) support **Regional Subnets**. A single subnet (e.g., `10.0.1.0/24`) automatically extends across all Availability Domains and all Fault Domains in the region.
  - Instances deployed in AD-1 and instances deployed in AD-2 can share the exact same subnet, default route table, and security lists.
  - This eliminates the subnet-proliferation complexity found in AWS multi-AZ architectures.
- **Off-Box Network Virtualization**: OCI implements off-box virtualization using customized SmartNICs. The virtualization overhead is removed from the host CPU/memory and placed on the network card, meaning network throughput and latency between Fault Domains within an AD is virtually identical to bare-metal performance (< 0.1ms).

## 7. Configuration
Comparing how compute instances are pinned to physical failure domains in Terraform:

### OCI Compute Instance with Explicit Fault Domain Assignment (Terraform)
```hcl
# Instance 1: Pinned to Fault Domain 1 in Regional Subnet
resource "oci_core_instance" "backend_node_1" {
  compartment_id      = var.compartment_id
  availability_domain = var.availability_domain # e.g. "Uwhp:US-ASHBURN-AD-1"
  fault_domain        = "FAULT-DOMAIN-1"        # Physical rack grouping 1
  display_name        = "backend-worker-01"
  shape               = "VM.Standard.E5.Flex"

  shape_config {
    ocpus         = 2
    memory_in_gbs = 16
  }

  create_vnic_details {
    subnet_id        = oci_core_subnet.regional_private.id # Regional subnet!
    assign_public_ip = false
    nsg_ids          = [oci_core_network_security_group.backend.id]
  }
}

# Instance 2: Pinned to Fault Domain 2 for hardware anti-affinity
resource "oci_core_instance" "backend_node_2" {
  compartment_id      = var.compartment_id
  availability_domain = var.availability_domain
  fault_domain        = "FAULT-DOMAIN-2"        # Physical rack grouping 2
  display_name        = "backend-worker-02"
  shape               = "VM.Standard.E5.Flex"

  shape_config {
    ocpus         = 2
    memory_in_gbs = 16
  }

  create_vnic_details {
    subnet_id        = oci_core_subnet.regional_private.id
    assign_public_ip = false
    nsg_ids          = [oci_core_network_security_group.backend.id]
  }
}
```

### AWS EC2 Instance with Spread Placement Group (Terraform)
```hcl
# AWS requires an explicit placement group to guarantee rack-level separation
resource "aws_placement_group" "rack_spread" {
  name     = "app-rack-spread"
  strategy = "spread" # Maximum 7 instances per AZ!
}

resource "aws_instance" "backend_node_1" {
  ami                  = var.ami_id
  instance_type        = "c6i.xlarge"
  subnet_id            = aws_subnet.zonal_az1.id # Strictly zonal subnet!
  placement_group      = aws_placement_group.rack_spread.id
  vpc_security_group_ids = [aws_security_group.backend.id]
}
```

## 8. Data Flow
```text
Inbound Traffic ──> [OCI Regional Load Balancer (Active-Standby across ADs/FDs)]
                           │
            ┌──────────────┴──────────────┐
            ▼                             ▼
   [Regional Subnet: 10.0.1.0/24]   [Regional Subnet: 10.0.1.0/24]
   (Compute in FAULT-DOMAIN-1)      (Compute in FAULT-DOMAIN-2)
            │                             │
            └──────────────┬──────────────┘
                           ▼ (Sub-millisecond intra-AD / inter-FD)
   [OCI Base DB System: Primary in FD-1] ──► [Data Guard Standby in FD-2]
```

## 9. Security
- **Fault Domain Isolation**: Hardware or firmware exploits confined to a physical hypervisor or rack switch cannot compromise compute workloads running in adjacent Fault Domains.
- **Tenancy Boundary vs. Compartment Boundary**: In OCI, an entire corporation belongs to a single **Tenancy** (root compartment). Compartments are logical, not physical, and can span all regions and fault domains. Security policies applied at the root compartment cascade down to all resources regardless of physical region.

## 10. Reliability
- **Maintenance Windows Without Downtime**: OCI conducts non-emergency infrastructure updates (hypervisor patching, firmware updates, switch maintenance) one Fault Domain at a time. If an application cluster distributes replicas across all 3 Fault Domains, 66.6% of the cluster remains online and serving traffic throughout scheduled maintenance.
- **Single-AD High Availability**: Unlike AWS, where deploying in a single AZ is considered an anti-pattern that provides zero rack resilience, deploying across 3 Fault Domains in a single-AD OCI region satisfies enterprise HA standards for localized hardware and power failures.

## 11. Scaling
- **OCI Instance Pool Elasticity Across Fault Domains**: When configuring an OCI Instance Pool, you can specify target Availability Domains and define the placement algorithm:
  - OCI can automatically distribute launched instances in a round-robin fashion across `FAULT-DOMAIN-1`, `FAULT-DOMAIN-2`, and `FAULT-DOMAIN-3`.
  - This guarantees that as the fleet autoscales from 3 to 30 instances, hardware anti-affinity is maintained automatically.

## 12. Observability
- **OCI Cloud Guard & Work Requests**: Track hardware lifecycle operations and instance status.
- **Fault Domain Metric Dimensions**: In OCI Monitoring, CPU and memory metrics emit the `faultDomain` dimension, allowing SREs to visualize performance anomalies correlated with specific physical rack infrastructure.

## 13. Cost
- **Zero Inter-AD Data Transfer Fees in OCI**:
  - In AWS, traffic between two AZs in the same region is billed at \$0.01/GB in and \$0.01/GB out `[Doc: AWS EC2 Data Transfer, checked 2026-09-03]`.
  - In OCI, network traffic between Availability Domains and Fault Domains within the same region is **completely free** `[Doc: OCI Networking Pricing, checked 2026-09-03]`. This makes high-volume distributed data replication significantly cheaper in OCI.

## 14. Failure Modes
- **The Single-AD Blind Spot**: A team assumes that because they deployed across 3 Fault Domains in a single-AD region (e.g., Zurich), they are protected from catastrophic disasters. A major fire or flood engulfs the entire physical data center building. Because all 3 Fault Domains reside in the same physical building, the entire region goes dark. *Mitigation: Pair single-AD regions with cross-region replication (e.g., Zurich paired with Frankfurt).*
- **AD Hardcoding Failures**: Hardcoding an Availability Domain string like `"Uwhp:US-ASHBURN-AD-1"` in Terraform scripts across tenancies. Availability Domain naming prefixes (`Uwhp:`) vary between OCI tenancies. Scripts must dynamically discover AD names using the `data "oci_identity_availability_domains"` data source.

## 15. Troubleshooting
When an instance in an OCI Fault Domain fails to launch:
1. **Inspect OCI Work Requests**: Navigate to `Identity & Security` $\to$ `Work Requests` or run `oci work-requests work-request list`.
2. **Check Capacity Allocation**: Look for error code `Out of host capacity in fault domain FAULT-DOMAIN-X`.
3. **Remediation**: Adjust the instance launch configuration to allow OCI to select the Fault Domain automatically, or launch into an alternative Fault Domain.

## 16. Common Mistakes
- **Using AD-Specific Subnets Unnecessarily in OCI**: OCI originally supported only AD-specific subnets (legacy). Modern OCI architectures should use **Regional Subnets** exclusively, unless specialized single-AD legacy database hardware mandates AD binding.
- **Ignoring Fault Domain Balance in Clustered Software**: Deploying a 3-node Redis cluster into AD-1, but letting all 3 nodes land in `FAULT-DOMAIN-1` because no fault domain was specified. A single rack maintenance reboot takes down the entire Redis cluster.

## 17. Trade-offs
| Dimension | Multi-AD Region (e.g., Ashburn, Phoenix) | Single-AD Region (e.g., Zurich, Tokyo) |
| :--- | :--- | :--- |
| **Physical Blast Radius** | Survives loss of an entire data center building | Vulnerable to catastrophic building-level disasters (fire, flood) |
| **Inter-Node Latency** | Low across ADs (1–2ms); Ultra-low within an AD (< 0.1ms) | Consistently ultra-low (< 0.1ms) across all Fault Domains |
| **Architectural Complexity** | Must distribute across both ADs and Fault Domains | Simple: Distribute across 3 Fault Domains within the single AD |
| **Cross-Region Requirement** | Cross-region DR recommended for catastrophic scenarios | Cross-region DR **mandatory** for tier-1 production systems |

## 18. Interview Questions
1. *What is the exact physical and logical difference between an AWS Availability Zone and an OCI Fault Domain?*
2. *Can a single-AD OCI region be considered 'Highly Available'? How would you defend this architecture to a risk auditor compared to a multi-AZ AWS region?*
3. *Why does OCI offer free inter-AD data transfer while AWS charges for cross-AZ traffic? What architectural design choices does this enable?*

## 19. Interview Answer
**Exemplary Answer to Question 1 & 2**:
> "An **AWS Availability Zone** consists of one or more discrete, physically separate data center buildings located kilometers apart, with independent utility power grids and municipal water supplies.
>
> An **OCI Fault Domain**, by contrast, is a physical hardware grouping (racks, power distribution units, top-of-rack network switches) located **within** a single Availability Domain. Inside every OCI Availability Domain, there are exactly 3 Fault Domains.
>
> **Can a single-AD region be Highly Available?**
> Yes, but with an explicit definition of blast radius:
> 1. Against localized hardware failures (a server motherboard failure, a dead top-of-rack switch, an unseated power supply, or rolling hypervisor patching), deploying across 3 Fault Domains in a single-AD OCI region provides **identical high availability** to multi-AZ architectures. No single hardware failure can knock out more than 33% of our nodes.
> 2. However, against catastrophic building-level disasters (a plane crash, catastrophic building fire, or massive flood), a single-AD region lacks physical building isolation. To satisfy enterprise tier-1 disaster recovery in a single-AD region, we must pair the single-AD region with an asynchronous cross-region DR pairing (e.g., OCI Frankfurt or Ashburn) using Remote Peering Gateways and cross-region Data Guard or Object Storage replication."

## 20. Hands-on Exercise
**Objective**: Dynamically discover OCI Availability Domains and verify Fault Domain balancing in Terraform.

### Verification Steps
1. Write a Terraform block querying `oci_identity_availability_domains`.
2. Provision a count of 3 instances in a loop, assigning `fault_domain = "FAULT-DOMAIN-${count.index + 1}"`.
3. Verify via OCI CLI:
   ```bash
   oci compute instance list --compartment-id <compartment-ocid> \
     --query "data[*].[display-name, availability-domain, fault-domain]" --output table
   ```
4. Confirm that all 3 instances reside in the same regional subnet but are pinned to 3 distinct Fault Domains.
