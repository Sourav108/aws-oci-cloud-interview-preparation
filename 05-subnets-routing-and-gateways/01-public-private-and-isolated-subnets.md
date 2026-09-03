# 01. Public, Private & Isolated Subnets

## 1. Problem
A staggering number of security incidents occur because engineering teams deploy databases or backend microservices into subnets with unintended internet routing. When an engineer configures an application to listen on `0.0.0.0` inside a subnet whose route table directs default traffic (`0.0.0.0/0`) to an Internet Gateway, attaching an accidental public IP exposes internal database listeners or administrative ports directly to global internet port scanners. Defense-in-depth requires enforcing architectural isolation at the **Route Table** level, not merely relying on application-level firewalls.

## 2. Cloud Concept
In enterprise cloud networking, subnets are classified into three distinct tiers based strictly on their **Route Table Destination Targets**:

1. **Public Subnet (Ingress Tier)**:
   - Contains a route table entry: `0.0.0.0/0` $\longrightarrow$ **Internet Gateway (IGW)**.
   - Instances can possess public IPv4 addresses and can send and receive traffic directly to/from the internet.
   - *Production Workloads*: Public Load Balancers (ALB, NLB, OCI LB), Bastion Hosts, and VPN Gateways.
2. **Private Subnet (Application Tier)**:
   - Contains a route table entry: `0.0.0.0/0` $\longrightarrow$ **NAT Gateway**.
   - Instances possess **private IP addresses only**. They can initiate outbound connections to the internet (to pull software patches, container images, or call third-party SaaS APIs), but external internet clients cannot initiate inbound connections.
   - *Production Workloads*: Application microservices, Kubernetes worker nodes, background queues, and caching clusters.
3. **Isolated / Dedicated Subnet (Data Tier)**:
   - Contains **no default route (`0.0.0.0/0`) whatsoever**.
   - Outbound internet access is physically unroutable. Traffic is strictly restricted to local VPC/VCN CIDR ranges, VPC Endpoints, or internal peering gateways.
   - *Production Workloads*: Relational databases (Amazon Aurora, RDS, OCI Base DB, Autonomous DB), Hardware Security Modules (HSMs), and sensitive payment data vaults.

### The Longest Prefix Match (LPM) Routing Engine
Cloud virtual routers evaluate route table rules using **Longest Prefix Match (LPM)**:
$$\text{More Specific Route (Longer Prefix /24)} > \text{Less Specific Route (Shorter Prefix /16)} > \text{Default Route (/0)}$$

If a route table has:
- `10.0.0.0/16` $\longrightarrow$ `local`
- `10.0.2.0/24` $\longrightarrow$ `tgw-attachment-12345`
- `0.0.0.0/0` $\longrightarrow$ `nat-gateway-98765`

A packet destined for `10.0.2.50` will **always** route to the Transit Gateway (`/24`), ignoring both the `/16` local route and the `/0` default route.

## 3. Mental Model
Think of 3-tier subnets as a medieval castle defense system:
- **Public Subnet** is the **Outer Courtyard & Drawbridge**. Anyone from the outside world can approach the gate. The guards (Load Balancers) verify visitor credentials and inspect baggage before letting anyone further.
- **Private Subnet** is the **Inner Castle Hall**. Servants (Application Servers) can send messengers out to the market (NAT Gateway) to buy supplies, but market traders cannot enter the inner hall.
- **Isolated Subnet** is the **Subterranean Royal Treasure Vault (Database)**. There is no door to the outside world. The only access path is an interior guarded staircase connecting directly to the castle throne room.

