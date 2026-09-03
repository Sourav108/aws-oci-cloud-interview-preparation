# 01. IPv4, IPv6, CIDR & Subnet Math

## 1. Problem
Subnetting miscalculations are among the most catastrophic and difficult-to-remediate mistakes in cloud architecture. When an organization provisions an initial Virtual Private Cloud (VPC) or Virtual Cloud Network (VCN) with an undersized CIDR block (e.g., `/24` with only 250 usable IPs), the network quickly exhausts available IP addresses as Kubernetes pods, container tasks, load balancers, and RDS instances scale out. Conversely, using arbitrary overlapping IP ranges (e.g., using `10.0.0.0/16` everywhere) makes subsequent enterprise mergers, on-premises DirectConnect/FastConnect integration, and cross-VPC peering impossible without complex, expensive Source NAT workarounds.

## 2. Cloud Concept
### IPv4 & CIDR Notation
An IPv4 address is a 32-bit binary integer divided into four 8-bit octets:
$$\text{IP Address} = \underbrace{10}_{\text{8 bits}} \cdot \underbrace{0}_{\text{8 bits}} \cdot \underbrace{1}_{\text{8 bits}} \cdot \underbrace{50}_{\text{8 bits}} = 32\text{ bits total}$$

**Classless Inter-Domain Routing (CIDR)** defines the boundary between the **Network Prefix** (identifying the subnet) and the **Host Identifier** (identifying the specific network interface):
$$\text{Total Addresses in a } /N \text{ Subnet} = 2^{(32 - N)}$$

| CIDR Prefix | Subnet Mask | Total IPv4 Addresses | Use Case in Cloud Architecture |
| :---: | :--- | :---: | :--- |
| `/16` | `255.255.0.0` | $65,536$ | Standard maximum size for an AWS VPC or OCI VCN |
| `/20` | `255.255.240.0` | $4,096$ | Large enterprise microservice tier or EKS/OKE pod subnet |
| `/24` | `255.255.255.0` | $256$ | Standard workload or database subnet |
| `/28` | `255.255.255.240` | $16$ | Minimum allowed subnet size in AWS VPC |
| `/30` | `255.255.255.252` | $4$ | Point-to-point interconnects (supported in OCI) |

### Cloud Provider IP Reservation Rules
In standard networking, RFC 1918 dictates that only 2 addresses are unusable: the **Network Address** (`.0`) and the **Broadcast Address** (`.255`). However, **cloud providers reserve additional IP addresses** inside every provisioned subnet for hypervisor and virtual routing functions:

$$\text{Usable IPs in AWS Subnet} = 2^{(32 - N)} - 5$$
$$\text{Usable IPs in OCI Subnet} = 2^{(32 - N)} - 3$$

## 3. Mental Model
Think of CIDR allocation as slicing a physical pie:
- A `/16` block is the whole pie ($65,536$ crumbs).
- Slicing the pie in half gives two `/17` blocks ($32,768$ crumbs each).
- Slicing into four gives four `/18` blocks ($16,384$ crumbs each).
- Slicing into 256 equal pieces gives two hundred and fifty-six `/24` blocks ($256$ crumbs each).
Once a slice of pie is allocated to a specific VPC or subnet, you cannot give those same crumbs to another network without causing an immediate IP address collision.

