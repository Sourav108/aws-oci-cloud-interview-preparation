# 01. VPC & VCN Architecture

## 1. Problem
In traditional data center engineering, isolating multi-tenant networks required physical switches, VLAN trunking (802.1Q), and complex Spanning Tree Protocol (STP) configurations, which were fundamentally capped at 4,094 VLAN IDs and prone to broadcast storms. In hyperscale public clouds, thousands of independent organizations run overlapping RFC 1918 private IP address spaces on the exact same underlying physical server hardware. When senior engineers fail to understand the software-defined overlay networking that powers AWS Virtual Private Clouds (VPC) and OCI Virtual Cloud Networks (VCN), they misconfigure MTUs, create unroutable cross-subnet topologies, or trigger catastrophic network interface attachment limits.

## 2. Cloud Concept
A cloud virtual network is a logically isolated, software-defined network (SDN) constructed entirely in software on top of a physical Clos spine-and-leaf data center network:

- **The Underlay Network**: The physical hardware network comprising physical racks, top-of-rack (ToR) switches, spine switches, and redundant optical fiber cables. All physical interfaces communicate using standard IPv4/IPv6 routing over Equal-Cost Multi-Path (ECMP) links.
- **The Overlay Network**: The virtual abstraction presented to the customer. Customer virtual machines attach virtual network interfaces (AWS ENI / OCI VNIC). When a VM transmits an IP packet, the cloud hypervisor captures the packet, encapsulates it inside an overlay tunnel header (Geneve in modern AWS Nitro, custom VXLAN in OCI SmartNICs), stamps it with a unique Virtual Network Identifier (VNI), and transmits it across the physical underlay.
- **Zonal vs. Regional Subnet Scopes**:
  - **AWS Amazon VPC**: The VPC is regional, but **every subnet is strictly Zonal** (bound to exactly one Availability Zone). High availability requires creating at least 3 distinct subnets, each with its own non-overlapping CIDR block.
  - **OCI Virtual Cloud Network (VCN)**: The VCN is regional, and **subnets are Regional by default**. A single OCI subnet CIDR block automatically spans all Availability Domains (ADs) and Fault Domains (FDs) in the region.

## 3. Mental Model
Think of cloud virtual networking as an international container shipping logistics network:
- The **Physical Underlay** is the open ocean and cargo shipping lanes. Cargo ships move between ports without knowing what is inside the steel shipping containers.
- The **Cloud Virtual Overlay (VPC/VCN)** is the sealed steel shipping container. Customer packets are packed inside.
- The **Virtual Network Identifier (VNI)** is the container serial number painted on the outside. Even if Company A and Company B both pack identical items addressed to `10.0.0.1` inside their respective containers, the ship's crew (the hypervisor underlay) routes the containers to completely different destinations based on the container serial number.

