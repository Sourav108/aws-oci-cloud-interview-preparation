# 03. Peering, Transit Gateway & Dynamic Routing Gateway (DRG)

## 1. Problem
As enterprise cloud footprints expand from a single development network into dozens or hundreds of distinct VPCs and VCNs across multiple accounts, tenancies, regions, and on-premises data centers, network topology complexity explodes. If engineers rely solely on direct point-to-point peering, the number of required connections scales quadratically:
$$\text{Number of Peering Connections} = \frac{N(N - 1)}{2}$$
Connecting 50 VPCs requires $\frac{50 \times 49}{2} = \mathbf{1,225\text{ separate peering links}}$ and tens of thousands of routing table entries. Furthermore, standard cloud peering is strictly **non-transitive**, meaning that routing through an intermediate VPC is dropped by the cloud hypervisor.

## 2. Cloud Concept
### Point-to-Point Peering vs. Hub-and-Spoke Transit Routing
- **Cloud Peering (AWS VPC Peering / OCI LPG & RPG)**:
  - A 1-to-1 direct virtual connection between two networks.
  - Traffic flows over the cloud provider's private optical backbone with zero single-point-of-failure and zero bandwidth bottlenecks.
  - **The Non-Transitive Rule**: If Network A is peered with Network B, and Network B is peered with Network C, **Network A cannot talk to Network C through Network B**. Traffic attempting to traverse an intermediate network is dropped.
- **Hub-and-Spoke Transit Routing (AWS Transit Gateway / OCI DRG v2)**:
  - A centralized, highly scalable software-defined regional virtual router.
  - VPCs, VCNs, VPNs, and Direct Connect / FastConnect circuits attach to the central hub.
  - **Full Transitive Routing**: Spoke VPC A can route traffic to Spoke VPC C via the central hub using standard route table lookups.

## 3. Mental Model
Think of cloud network interconnection as an airline flight network:
- **Point-to-Point Peering** is a **Direct Non-Stop Charter Flight**. You fly directly from City A to City B. It is the fastest, lowest-latency path with zero layovers, but creating direct non-stop flights between every single pair of cities in the world is mathematically impossible.
- **Transit Hubs (AWS TGW / OCI DRG)** are **Major Airport Hubs (e.g., Frankfurt, Atlanta, Chicago)**. Every spoke city runs a single flight to the central hub. Passengers change planes at the hub to reach any destination in the network. Adding a new city requires only 1 new flight route to the hub, not 50 new routes.

## 4. Architecture Diagram
```text
QUADRATIC POINT-TO-POINT PEERING (N=4: 6 Links | N=50: 1,225 Links):
[VPC A] ───────────── [VPC B]
   │    ╲           ╱    │
   │      ╲       ╱      │
   │        ╲   ╱        │
   │          ╳          │
   │        ╱   ╲        │
   │      ╱       ╲      │
   │    ╱           ╲    │
[VPC C] ───────────── [VPC D]

SCALABLE HUB-AND-SPOKE TRANSIT TOPOLOGY (N Connections):
[Spoke VPC / VCN A] ───┐                     ┌─── [Spoke VPC / VCN B]
                       ▼                     ▼
          ┌───────────────────────────────────────┐
          │  CENTRAL CLOUD TRANSIT ROUTING HUB    │
          │  * AWS Transit Gateway (TGW)          │
          │  * OCI Dynamic Routing Gateway (DRG)  │
          │  (Route Tables / Route Domains / VRF) │
          └──────────────────┬────────────────────┘
                             ▲
         ┌───────────────────┴───────────────────┐
         ▼                                       ▼
[Shared Services VPC / VCN]           [On-Premises DirectConnect / FastConnect]
```

## 5. AWS Implementation
In AWS:
- **AWS VPC Peering**:
  - Connects two VPCs within the same region or across regions (Inter-Region VPC Peering).
  - Encrypted in flight using modern AEAD ciphers over the private AWS backbone.
  - Zero hourly charge; users pay only standard cross-AZ / cross-region data transfer fees `[Doc: Amazon VPC Peering Pricing, checked 2026-09-03]`.
  - Strictly non-transitive: cannot route on-premises Direct Connect traffic through a peered VPC.