## 4. Architecture Diagram
```text
┌────────────────────────────────────────────────────────────────────────┐
│                   AWS /24 SUBNET: 10.0.1.0/24                          │
│                                                                        │
│  [10.0.1.0]   ──> Network Address (Reserved by RFC 1918)               │
│  [10.0.1.1]   ──> VPC Virtual Router (Reserved by AWS)                 │
│  [10.0.1.2]   ──> AmazonProvidedDNS (Reserved by AWS: Base IP + 2)     │
│  [10.0.1.3]   ──> Future Use / Hypervisor (Reserved by AWS)            │
│  [10.0.1.4]   ──> FIRST USABLE IP ADDRESS FOR CUSTOMER ENI             │
│       │                                                                │
│      ...      ──> 251 Usable Customer Host IPs (10.0.1.4 to 10.0.1.254) │
│       │                                                                │
│  [10.0.1.254] ──> LAST USABLE IP ADDRESS FOR CUSTOMER ENI              │
│  [10.0.1.255] ──> Network Broadcast Address (Reserved by AWS)          │
├────────────────────────────────────────────────────────────────────────┤
│                   OCI /24 SUBNET: 10.0.1.0/24                          │
│                                                                        │
│  [10.0.1.0]   ──> Network Address (Reserved by RFC 1918)               │
│  [10.0.1.1]   ──> Default Gateway / Virtual Router (Reserved by OCI)   │
│  [10.0.1.2]   ──> FIRST USABLE IP ADDRESS FOR CUSTOMER VNIC            │
│       │                                                                │
│      ...      ──> 253 Usable Customer Host IPs (10.0.1.2 to 10.0.1.254) │
│       │                                                                │
│  [10.0.1.255] ──> Network Broadcast Address (Reserved by OCI)          │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS Amazon VPC:
- **VPC CIDR Size Limits**: A VPC must be between `/16` ($65,536$ IPs) and `/28` ($16$ IPs) `[Doc: Amazon VPC Quotas, checked 2026-09-03]`.
- **The 5 Reserved Subnet IPs**:
  1. `10.0.1.0`: Network address.
  2. `10.0.1.1`: Reserved by AWS for the VPC router.
  3. `10.0.1.2`: Reserved by AWS for DNS mapping (`AmazonProvidedDNS` resolves at the VPC network range $+ 2$ or `169.254.169.253`).
  4. `10.0.1.3`: Reserved by AWS for future internal operations.
  5. `10.0.1.255`: Network broadcast address (VPCs do not support broadcast, but the IP remains reserved).
- **Secondary CIDR Blocks**: If a VPC runs out of IP addresses, AWS allows associating up to 5 secondary IPv4 CIDR blocks to the VPC without tearing down existing infrastructure.
- **VPC IPv6**: AWS assigns a fixed `/56` IPv6 CIDR block automatically upon request. Subnets in an IPv6-enabled VPC are always allocated a fixed `/64` CIDR block ($18,446,744,073,709,551,616$ addresses).

## 6. OCI Implementation
In Oracle Cloud Infrastructure VCN:
- **VCN CIDR Size Limits**: An OCI VCN supports CIDRs from `/16` ($65,536$ IPs) down to `/30` ($4$ IPs) `[Doc: OCI VCN Overview, checked 2026-09-03]`.
- **The 3 Reserved Subnet IPs**:
  1. `10.0.1.0`: Network address.
  2. `10.0.1.1`: Default gateway / virtual router for the subnet.
  3. `10.0.1.255`: Network broadcast address.
  - *Critical Difference*: **OCI does not reserve `.2` or `.3`**. The first usable IP address on an OCI subnet is `10.0.1.2`. DNS resolution is handled directly by the default gateway (`10.0.1.1`) or via a dedicated VCN Private DNS Resolver.
- **Multiple VCN CIDRs**: OCI allows up to 5 non-overlapping IPv4 CIDR blocks per VCN out-of-the-box.
- **Regional Subnets Math**: Because an OCI subnet is **Regional by default**, the subnet's IP address pool is shared across all Availability Domains. If you allocate a `/24` ($253$ usable IPs), those 253 IPs can be dynamically assigned to instances running in AD-1, AD-2, or AD-3 without fragmenting the CIDR block into per-AD slices.

## 7. Configuration
Comparing subnet provisioning and calculating available host IPs in Terraform:

### AWS Subnet Definition (Terraform)
```hcl
# AWS Subnet in us-east-1a: 256 total - 5 reserved = 251 usable IPs
resource "aws_subnet" "private_workload" {
  vpc_id            = var.vpc_id
  cidr_block        = "10.0.10.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "private-app-us-east-1a"
    # Documenting usable hosts: 10.0.10.4 through 10.0.10.254
  }
}
```

### OCI Subnet Definition (Terraform)
```hcl
# OCI Regional Subnet: 256 total - 3 reserved = 253 usable IPs across ALL ADs
resource "oci_core_subnet" "private_workload" {
  compartment_id = var.compartment_id
  vcn_id         = var.vcn_id
  cidr_block     = "10.0.10.0/24"
  display_name   = "private-app-regional"

  # Omission of availability_domain makes this a Regional Subnet!
  prohibit_public_ip_on_vnic = true
  dns_label                  = "privapp"
}
```

## 8. Data Flow
```text
Host ARP Resolution Flow in Cloud Hypervisor:
Instance (10.0.1.4) wants to talk to Database (10.0.2.10)
  │
  ├──> 1. Instance checks subnet mask: 10.0.2.10 is outside local /24 subnet.
  │
  ├──> 2. Instance broadcasts ARP for Default Gateway IP:
  │       - AWS: ARP for 10.0.1.1
  │       - OCI: ARP for 10.0.1.1
  │
  ├──> 3. Cloud Hypervisor (Nitro / SmartNIC) intercepts ARP query directly:
  │       - Instantly returns synthetic router MAC (e.g., 12:34:56:78:9a:bc)
  │       - Zero physical broadcast traffic floods the data center network!
  │
  └──> 4. Packet encapsulated into Geneve/VXLAN overlay and routed to target.