## 4. Architecture Diagram
```text
┌────────────────────────────────────────────────────────────────────────┐
│                        PHYSICAL UNDERLAY NETWORK                       │
│      [Spine Switch 1] ─── [Spine Switch 2] ─── [Spine Switch 3]        │
│             │                    │                    │                │
│      [Leaf Switch A]      [Leaf Switch B]      [Leaf Switch C]         │
└─────────────┬────────────────────┬────────────────────┬────────────────┘
              │                    │                    │
              ▼                    ▼                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   SOFTWARE-DEFINED OVERLAY FABRIC                      │
│                                                                        │
│   AWS: Nitro Hypervisors (Geneve Encapsulation)                        │
│   OCI: SmartNIC Hardware Off-Box Virtualization (VXLAN Encapsulation)  │
├────────────────────────────────────────────────────────────────────────┤
│   TENANT A: VPC / VCN 10.0.0.0/16       TENANT B: VPC / VCN 10.0.0.0/16│
│   (VNI: 10042)                          (VNI: 90210)                   │
│                                                                        │
│   AWS: Zonal Subnets                    AWS: Zonal Subnets             │
│   [Subnet 10.0.1.0/24 in AZ-1]          [Subnet 10.0.1.0/24 in AZ-1]   │
│   [Subnet 10.0.2.0/24 in AZ-2]          [Subnet 10.0.2.0/24 in AZ-2]   │
│                                                                        │
│   OCI: Regional Subnet                  OCI: Regional Subnet           │
│   [Subnet 10.0.10.0/24 spans all ADs]   [Subnet 10.0.10.0/24 spans all]│
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS Amazon VPC:
- **VPC Boundaries**: A VPC is constrained to a single AWS Region and a single AWS Account.
- **CIDR Blocks**: Supports up to 5 IPv4 CIDR blocks (1 primary and up to 4 secondary) `[Doc: Amazon VPC Quotas, checked 2026-09-03]`. Sizes range from `/16` (65,536 IPs) down to `/28` (16 IPs).
- **Subnet Constraints**: Every subnet must reside entirely within one AZ and cannot span zones. If an application requires 3 AZs for multi-AZ high availability, the architect must create at least 3 separate subnets and size their CIDR blocks appropriately.
- **Hyperplane Fabric**: AWS routes stateful traffic (ALB, NLB, NAT Gateway, EFS, Transit Gateway) through **Hyperplane**, a massively distributed, internal software-defined network cluster that manages connection tracking and packet rewriting at multi-terabit scale `[Doc: AWS Hyperplane Architecture, checked 2026-09-03]`.
- **Elastic Network Interfaces (ENIs)**: Virtual network interfaces that attach to EC2 instances. Each instance type has a hard hardware limit on the number of ENIs and secondary private IP addresses it can support.

## 6. OCI Implementation
In Oracle Cloud Infrastructure VCN:
- **VCN Boundaries**: A VCN belongs to a specific OCI Region and is assigned to a specific **Compartment** in the tenancy hierarchy.
- **CIDR Blocks**: Supports up to 5 non-overlapping IPv4 CIDRs per VCN, ranging from `/16` down to `/30` (4 IPs) `[Doc: OCI VCN Overview, checked 2026-09-03]`.
- **Regional Subnets by Default**: An OCI subnet is **Regional**. It spans all Availability Domains and Fault Domains in the region.
  - An instance in AD-1 and an instance in AD-2 can share the exact same subnet (`10.0.1.0/24`), same default route table, and same security lists.
  - This drastically reduces operational overhead: instead of managing 3 subnets, 3 route tables, and 3 security associations for a multi-AZ app tier, an OCI architect manages **exactly 1 regional subnet**.
- **Off-Box Network Virtualization**:
  - In AWS, hypervisor networking runs on custom Nitro ASIC cards, but VM host memory is still shared.
  - In OCI, network virtualization is completely offloaded to custom **SmartNICs** (off-box virtualization) `[Doc: OCI Virtual Networking, checked 2026-09-03]`. The host hypervisor does not process network encapsulation; packets are encapsulated directly on the SmartNIC. This delivers near-zero jitter, consistent sub-millisecond latencies, and isolates tenant network processing from the physical CPU.
- **Virtual Network Interface Cards (VNICs)**: Attach directly to compute shapes. OCI allows dynamic attachment and detachment of secondary VNICs to running instances across different VCNs and subnets.

## 7. Configuration
Comparing basic network provisioning in Terraform across AWS and OCI:

### AWS Multi-AZ VPC Architecture (Terraform)
```hcl
# AWS VPC: Requires separate zonal subnets per AZ
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "prod-vpc" }
}

resource "aws_subnet" "app_az1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  tags = { Name = "app-subnet-az1" }
}

resource "aws_subnet" "app_az2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"
  tags = { Name = "app-subnet-az2" }
}
```

### OCI Regional VCN Architecture (Terraform)
```hcl
# OCI VCN: A single regional subnet automatically spans ALL ADs
resource "oci_core_vcn" "main" {
  compartment_id = var.compartment_id
  cidr_blocks    = ["10.0.0.0/16"]
  display_name   = "prod-vcn"
  dns_label      = "prodvcn"
}

# Regional Subnet (availability_domain attribute omitted)
resource "oci_core_subnet" "app_regional" {
  compartment_id             = var.compartment_id
  vcn_id                     = oci_core_vcn.main.id
  cidr_block                 = "10.0.1.0/24"
  display_name               = "app-subnet-regional"
  dns_label                  = "app"
  prohibit_public_ip_on_vnic = true # Private subnet
}
```

## 8. Data Flow
```text
Intra-VPC / Intra-VCN Packet Traversal:
Instance A (10.0.1.5) ──> Wants to send to ──> Instance B (10.0.2.10)
        │
        ▼ 1. Guest OS Network Stack
Kernel checks routing table: 10.0.2.10 matches local VPC/VCN CIDR.
Target MAC resolved via synthetic hypervisor ARP.
        │
        ▼ 2. Hypervisor / SmartNIC Encapsulation