## 4. Architecture Diagram
```text
┌────────────────────────────────────────────────────────────────────────┐
│                        VPC / VCN (10.0.0.0/16)                         │
│                                                                        │
│   PUBLIC INGRESS TIER (Route Table: 0.0.0.0/0 ──> Internet Gateway)    │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ [Public Load Balancer: ALB / OCI LB]   [NAT Gateway]           │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │ Inbound Reverse Proxy Traffic      │
│                                   ▼                                    │
│   PRIVATE APPLICATION TIER (Route Table: 0.0.0.0/0 ──> NAT Gateway)    │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ [EC2 Autoscaling Group / OCI Instance Pool / K8s Pods]         │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │ Internal Private SQL Queries       │
│                                   ▼                                    │
│   ISOLATED DATA TIER (Route Table: NO 0.0.0.0/0 ROUTE!)                │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ [Aurora PostgreSQL Primary] ──► [Aurora Read Replica]          │   │
│   │ [OCI Base Database System]  ──► [Data Guard Standby]           │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS Amazon VPC:
- **Subnet Route Table Association**: By default, subnets attach to the VPC **Main Route Table**. Best practice: Never use the Main Route Table for workloads. Explicitly create and associate dedicated route tables per tier:
  - `rt_public` associated with public subnets.
  - `rt_private_az1`, `rt_private_az2` associated with private subnets (pointing to local AZ NAT Gateways).
  - `rt_isolated` associated with DB subnets (containing only the default local route `10.0.0.0/16 -> local`).
- **Implicit Local Route**: In AWS, the VPC CIDR block route (`10.0.0.0/16 -> local`) is created automatically in every route table and **cannot be modified, edited, or deleted** `[Doc: Amazon VPC Route Tables, checked 2026-09-03]`.
- **DB Subnet Groups**: Managed services like Amazon RDS require an `aws_db_subnet_group` spanning at least 2 Availability Zones. Binding this subnet group to isolated subnets guarantees that the RDS cluster cannot be reached from the internet, even if an engineer accidentally toggles the "Publicly Accessible" flag.

## 6. OCI Implementation
In Oracle Cloud Infrastructure VCN:
- **VCN Route Tables & Route Rules**: OCI route tables contain a list of `route_rules`. Each rule specifies:
  - `destination`: Destination CIDR or Service CIDR.
  - `destination_type`: `CIDR_BLOCK` or `SERVICE_CIDR_BLOCK`.
  - `network_entity_id`: Target gateway (IGW, NAT GW, Service GW, LPG, or DRG) `[Doc: OCI Route Tables, checked 2026-09-03]`.
- **Regional Subnets Simplify 3-Tier Architecture**:
  - In AWS, a 3-tier architecture across 3 AZs requires **9 subnets** (3 public, 3 private, 3 isolated) and **multiple route tables**.
  - In OCI, because subnets are Regional, the exact same 3-tier architecture requires **only 3 regional subnets**:
    1. `public_subnet`: Spans all ADs; associated with route table pointing `0.0.0.0/0` to Internet Gateway.
    2. `private_subnet`: Spans all ADs; associated with route table pointing `0.0.0.0/0` to Regional NAT Gateway.
    3. `isolated_db_subnet`: Spans all ADs; associated with route table containing zero external rules.
- **Prohibit Public IP Flag**: OCI subnets possess a native boolean attribute `prohibit_public_ip_on_vnic = true`. When enabled, the OCI control plane physically blocks any API call attempting to assign an ephemeral or reserved public IP to any VNIC created in that subnet.

## 7. Configuration
Comparing 3-tier route table configuration in Terraform across AWS and OCI:

### AWS 3-Tier Subnet Route Tables (Terraform)
```hcl
# 1. Public Route Table (IGW)
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = { Name = "public-rt" }
}

# 2. Private Route Table (NAT Gateway)
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat_gw.id
  }
  tags = { Name = "private-rt" }
}

# 3. Isolated Database Route Table (NO 0.0.0.0/0 ROUTE)
resource "aws_route_table" "isolated" {
  vpc_id = aws_vpc.main.id
  # Only implicit local route 10.0.0.0/16 -> local exists!
  tags = { Name = "isolated-db-rt" }
}
```

### OCI 3-Tier Subnet Route Tables (Terraform)
```hcl
# 1. Public Route Table (IGW)
resource "oci_core_route_table" "public_rt" {
  compartment_id = var.compartment_id
  vcn_id         = oci_core_vcn.main.id
  display_name   = "public-route-table"
  route_rules {
    destination       = "0.0.0.0/0"
    destination_type  = "CIDR_BLOCK"
    network_entity_id = oci_core_internet_gateway.igw.id
  }
}