- **AWS Transit Gateway (TGW)**:
  - A regional network transit hub powered by the AWS Hyperplane distributed fabric.
  - Connects up to **5,000 VPCs** and on-premises VPN/Direct Connect connections per TGW `[Doc: AWS Transit Gateway Quotas, checked 2026-09-03]`.
  - Supports multiple independent **Transit Gateway Route Tables** to build isolated network domains (e.g., Production VRF, Development VRF, Shared Services VRF).
  - Can peer with other Transit Gateways in different AWS regions (Inter-Region TGW Peering).

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **Local Peering Gateway (LPG) & Remote Peering Gateway (RPG)**:
  - *LPG*: 1-to-1 peering between two VCNs within the **same region**. Supports cross-tenancy peering.
  - *RPG*: 1-to-1 peering between two VCNs in **different regions**.
- **The Dynamic Routing Gateway (DRG v2)**:
  - **MAJOR ARCHITECTURAL ADVANTAGE IN OCI**:
  - In OCI, the **Dynamic Routing Gateway (DRG v2)** is an ultra-versatile, enterprise-grade virtual router that unifies all transit routing into a single primitive `[Doc: OCI Dynamic Routing Gateway, checked 2026-09-03]`.
  - A single DRG can attach to:
    1. VCNs in the same region.
    2. IPsec VPN tunnels to on-premises data centers.
    3. OCI FastConnect virtual circuits.
    4. Remote Peering connections to DRGs in other OCI regions.
    5. Cross-tenancy VCN attachments.
  - **Full Transitive Routing & VRF Support**: DRG v2 acts as a full transit hub. Spoke VCNs can route to each other, to on-premises, and across regions through a single DRG.
  - **Dynamic Route Distribution**: Supports automated route distribution, ECMP routing across multiple IPsec tunnels, and independent DRG route tables.
  - **Cost Advantage**: Unlike AWS TGW, which charges an hourly attachment fee per VPC, **OCI DRG has zero hourly attachment fees** `[Doc: OCI Networking Pricing, checked 2026-09-03]`.

## 7. Configuration
Comparing hub-and-spoke transit configuration in Terraform:

### AWS Transit Gateway Attachment (Terraform)
```hcl
# Create AWS Transit Gateway
resource "aws_ec2_transit_gateway" "central_hub" {
  description                     = "Central transit gateway"
  auto_accept_shared_attachments  = "enable"
  default_route_table_association = "enable"
  default_route_table_propagation = "enable"
  tags = { Name = "central-tgw" }
}

# Attach Spoke VPC A to Transit Gateway
resource "aws_ec2_transit_gateway_vpc_attachment" "spoke_a" {
  transit_gateway_id = aws_ec2_transit_gateway.central_hub.id
  vpc_id             = var.spoke_a_vpc_id
  subnet_ids         = var.spoke_a_tgw_subnet_ids # Dedicated /28 subnets per AZ!

  tags = { Name = "tgw-attachment-spoke-a" }
}

# Route in Spoke VPC A directing traffic destined for Spoke VPC B (10.2.0.0/16) to TGW
resource "aws_route" "spoke_a_to_tgw" {
  route_table_id         = var.spoke_a_route_table_id
  destination_cidr_block = "10.2.0.0/16"
  transit_gateway_id     = aws_ec2_transit_gateway.central_hub.id
}
```

### OCI Dynamic Routing Gateway (DRG v2) Attachment (Terraform)
```hcl
# Create OCI DRG v2
resource "oci_core_drg" "central_drg" {
  compartment_id = var.compartment_id
  display_name   = "central-hub-drg"
}

# Attach Spoke VCN A to DRG
resource "oci_core_drg_attachment" "spoke_a" {
  drg_id       = oci_core_drg.central_drg.id
  display_name = "drg-attachment-spoke-a"

  network_details {
    id   = var.spoke_a_vcn_id
    type = "VCN"
  }
}

# Route in Spoke VCN A directing traffic destined for Spoke VCN B (10.2.0.0/16) to DRG
resource "oci_core_route_table" "spoke_a_rt" {
  compartment_id = var.compartment_id
  vcn_id         = var.spoke_a_vcn_id
  display_name   = "spoke-a-route-table"

  route_rules {
    destination       = "10.2.0.0/16"
    destination_type  = "CIDR_BLOCK"
    network_entity_id = oci_core_drg.central_drg.id # Routed via DRG!
  }
}
```