```

## 9. Security
- **Subnet Separation & Egress Boundaries**:
  - *Public Subnets*: Subnets with a route table directing `0.0.0.0/0` to an Internet Gateway (IGW).
  - *Private Subnets*: Route table directs `0.0.0.0/0` to a NAT Gateway.
  - *Isolated / Database Subnets*: Route table contains **no default route** (`0.0.0.0/0`) whatsoever, preventing all internet communication even if an application is compromised.
- **CIDR Restriction in Security Rules**: Never use `0.0.0.0/0` in ingress rules. Scope ingress rules strictly to peer security group IDs or narrow internal CIDR blocks (e.g., `10.0.0.0/8`).

## 10. Reliability
- **Avoiding Subnet Exhaustion Outages**:
  When a Kubernetes cluster (EKS with VPC CNI or OKE with VCN-Native Pod Networking) scales out during a traffic surge, each pod requests an independent secondary IP address from the underlying subnet.
  If the subnet is sized at `/24` ($251$ usable IPs), and each node runs 30 pods, the subnet will be completely exhausted after only 8 nodes ($8 \times 30 = 240\text{ IPs}$). Subsequent pod deployments will crash with `FailedCreatePodSandBox` errors.
  *Standard*: Size container subnets at minimum `/20` ($4,096$ IPs) or `/19` ($8,192$ IPs).

## 11. Scaling
- **Non-Overlapping Enterprise CIDR Allocation Plan**:
  To allow seamless inter-VPC peering, Transit Gateway routing, and on-premises VPN links, enterprise cloud networks allocate IP blocks hierarchically from RFC 1918 space (`10.0.0.0/8`):

```text
RFC 1918 Corporate Supernet: 10.0.0.0/8
 ├── Production Environment (10.0.0.0/12)
 │    ├── VPC Production US-East (10.0.0.0/16)
 │    │    ├── Public Subnets (10.0.0.0/20)
 │    │    ├── App Tier Subnets (10.0.16.0/20)
 │    │    └── DB Tier Subnets (10.0.32.0/20)
 │    └── VPC Production EU-Central (10.1.0.0/16)
 └── Non-Production Environment (10.16.0.0/12)
      ├── Staging VPC (10.16.0.0/16)
      └── Development VPC (10.17.0.0/16)