# 2. Private Route Table (NAT Gateway)
resource "oci_core_route_table" "private_rt" {
  compartment_id = var.compartment_id
  vcn_id         = oci_core_vcn.main.id
  display_name   = "private-route-table"
  route_rules {
    destination       = "0.0.0.0/0"
    destination_type  = "CIDR_BLOCK"
    network_entity_id = oci_core_nat_gateway.nat_gw.id
  }
}

# 3. Isolated DB Route Table (EMPTY ROUTE RULES)
resource "oci_core_route_table" "isolated_rt" {
  compartment_id = var.compartment_id
  vcn_id         = oci_core_vcn.main.id
  display_name   = "isolated-db-route-table"
  # Zero route rules: completely isolated from external networks!
}
```

## 8. Data Flow
```text
Tier-to-Tier Request Traversal:
1. Client (Public Internet) ──> Hits Public ALB in Public Subnet
2. Public ALB terminates TLS, forwards reverse-proxy request to Private App IP (10.0.1.50)
3. App Server in Private Subnet processes request, queries Database in Isolated Subnet (10.0.2.100:5432)
   * Packet evaluated by VPC/VCN Router: Destination matches local VPC CIDR (10.0.0.0/16).
   * Packet delivered directly across private hypervisor overlay.
4. Database responds to App Server.
5. If App Server needs to verify credit card with Stripe:
   * Packet destined for public internet IP (54.187.159.182).
   * Route table matches 0.0.0.0/0 ──> Forwarded to NAT Gateway in Public Subnet.
   * NAT Gateway executes SNAT and transmits via Internet Gateway.