## 8. Data Flow
```text
Inter-VPC Packet Flow via Transit Hub:
Instance in Spoke VPC A (10.1.1.5) ──> Wants to call ──> Instance in Spoke VPC B (10.2.1.10)
        │
        ▼ 1. Subnet Route Table Lookup
Kernel matches route: 10.2.0.0/16 targets Transit Gateway ENI / DRG Attachment.
        │
        ▼ 2. Crosses Hyperplane / SmartNIC Fabric into Hub
AWS TGW / OCI DRG receives packet:
* Evaluates TGW / DRG Route Table.
* Identifies egress attachment: Spoke VPC B Attachment.
* Verifies security domains / route propagation allow traffic.
        │
        ▼ 3. Enters Destination Subnet
Packet delivered to Spoke VPC B virtual interface.
Target instance receives packet without SNAT (Source IP remains 10.1.1.5).
```

## 9. Security
- **Network Segmentation via Route Domains (VRFs)**:
  - Create separate Transit Gateway Route Tables (e.g., `Prod_RT`, `Dev_RT`, `Sec_Inspection_RT`).
  - Production VPC attachments associate only with `Prod_RT`; Development VPCs associate only with `Dev_RT`.
  - This mathematically isolates production from development across the entire enterprise network.
- **Centralized Firewall Inspection Appliance (Bump-in-the-Wire)**:
  - Route all inter-VPC and internet-bound traffic through an **Inspection VPC** containing AWS Network Firewall, OCI Network Firewall (Palo Alto-powered), or third-party firewall clusters before routing to the destination spoke.

## 10. Reliability
- **Multi-AZ Attachment Architecture in AWS**:
  - When attaching a VPC to an AWS Transit Gateway, AWS provisions an internal cross-account Elastic Network Interface (ENI) in **every AZ** selected.
  - *Reliability Rule*: Always select at least **3 AZs** during attachment. Best practice: Provision a dedicated, isolated `/28` subnet in each AZ solely for the Transit Gateway ENIs to prevent IP address churn.
- **OCI DRG High Availability**: The DRG is regional and automatically fault-tolerant across all Availability Domains and Fault Domains.

## 11. Scaling
- **Throughput Scalability**:
  - AWS Transit Gateway supports up to **50 Gbps of burst throughput per VPC attachment** `[Doc: AWS Transit Gateway Quotas, checked 2026-09-03]`. For higher bandwidth, deploy multiple attachments or use ECMP across VPN connections.
  - OCI DRG v2 delivers wire-speed throughput limited only by underlying compute VNIC bandwidth shapes (up to 100 Gbps per bare metal shape).

## 12. Observability
- **Transit Gateway Flow Logs**: Capture packet metrics at the transit hub layer to monitor inter-VPC bandwidth consumers.
- **OCI Network Visualizer**: Native console tool in OCI that renders a real-time topology map of DRG attachments, route tables, and VCN connections.

## 13. Cost
- **AWS Transit Gateway Pricing**:
  - Hourly fee: **\$0.05 per hour per VPC attachment** ($\approx \$36/\text{month per VPC}$) `[Doc: AWS Transit Gateway Pricing, checked 2026-09-03]`.
  - Data processing fee: **\$0.02 per GB of data processed**.
  - In an enterprise with 50 VPCs transferring 50 TB/month, TGW costs:
    $$(50 \times \$36) + (50,000 \times \$0.02) = \$1,800 + \$1,000 = \mathbf{\$2,800/\text{month}}.$$
- **OCI DRG Cost Advantage**:
  - OCI DRG v2 incurs **zero hourly attachment fees**. There is no cost for attaching VCNs to a DRG `[Doc: OCI Networking Pricing, checked 2026-09-03]`. Inter-VCN data transfer within the same region is **100% free**.

## 14. Failure Modes
- **The Asymmetric Return Route Blackhole**: A route table in Spoke A directs `10.2.0.0/16` to the Transit Gateway, but the route table in Spoke B lacks a return route for `10.1.0.0/16`. Packets arrive at Spoke B, but response packets are dropped by Spoke B's default route.
- **The Blackhole Subnet Deletion in AWS**: An administrator deletes the subnet hosting the Transit Gateway attachment ENI in `us-east-1a`. All cross-VPC traffic originating from or destined for instances in `us-east-1a` is silently blackholed.