```

## 12. Observability
- **Subnet Available IP Telemetry**:
  - AWS: Monitor `AvailableIpAddressCount` via `aws ec2 describe-subnets`. Create CloudWatch alarms when available IPs fall below 20% of capacity.
  - OCI: Query `oci network subnet get` to inspect IP allocation density.

## 13. Cost
- **Public IPv4 Address Charges**:
  Cloud providers now charge for all public IPv4 addresses (even when attached to running instances):
  - AWS charges **\$0.005 per hour per public IPv4 address** ($\approx \$3.65/\text{month per IP}$) `[Doc: AWS VPC Pricing, checked 2026-09-03]`.
  - In an architecture with 1,000 instances, assigning public IPs costs \$3,650/month in waste. Placing instances in private subnets with a NAT Gateway eliminates this charge.

## 14. Failure Modes
- **The Transit Gateway Overlap Collision**: Two corporate subsidiaries both build their cloud environments using `10.0.0.0/16`. The parent company attempts to connect both VPCs to a central AWS Transit Gateway or OCI Dynamic Routing Gateway (DRG). The virtual router rejects the connection because routing tables cannot distinguish between two identical destination CIDR blocks. *Mitigation: Requires deploying complex bidirectional Private NAT Gateway translation.*
- **The EKS CNI Secondary IP Starvation**: An engineer provisions a `/24` subnet for an EKS cluster. The AWS VPC CNI assigns 1 primary IP and 15 secondary IPs per ENI to speed up pod scheduling. The subnet runs out of IP addresses before a single application pod is deployed because the daemonset pre-allocated all available IPs.

## 15. Troubleshooting
When an instance or pod reports IP allocation failure:
1. **Query Available IPs**:
   ```bash
   aws ec2 describe-subnets --subnet-ids subnet-12345 \
     --query "Subnets[*].[SubnetId, AvailableIpAddressCount, CidrBlock]" --output table
   ```
2. **Inspect Secondary ENIs**: Check if terminated pods or instances left orphaned ENIs in `available` status hoarding IPs.
3. **Emergency Workaround**: Associate a secondary CIDR block (e.g., `100.64.0.0/16` from CGNAT space) to the VPC and attach a new subnet to the cluster.

## 16. Common Mistakes
- **Assuming `.1` is Available for a Database in AWS**: Attempting to assign `10.0.1.1` as a static private IP to a database server. In AWS, `.1` is permanently reserved for the VPC virtual router.
- **Confusing Subnet Mask Length with Capacity**: Thinking a `/28` subnet can hold 28 hosts. A `/28` provides $2^{(32-28)} = 16$ total IPs. In AWS, subtracting 5 reserved IPs leaves only **11 usable hosts**.

## 17. Trade-offs
| Subnet Size | Usable Hosts (AWS) | Advantage | Risk / Trade-off |
| :---: | :---: | :--- | :--- |
| `/28` | $11$ | Conserves precious corporate IP space | Instant exhaustion; no room for autoscaling |
| `/24` | $251$ | Clean boundary, simple mental math | Rapidly exhausted by container workloads |
| `/20` | $4,091$ | Massive headroom for Kubernetes pods & tasks | Consumes large chunk of corporate CIDR block |

## 18. Interview Questions
1. *In an AWS `/24` subnet, exactly how many IP addresses are usable for EC2 instances, and what is each reserved IP used for? How does this differ in OCI?*
2. *You need to design a VPC that will host an EKS cluster scaling to 500 pods. What CIDR prefix do you allocate to the pod subnets, and why?*
3. *Two companies merge and need to interconnect their AWS VPCs, but both use `10.0.0.0/16`. How do you enable bidirectional communication without re-architecting either company's subnets?*

## 19. Interview Answer
**Exemplary Answer to Question 3**:
> "When two networks have identical overlapping CIDR blocks (`10.0.0.0/16`), standard VPC Peering, Transit Gateway, or Direct Routing Gateways cannot route traffic because the routing engine cannot determine which physical VPC owns the `10.0.X.X` destination.
>
> To solve this without ripping out and re-addressing either company's infrastructure, we have three architectural options:
>
> 1. **Private NAT Gateway (The Dual-NAT Pattern)**:
>    - In Company A's VPC, we assign a secondary, non-overlapping CIDR block from the Carrier-Grade NAT (CGNAT) RFC 6598 space (e.g., `100.64.0.0/16`).
>    - We provision an AWS **Private NAT Gateway** in this secondary subnet.
>    - When an instance in Company A talks to Company B, traffic is source-translated to an IP in `100.64.X.X`. Company B sees traffic originating from a unique, non-overlapping IP space and routes response packets back without conflict.
> 2. **AWS PrivateLink / OCI Private Endpoint (The Recommended Microservices Pattern)**:
>    - If the two environments only need to share specific APIs (e.g., an Order Service or Billing Service), we eliminate network-level routing entirely.
>    - We expose Company B's service behind a Network Load Balancer (NLB) attached to an AWS PrivateLink Service.
>    - In Company A, we create an Interface VPC Endpoint inside a non-conflicting secondary subnet. Traffic flows over AWS's private Hyperplane backbone using local endpoint IPs, completely bypassing network CIDR overlap restrictions."

## 20. Hands-on Exercise
**Objective**: Calculate and verify subnet IP availability using Python and CLI inspection.

### Subnet Math Validation Script (Python)
```python
import ipaddress

def analyze_cloud_subnet(cidr_str):
    net = ipaddress.IPv4Network(cidr_str)
    total_ips = net.num_addresses
    aws_usable = max(0, total_ips - 5)
    oci_usable = max(0, total_ips - 3)

    print(f"CIDR Block: {cidr_str}")
    print(f"Total Addresses: {total_ips}")
    print(f"AWS Usable IPs:  {aws_usable} (First: {net[4]}, Last: {net[-2]})")
    print(f"OCI Usable IPs:  {oci_usable} (First: {net[2]}, Last: {net[-2]})")
    print(f"AWS Reserved:    {net[0]} (Net), {net[1]} (Router), {net[2]} (DNS), {net[3]} (Future), {net[-1]} (Broadcast)")
    print(f"OCI Reserved:    {net[0]} (Net), {net[1]} (Gateway), {net[-1]} (Broadcast)\n")

analyze_cloud_subnet("10.0.1.0/24")
analyze_cloud_subnet("10.0.2.0/28")
```