```

## 9. Security
- **Impossibility of Inbound Internet Ingress in Isolated Subnets**: Even if a developer accidentally assigns an application to listen on `0.0.0.0:80` without authentication on a database instance in an isolated subnet, the instance is **physically unreachable from the internet**. The virtual router possesses no gateway entity to route incoming or outgoing public packets.
- **Microsegmentation Between Tiers**: Pair subnet routing with security groups/NSGs. The Isolated Subnet must strictly allow inbound connections on port 5432/1521 originating from the Application Tier Security Group ID or NSG ID.

## 10. Reliability
- **Multi-AZ / Multi-AD Redundancy per Tier**:
  - In AWS, each tier must span at least 3 AZs: 3 public subnets, 3 private subnets, and 3 isolated subnets.
  - In OCI, each of the 3 regional subnets automatically protects against individual AD or Fault Domain failures.

## 11. Scaling
- **Subnet CIDR Sizing Strategy**:
  - *Public Ingress Tier*: Typically small (`/24` or `/25`). Hosts only load balancers, NAT gateways, and bastions.
  - *Private Application Tier*: Sized large (`/20` or `/19`). Hosts container pods, microservices, and autoscaling fleets.
  - *Isolated Data Tier*: Sized moderate (`/24` or `/23`). Hosts database primaries, replicas, and cache clusters.

## 12. Observability
- **Route Table Verification via CLI**:
  - AWS: `aws ec2 describe-route-tables --route-table-ids <rt-id>`.
  - OCI: `oci network route-table get --rt-id <rt-id>`.
- **VPC Flow Logs per Subnet**: Enable flow logs specifically on the isolated database subnets to audit all internal access attempts and verify zero external IP entries.

## 13. Cost
- Subnets and Route Tables are **free cloud primitives**.
- Sizing subnets appropriately avoids future IP migration costs. Placing databases in isolated subnets prevents accidental data egress bandwidth charges.

## 14. Failure Modes
- **The "Main Route Table" Contamination**: An administrator modifies the default VPC Main Route Table to route `0.0.0.0/0` to an Internet Gateway to test a public VM. Any existing private or database subnet that was not explicitly associated with a custom route table automatically falls back to the Main Route Table, instantly converting isolated subnets into public subnets.
- **The Blackhole Route Conflict**: A route table contains a route for `10.0.0.0/16 -> local`, and an administrator adds a more specific route `10.0.1.0/24 -> tgw-attachment-xxx`. Traffic destined for instances in `10.0.1.0/24` is routed away from the local VPC to the Transit Gateway due to Longest Prefix Match, severing internal communication.

## 15. Troubleshooting
When an instance in an isolated subnet cannot be reached by the application tier:
1. **Check Route Table Association**: Ensure the database subnet is associated with the isolated route table and has not been left orphaned.
2. **Verify Local CIDR**: Confirm that both the application subnet (`10.0.1.0/24`) and database subnet (`10.0.2.0/24`) fall within the VPC/VCN primary CIDR block (`10.0.0.0/16`).
3. **Inspect Security Group Referencing**: Verify that the database Security Group / NSG explicitly allows inbound traffic from the application Security Group / NSG.

## 16. Common Mistakes
- **Putting Databases in Private Subnets with NAT Routes**: Deploying databases into a private subnet that has a route to a NAT Gateway. While the database is not publicly reachable from the internet, it can initiate outbound connections to the internet, creating an exfiltration vector if SQL injection occurs. Place databases in **Isolated Subnets** with no internet route.
- **Forgetting Subnet Association**: Creating a custom route table in AWS or OCI but forgetting to associate it with the subnet. The subnet defaults to the VPC Main Route Table.

## 17. Trade-offs
| Subnet Tier | Inbound Access | Outbound Internet Access | Security Posture | Best Workloads |
| :--- | :--- | :--- | :--- | :--- |
| **Public Subnet** | Direct from Internet | Direct via IGW | Low isolation; High exposure | ALB, NLB, Bastions |
| **Private Subnet** | Reverse Proxy / Internal only | Outbound via NAT GW | Moderate isolation; Egress allowed | Microservices, K8s Pods |
| **Isolated Subnet**| Internal VPC only | None (Zero Internet) | Maximum isolation; Zero egress | Databases, HSMs, Caches |

## 18. Interview Questions
1. *Why should production relational databases be deployed into an 'Isolated Subnet' rather than a standard 'Private Subnet'? What security risk does a NAT Gateway introduce to a database?*
2. *Explain how Longest Prefix Match (LPM) works in cloud virtual route tables. If a route table has routes for `0.0.0.0/0`, `10.0.0.0/16`, and `10.0.2.0/24`, which route handles traffic destined for `10.0.2.15`?*
3. *How does OCI's `prohibit_public_ip_on_vnic` attribute provide a stronger security guarantee than AWS VPC subnet configurations?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "Deploying production relational databases into an **Isolated Subnet** (a subnet whose route table contains zero external routes to an Internet Gateway or NAT Gateway) rather than a standard Private Subnet is a core defense-in-depth requirement:
>
> 1. **The Exfiltration Risk of NAT Gateways**:
>    - In a standard Private Subnet, the route table contains `0.0.0.0/0 -> NAT Gateway`. While external attackers cannot initiate inbound connections to the database, the database instance **can initiate outbound connections to the internet**.
>    - If an attacker compromises the application tier and executes a remote code execution (RCE) or advanced SQL injection vulnerability (e.g., PostgreSQL `COPY TO PROGRAM` or Oracle `UTL_HTTP`), the compromised database can establish an outbound reverse shell or exfiltrate customer data directly to an attacker-controlled external command-and-control server.
> 2. **The Isolated Subnet Guarantee**:
>    - In an Isolated Subnet, the route table contains **no default route**. Packets destined for any public IP address are physically dropped by the virtual router at the hypervisor layer.
>    - Even if an attacker obtains root privileges on the database operating system, they cannot exfiltrate data directly to the internet.
> 3. **Database Maintenance in Isolated Subnets**:
>    - Database engine patching and backups do not require internet access: cloud providers manage database binaries internally, and backups are streamed directly to Amazon S3 via VPC Gateway Endpoints or to OCI Object Storage via Service Gateways over the internal cloud backbone."

## 20. Hands-on Exercise
**Objective**: Build a 3-tier route table topology in Terraform and verify routing isolation.

### Verification Steps
1. Deploy 3 subnets in a VPC/VCN: Public (`10.0.1.0/24`), Private (`10.0.2.0/24`), and Isolated (`10.0.3.0/24`).
2. Attach Public RT to Public Subnet (route to IGW).
3. Attach Private RT to Private Subnet (route to NAT GW).
4. Attach Isolated RT to Isolated Subnet (zero external routes).
5. Launch an instance in the Isolated Subnet. Connect via SSM / OCI Bastion.
6. Run `curl -I https://google.com`: connection must immediately fail with `No route to host`, verifying that the isolated tier cannot reach the internet.