## 15. Troubleshooting
When instances in Spoke A cannot reach Spoke B through a Transit Gateway or DRG:
1. **Trace Route Tables at 3 Distinct Layers**:
   - Layer 1: Spoke A Subnet Route Table (Does `10.2.0.0/16` point to TGW/DRG?).
   - Layer 2: Transit Gateway / DRG Route Table (Is Spoke B's attachment propagated in the hub route table?).
   - Layer 3: Spoke B Subnet Route Table (Does `10.1.0.0/16` point back to TGW/DRG?).
2. **Inspect Security Groups at Both Ends**: Does Spoke B's Security Group permit ingress from Spoke A's private IP CIDR?
3. **Verify Transit Gateway Subnet Association**: In AWS, ensure the instance's AZ has an active TGW attachment ENI provisioned.

## 16. Common Mistakes
- **Using Full-Mesh VPC Peering at Scale**: Attempting to connect 40 VPCs using direct VPC peering links. The organization drowns in routing table updates whenever a new VPC is provisioned.
- **Forgetting that Peering is Non-Transitive**: Setting up VPC Peering between VPC A and VPC B (which hosts a Corporate DirectConnect VPN to on-premises) and expecting instances in VPC A to reach on-premises. AWS drops the packets; on-premises transit mandates Transit Gateway or Direct Connect Gateway.

## 17. Trade-offs
| Solution | Architecture | Hourly Fee | Latency | Management Complexity |
| :--- | :--- | :---: | :---: | :---: |
| **VPC / VCN Peering** | Point-to-Point Mesh | Free | Lowest (< 1ms) | Quadratic ($O(N^2)$) — Unmanageable at scale |
| **AWS Transit Gateway** | Hub-and-Spoke Router | \$0.05/hr/att + \$0.02/GB | Low (1–2ms) | Linear ($O(N)$) — Highly scalable |
| **OCI DRG v2** | Hub-and-Spoke Router | **Free** | Lowest (< 1ms) | Linear ($O(N)$) — Enterprise standard |

## 18. Interview Questions
1. *You have 100 VPCs that need to communicate with a shared services VPC and an on-premises data center. Would you choose VPC Peering or AWS Transit Gateway? Calculate and defend the cost and architectural trade-offs.*
2. *Explain the concept of 'non-transitive routing' in cloud peering. Why do cloud providers enforce this restriction?*
3. *How does OCI's Dynamic Routing Gateway (DRG v2) compare to AWS Transit Gateway in terms of capabilities and billing economics?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "For an enterprise with 100 VPCs requiring communication with a shared services VPC and an on-premises data center, **AWS Transit Gateway (TGW) is the mandatory architectural choice**:
>
> 1. **The Peering Scaling Failure**:
>    - Full-mesh peering across 100 VPCs would require $\frac{100 \times 99}{2} = \mathbf{4,950\text{ individual peering connections}}$, which breaches AWS service quotas and creates operational chaos.
>    - Even a hub-and-spoke peering topology (peering all 100 spoke VPCs only to the Shared Services VPC) fails because **VPC Peering is non-transitive**: the spokes cannot transit through the Shared Services VPC to reach the on-premises Direct Connect link.
>
> 2. **The Transit Gateway Solution**:
>    - We deploy a regional AWS Transit Gateway. Each of the 100 VPCs establishes a single attachment ($O(N)$ linear complexity).
>    - On-premises Direct Connect connects directly to the TGW via a Direct Connect Gateway. Spoke VPCs access on-premises resources seamlessly through the TGW hub.
>    - We implement two TGW Route Tables:
>      - `Spoke_Route_Table`: Directs on-premises traffic (`10.0.0.0/8`) and Shared Services traffic (`172.16.0.0/16`) to their respective attachments, while isolating spokes from talking directly to each other.
>      - `Shared_Services_Route_Table`: Propagates routes to all 100 spoke VPCs.
>
> 3. **Cost Defense**:
>    - 100 attachments at \$0.05/hr costs $\$3,600/\text{month}$ plus \$0.02/GB data processing.
>    - In a Staff interview, you defend this expenditure by proving that the engineering labor required to manually manage thousands of peering links and build custom NAT proxy EC2 instances far exceeds $\$3,600/\text{month}$, while introducing massive single-points-of-failure."

## 20. Hands-on Exercise
**Objective**: Verify non-transitive routing behavior by simulating a 3-VPC mesh.

### Verification Steps
1. Create 3 VPCs: VPC-A (`10.1.0.0/16`), VPC-B (`10.2.0.0/16`), and VPC-C (`10.3.0.0/16`).
2. Create VPC Peering A $\longleftrightarrow$ B and Peering B $\longleftrightarrow$ C. Do **not** create Peering A $\longleftrightarrow$ C.
3. In VPC-A's route table, add route: `10.3.0.0/16` pointing to Peering A-B.
4. From an instance in VPC-A, execute `ping 10.3.1.5` (an instance in VPC-C).
5. Observe that packets are dropped immediately at the VPC-B hypervisor boundary, proving that cloud peering is strictly non-transitive.