Nitro / SmartNIC intercepts packet:
* Wraps original packet in Geneve / VXLAN header.
* Outer Source IP: Hypervisor Host A Physical IP (Underlay).
* Outer Destination IP: Hypervisor Host B Physical IP (Underlay).
* VNI Tag: 10042 (Identifies Tenant VPC/VCN).
        │
        ▼ 3. Physical Spine-and-Leaf Clos Network
Physical switches forward outer packet using standard IP ECMP routing.
Zero inspection of internal payload or customer private IP.
        │
        ▼ 4. Destination Hypervisor / SmartNIC Decapsulation
SmartNIC on Host B strips Geneve/VXLAN header.
Validates VNI matches Target VNIC.
Injects decapsulated packet directly into Instance B's virtual interface.
```

## 9. Security
- **Strict Tenant Isolation**: Two customers sharing the same physical blade server cannot snoop on each other's traffic because the hardware SmartNIC/Nitro card drops any packet whose outer VNI does not match the recipient's provisioned interface.
- **Default Deny Rule in Security Constructs**: In both AWS and OCI, all unsolicited inbound network traffic to a compute instance is blocked by default until explicit ingress security rules are created.

## 10. Reliability
- **Avoiding MTU Mismatches in Cloud Overlays**:
  - The Geneve and VXLAN encapsulation headers add $50\text{ to }70\text{ bytes}$ of overhead to every packet traversing the physical underlay.
  - Cloud providers engineer their physical underlay to support physical MTUs of at least $9,216\text{ bytes}$, allowing customer VMs to transmit full $9,001\text{-byte}$ jumbo frames without fragmentation `[Doc: AWS Jumbo Frames, checked 2026-09-03]`.
  - When traffic leaves the VPC/VCN towards the internet or a VPN tunnel, the gateway automatically clamps the MSS (Maximum Segment Size) to prevent packet blackholes.

## 11. Scaling
- **Secondary CIDR Expansion**:
  When a VPC or VCN approaches 80% IP allocation capacity, you can attach up to 4 additional CIDR blocks without taking down running instances:
  - In AWS, secondary CIDRs must not overlap with existing VPC CIDRs or peered VPCs.
  - In OCI, secondary CIDRs can be added directly via the `oci_core_vcn` resource.
- **ENI Density Limits**: Every compute instance type has an architectural limit on how many ENIs/VNICs it can host. Sizing instance types must account for secondary IP allocation if deploying Kubernetes pods via cloud CNI plugins.

## 12. Observability
- **VPC / VCN Flow Logs**:
  Collect 100% of network traffic metadata (source, destination, port, protocol, action `ACCEPT`/`REJECT`, packet count, byte count).
  - AWS: Ships logs to CloudWatch Logs or Amazon S3 with 1-minute or 10-minute aggregation windows.
  - OCI: Ships logs to OCI Logging with native integration into Service Connector Hub for real-time streaming to Kafka/SIEMs.

## 13. Cost
- **VPC / VCN Provisioning Cost**: The VPC and VCN software-defined primitives themselves are **100% free** in both AWS and OCI. There is zero hourly charge for creating a VPC or VCN.
- **What Incurs Charges**:
  - Attached public IPv4 addresses (\$0.005/hr in AWS) `[Doc: AWS VPC Pricing, checked 2026-09-03]`.
  - NAT Gateways (\$0.045/hr + \$0.045/GB in AWS; OCI charges only for outbound internet egress, not NAT gateway hours) `[Doc: OCI Networking Pricing, checked 2026-09-03]`.
  - Cross-AZ data transfer (\$0.01/GB in/out in AWS; free in OCI).

## 14. Failure Modes
- **The Secondary CIDR Peering Routing Blindspot**: An organization adds a secondary CIDR `10.1.0.0/16` to an existing AWS VPC. Existing VPC peering connections do not automatically update their route tables. Traffic destined for the secondary CIDR is dropped at the peering boundary until the peering route tables are updated manually in both VPCs.
- **The Ephemeral Port Range Subnet Exhaustion**: A team provisions a `/26` subnet ($59$ usable IPs) for an internal load balancer. Under high traffic, the ALB automatically scales out and provisions 40 internal network interfaces across the subnet, exhausting available IPs and blocking autoscaling for the backend application tier.

## 15. Troubleshooting
When instances inside the same VPC/VCN cannot communicate:
1. **Verify Subnet Route Table**: Check that the local route (`10.0.0.0/16 -> local`) is active. In AWS, this route is implicit and cannot be deleted; in OCI, ensure the VCN default route rules do not override internal CIDR blocks.
2. **Inspect Flow Logs**: Filter VPC/VCN Flow Logs by the target IP address. Look for `REJECT` flags to determine if the packet was dropped by a firewall rule.
3. **Verify OS ARP Resolution**: Run `ip neigh show` on the instance. If the gateway IP shows `FAILED`, the instance virtual interface is misconfigured or disabled at the OS level.

## 16. Common Mistakes
- **Designing Zonal Architectures in OCI**: Creating 3 separate subnets in an OCI VCN simply because "that's how we did it in AWS". In OCI, this causes unnecessary routing complexity, fragmented IP pools, and multiple security list bindings. Use **Regional Subnets** in OCI.
- **Using Overlapping CIDRs Across Departments**: Allowing individual engineering teams to provision VPCs using `10.0.0.0/16` independently. When enterprise integration or centralized logging is required later, connecting them via Transit Gateway or peering requires complex, expensive NAT translation.

## 17. Trade-offs
| Architectural Approach | Advantage | Trade-off / Cost |
| :--- | :--- | :--- |
| **Monolithic Large VPC (`/16`)** | Simple routing, plenty of IP space | Large blast radius; harder to isolate microservices |
| **Multi-VPC Micro-Segmentation** | Strict security boundaries, isolated blast radius | High operational complexity; requires Transit Gateway and peering routing |
| **Regional Subnets (OCI)** | Simplified routing, single CIDR across all ADs | Cannot enforce AD-specific routing boundaries without extra route tables |
| **Zonal Subnets (AWS)** | Explicit physical failure domain binding | Subnet proliferation; risk of uneven IP exhaustion across AZs |

## 18. Interview Questions
1. *What is the fundamental architectural difference between how AWS and OCI handle subnet scopes, and how does this impact multi-AZ application design?*
2. *Explain how two completely unrelated AWS or OCI customers can use the exact same private IP address (`10.0.1.50`) on the same physical rack hardware without routing conflicts.*
3. *You need to connect an AWS VPC to an on-premises data center via Direct Connect, but your VPC is currently sized at `/24` and running out of IPs. How do you expand your address space without causing downtime?*

## 19. Interview Answer
**Exemplary Answer to Question 2**:
> "Two different customers can safely use the exact same private IP address on the same physical hardware because cloud providers decouple the **logical overlay network** from the **physical underlay network** using encapsulation protocols like Geneve or VXLAN.
>
> 1. **Underlay Isolation**: The physical hardware servers and switches in the data center communicate strictly using real, globally unique underlay IP addresses.
> 2. **Virtual Network Identifiers (VNIs)**: When Customer A's VM sends a packet from `10.0.1.50`, the custom hardware hypervisor (AWS Nitro ASIC or OCI SmartNIC) intercepts the packet at the virtual interface layer.
> 3. **Encapsulation**: The hypervisor wraps Customer A's original packet inside an outer Geneve or VXLAN packet. In the outer header, it stamps a unique 24-bit **Virtual Network Identifier (VNI)** corresponding strictly to Customer A's VPC (e.g., `VNI: 10042`). Customer B's packet on the same physical host is encapsulated with `VNI: 90210`.
> 4. **Delivery & Decapsulation**: The physical data center switches route the outer packet based purely on physical hypervisor destination IPs. When the packet arrives at the receiving host, the hypervisor inspects the VNI tag. If the VNI does not match Customer A's provisioned interface, the packet is immediately dropped in hardware.
>
> Thus, customer private IP spaces are completely invisible to the physical network, and identical private IPs can coexist across millions of independent overlay networks with zero routing conflict."

## 20. Hands-on Exercise
**Objective**: Query and verify VPC/VCN properties and inspect overlay network configuration.

### Verification Steps
1. In AWS CLI, inspect VPC CIDR associations:
   ```bash
   aws ec2 describe-vpcs --vpc-ids <vpc-id> \
     --query "Vpcs[*].CidrBlockAssociationSet[*].[CidrBlock, CidrBlockState.State]" --output table
   ```
2. In OCI CLI, inspect VCN CIDR blocks:
   ```bash
   oci network vcn get --vcn-id <vcn-id> \
     --query "data.[\"cidr-blocks\", \"display-name\"]" --output table
   ```
3. On a running Linux cloud instance, inspect MTU and verify jumbo frame support:
   ```bash
   ip link show eth0
   ```
   *Expected Result*: MTU will display `9001` (AWS) or `9000` (OCI).
