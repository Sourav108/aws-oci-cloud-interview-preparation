# 02. NAT Gateways, Service Gateways & VPC Endpoints

## 1. Problem
One of the most common and costly blunders in cloud architecture is routing all internal cloud service API traffic through public NAT Gateways. When instances in private subnets upload terabytes of logs to Amazon S3, query DynamoDB tables, or push database backups to OCI Object Storage via a standard NAT Gateway, companies incur thousands of dollars in redundant data processing fees, degrade network throughput, and unnecessarily expose internal cloud API calls to the public internet edge. Understanding how to bypass NAT Gateways using **VPC Endpoints** in AWS and **Service Gateways** in OCI is a fundamental competency for senior infrastructure engineers.

## 2. Cloud Concept
### Outbound Internet Access: NAT Gateways
- **NAT Gateway**: A software-defined Network Address Translation service that executes Source NAT (SNAT). Instances in private subnets forward internet-bound packets to the NAT Gateway, which translates the private IP into its public Elastic IP, enabling outbound communication while blocking inbound connections from the internet.
- **The AWS NAT Gateway**: A **zonal primitive**. A single NAT Gateway resides in one Availability Zone. If that AZ experiences an outage, the NAT Gateway fails. High availability mandates running 1 NAT Gateway per AZ.
- **The OCI NAT Gateway**: A **regional primitive**. A single OCI NAT Gateway is distributed across the physical network fabric and serves all Availability Domains and Fault Domains with automated platform resilience.

### Private Cloud Backbone Access: Endpoints & Service Gateways
Instead of traversing a NAT Gateway to reach public cloud service endpoints (e.g., S3, DynamoDB, OCI Object Storage, Vault), cloud providers allow routing traffic directly over their internal private optical backbones:

- **AWS VPC Gateway Endpoints**:
  - Attached directly to VPC Route Tables.
  - Supported strictly for **Amazon S3** and **Amazon DynamoDB**.
  - **100% Free**: Zero hourly fees, zero data processing fees `[Doc: AWS PrivateLink Pricing, checked 2026-09-03]`.
- **AWS Interface VPC Endpoints (AWS PrivateLink)**:
  - Provisions an Elastic Network Interface (ENI) with a private IP inside your subnet.
  - Supports virtually all other AWS services (KMS, Secrets Manager, CloudWatch, ECR, SQS) and custom microservices.
  - *Billing*: Incurs an hourly fee per AZ plus a per-GB data processing fee.
- **OCI Service Gateway**:
  - An architectural masterstroke in OCI: A single virtual gateway attached to a VCN that routes traffic directly to the **Oracle Services Network (OSN)**.
  - Enables private subnets to access **ALL OCI services in the region** (Object Storage, Autonomous DB, KMS Vault, OCI Registry) without public IPs, without NAT Gateways, and with **zero data transfer or processing charges** `[Doc: OCI Service Gateway Overview, checked 2026-09-03]`.

## 3. Mental Model
Think of cloud service access routes as traveling to a company subsidiary:
- **Routing via NAT Gateway** is driving your car onto the public interstate highway, paying highway toll booths (NAT data processing fees), driving through public traffic, and turning back into the company's private branch office.
- **Routing via VPC Endpoint / Service Gateway** is walking through an underground private tunnel connecting the two buildings. It is completely private, immune to highway traffic jams, and charges zero toll fees.

## 4. Architecture Diagram
```text
THE EXPENSIVE, SUB-OPTIMAL NAT PATH:
[Private App Server (10.0.1.5)] ──> [NAT Gateway] ──> [Internet Gateway] ──> [Public Internet] ──> [Amazon S3 / OCI Object Storage]
* Billed: $0.045/GB NAT Data Processing Fees!
* Packets traverse internet edge routers.

THE OPTIMIZED, PRIVATE BACKBONE PATH:
AWS:
[Private App Server (10.0.1.5)] ──> [VPC Route Table: pl-xxxx ──> Gateway Endpoint] ──► [Amazon S3 (Private Backbone: $0.00)]
[Private App Server (10.0.1.5)] ──> [Interface Endpoint ENI: 10.0.1.50 (PrivateLink)] ─► [AWS Secrets Manager / KMS]

OCI:
[Private App Server (10.0.1.5)] ──> [VCN Route Table: All Services ──> Service GW] ──► [OCI Object Storage / Vault ($0.00)]
```

## 5. AWS Implementation
In AWS:
- **Gateway Endpoints (S3 & DynamoDB)**:
  - When created, AWS creates a **Prefix List** (e.g., `pl-63a5400a` representing all public S3 IP CIDR blocks in the region).
  - AWS injects a route into your selected route tables: `Destination: pl-63a5400a` $\longrightarrow$ `Target: vpce-xxxx`.
  - Instances in the subnet automatically route all S3 traffic over the internal AWS fabric.
  - Supports **VPC Endpoint Policies**: fine-grained IAM JSON policies attached to the endpoint that restrict which specific S3 buckets can be accessed (preventing data exfiltration to unauthorized personal AWS accounts).
- **Interface Endpoints (AWS PrivateLink)**:
  - Allocates a private IP address from your subnet's CIDR block.
  - Leverages **Private DNS**: AWS overrides public DNS resolution so that calls to `secretsmanager.us-east-1.amazonaws.com` resolve locally to the private ENI IP (`10.0.1.50`) instead of the public internet IP.
  - Controlled by standard Security Groups attached to the endpoint ENI.

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **The OCI Service Gateway**:
  - When creating a Service Gateway, you select the service label:
    - `All <Region> Services in Oracle Services Network`: Covers Object Storage, Autonomous DB, Streaming, Vault, OCI Logging, and Registry.
    - `OCI <Region> Object Storage Only`: Restricts access strictly to Object Storage.
  - In your private route table, add a route rule:
    - `Destination Type`: `SERVICE_CIDR_BLOCK`.
    - `Destination`: `All <Region> Services In Oracle Services Network`.
    - `Network Entity`: Service Gateway OCID.
- **Zero Configuration Complexity**:
  - In AWS, accessing 10 services privately requires provisioning 10 separate Interface Endpoints across 3 AZs ($10 \times 3 = 30\text{ ENIs}$, costing over $\$200/\text{month}$ in baseline hourly fees).
  - In OCI, a single **Service Gateway** provides private access to **every native service** with zero hourly charges and zero per-GB fees.
- **OCI Private Endpoints**:
  - For customer-hosted applications or managed databases, OCI supports Private Endpoints (VNIC in a customer subnet) allowing private access to Autonomous Database or external SaaS services.

## 7. Configuration
Comparing private service access configuration in Terraform across AWS and OCI:

### AWS S3 Gateway Endpoint (Terraform)
```hcl
# Free S3 Gateway Endpoint attached to private route tables
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = var.vpc_id
  service_name      = "com.amazonaws.us-east-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [var.private_route_table_id]

  # Endpoint Policy restricting access strictly to corporate bucket
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AllowOnlyCorporateBucket"
        Effect    = "Allow"
        Principal = "*"
        Action    = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
        Resource = [
          "arn:aws:s3:::my-corp-data-bucket",
          "arn:aws:s3:::my-corp-data-bucket/*"
        ]
      }
    ]
  })
  tags = { Name = "s3-gateway-endpoint" }
}
```

### OCI Service Gateway (Terraform)
```hcl
# Fetch all services CIDR label for the region
data "oci_core_services" "all_services" {
  filter {
    name   = "name"
    values = [".*All.*Services.*"]
    regex  = true
  }
}

# Create OCI Service Gateway
resource "oci_core_service_gateway" "sgw" {
  compartment_id = var.compartment_id
  vcn_id         = var.vcn_id
  display_name   = "central-service-gateway"

  services {
    service_id = data.oci_core_services.all_services.services[0].id
  }
}

# Add Route Rule to Private Subnet Route Table
resource "oci_core_route_table" "private_rt" {
  compartment_id = var.compartment_id
  vcn_id         = var.vcn_id
  display_name   = "private-route-table"

  # Route internal Oracle services to Service Gateway (Zero NAT fees!)
  route_rules {
    destination       = data.oci_core_services.all_services.services[0].cidr_block
    destination_type  = "SERVICE_CIDR_BLOCK"
    network_entity_id = oci_core_service_gateway.sgw.id
  }

  # Route general internet egress to NAT Gateway
  route_rules {
    destination       = "0.0.0.0/0"
    destination_type  = "CIDR_BLOCK"
    network_entity_id = var.nat_gateway_id
  }
}
```

## 8. Data Flow
```text
Private S3 / Object Storage Upload Packet Flow:
1. Application server (10.0.1.5) executes PUT s3.us-east-1.amazonaws.com
2. DNS resolves to public S3 IP (52.216.142.100)
3. Kernel checks Route Table:
   * 52.216.142.100 matches Prefix List pl-63a5400a (/20)
   * Longest Prefix Match selects Gateway Endpoint OVER default route 0.0.0.0/0 (/0)
4. Hypervisor intercepts packet:
   * Bypasses NAT Gateway completely!
   * Transmits directly across AWS private fiber to S3 storage node.
5. Zero NAT processing charges incurred; zero public internet exposure.
```

## 9. Security
- **Data Exfiltration Prevention via Endpoint Policies**:
  - Without an endpoint policy, a compromised EC2 instance can use the S3 Gateway Endpoint to exfiltrate gigabytes of sensitive customer records to an attacker-controlled external AWS account's S3 bucket.
  - Applying an **Endpoint Policy** that permits `s3:*` actions **strictly on company-owned bucket ARNs** blocks uploads to foreign buckets at the network layer, even if the application has valid AWS credentials.
- **PrivateLink Security Groups**: Interface endpoints in AWS have Security Groups attached, allowing you to restrict which specific microservice IP addresses can call the AWS KMS or Secrets Manager API.

## 10. Reliability
- **Eliminating NAT Gateway Bottlenecks**:
  - Standard NAT Gateways support up to 100 Gbps, but burst spikes can cause packet drops.
  - S3 Gateway Endpoints and OCI Service Gateways have **no throughput limits**. They scale horizontally across the cloud provider's core fabric, eliminating a major potential availability bottleneck during data-heavy operations.

## 11. Scaling
- **NAT Gateway Ephemeral Port Protection**:
  - High-frequency calls to DynamoDB or S3 through a NAT Gateway consume source ports rapidly.
  - Moving S3 and DynamoDB to Gateway Endpoints removes millions of packets from the NAT Gateway, preserving ephemeral ports for external third-party API calls.

## 12. Observability
- **Monitoring NAT Gateway vs. Endpoint Traffic**:
  - AWS CloudWatch: Monitor `BytesInFromDestination` and `BytesOutToDestination` on NAT Gateways. A sudden drop in bytes processed after deploying an S3 Gateway Endpoint proves successful traffic migration.
  - In VPC Flow Logs, packets routed to an S3 Gateway Endpoint display the target S3 IP and are logged as `ACCEPT`.

## 13. Cost Economics: The NAT Elimination Math
Consider an enterprise platform backing up 100 TB of database snapshots and video files per month to Amazon S3:

### Scenario A: Unoptimized (Routing to S3 via NAT Gateway)
- NAT Gateway Hourly Fee: $3\text{ AZs} \times \$0.045/\text{hr} \times 730\text{ hrs} = \$98.55/\text{month}$.
- NAT Data Processing Fee: $100\text{ TB} \times 1,000\text{ GB/TB} \times \$0.045/\text{GB} = \mathbf{\$4,500.00/\text{month}}$.
- Total Monthly Cost: **\$4,598.55**.

### Scenario B: Optimized (Deploying S3 VPC Gateway Endpoint)
- S3 Gateway Endpoint Hourly Fee: **\$0.00 (Free)**.
- S3 Gateway Data Processing Fee: **\$0.00 (Free)**.
- Total Monthly Cost for S3 Traffic: **\$0.00**.
- **Net Annual Savings: \$54,000.00**.

In OCI, deploying an **OCI Service Gateway** delivers the exact same 100% cost elimination for all Oracle cloud service traffic.

## 14. Failure Modes
- **The Interface Endpoint Subnet IP Depletion**: An organization deploys Interface Endpoints for 15 AWS services across 3 AZs. Each endpoint consumes a private IP from the subnet. If subnets are sized at `/28` (11 usable IPs), the endpoints exhaust all available IPs, preventing new EC2 instances or container pods from launching.
- **The Private DNS Split-Brain Blackhole**: When enabling Private DNS on an AWS Interface Endpoint for Amazon ECR, AWS overrides the public DNS record `api.ecr.us-east-1.amazonaws.com` to point to the private ENI IP. If the Security Group on the Interface Endpoint ENI blocks port 443 from the Kubernetes node security group, all `docker pull` commands across the entire cluster immediately fail.

## 15. Troubleshooting
When traffic to S3 or OCI Object Storage fails or times out:
1. **Verify Route Table Association**: Check if the subnet route table actually contains the S3 Prefix List (`pl-xxxx`) or OCI Service CIDR rule.
2. **Inspect Endpoint Policy**: Does the VPC Endpoint Policy explicitly permit the requested action (e.g., `s3:GetObject`) on the target bucket?
3. **Verify Security Group Rules on Interface Endpoints**: If using AWS PrivateLink or OCI Private Endpoints, verify that inbound port 443 is allowed from the calling instance's IP.

## 16. Common Mistakes
- **Paying for Interface Endpoints for S3 When Gateway Endpoints Are Free**: AWS offers both Gateway Endpoints (free) and Interface Endpoints (paid) for S3. Using Interface Endpoints for standard S3 access incurs unnecessary hourly and data processing fees. Use **Gateway Endpoints** unless on-premises networks need to access S3 over Direct Connect.
- **Routing All Traffic to NAT Gateway in OCI**: Deploying an OCI VCN and routing `0.0.0.0/0` to a NAT Gateway without adding a Service Gateway route. OCI users waste network hops and latency by failing to use the free Service Gateway.

## 17. Trade-offs
| Gateway Primitive | Scope | Hourly Cost | Data Fee | Best Use Case |
| :--- | :--- | :---: | :---: | :--- |
| **AWS NAT Gateway** | Public Internet | \$0.045/hr | \$0.045/GB | Outbound calls to external SaaS APIs (Stripe, Twilio) |
| **OCI NAT Gateway** | Public Internet | Free | Egress rates | Outbound calls to external SaaS APIs |
| **AWS S3/DynamoDB Gateway Endpoint**| S3 & DynamoDB | **Free** | **Free** | All internal S3 and DynamoDB data traffic |
| **AWS PrivateLink (Interface Endpoint)**| AWS & Custom APIs | \$0.01/hr | \$0.01/GB | KMS, Secrets Manager, ECR, B2B SaaS endpoints |
| **OCI Service Gateway** | All Regional OCI Services | **Free** | **Free** | Object Storage, Autonomous DB, Vault, OCI APIs |

## 18. Interview Questions
1. *You notice your monthly AWS bill includes \$6,000 in 'NAT Gateway Data Processing' charges. How do you investigate the source of this traffic, and what architectural change permanently eliminates this cost?*
2. *Explain the architectural and operational differences between an AWS Gateway VPC Endpoint and an Interface VPC Endpoint (PrivateLink).*
3. *How does OCI's Service Gateway simplify private cloud architecture compared to AWS's endpoint model?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "A \$6,000 monthly charge in AWS NAT Gateway Data Processing indicates that approximately **133 TB of data** ($6,000 / \$0.045/\text{GB}$) is flowing through NAT Gateways from private subnets to external IP destinations.
>
> 1. **Investigation Protocol**:
>    - I will enable **VPC Flow Logs** on the private subnets and query CloudWatch Logs Insights.
>    - Grouping by destination IP and summing `bytes` processed, we typically find that 80–90% of high-volume NAT traffic is destined for **Amazon S3** (application logs, database backups, media assets) or **Amazon DynamoDB**.
> 2. **The Architectural Remediation**:
>    - I will provision an **Amazon S3 Gateway VPC Endpoint** and a **DynamoDB Gateway VPC Endpoint** in the VPC.
>    - I will associate these endpoints with all private subnet route tables.
> 3. **The Mechanical Effect**:
>    - AWS automatically injects routes for the regional S3 and DynamoDB prefix lists into the private route tables.
>    - Because these routes have `/20` or `/24` prefix lengths, Longest Prefix Match ensures that instances automatically divert S3/DynamoDB traffic away from the default route (`0.0.0.0/0 -> NAT Gateway`) to the Gateway Endpoint.
>    - The traffic now flows over AWS's internal private fiber fabric. Because Gateway Endpoints have **zero hourly fees and zero data processing fees**, that \$6,000 line item drops to **\$0.00** on the next billing cycle, while simultaneously improving upload throughput and reducing latency."

## 20. Hands-on Exercise
**Objective**: Deploy an S3 Gateway Endpoint in Terraform, verify routing table injection, and measure latency.

### Verification Steps
1. Deploy an EC2 instance in a private subnet with a route to a NAT Gateway.
2. Execute an S3 download via AWS CLI: `aws s3 cp s3://<bucket>/100MB.bin .` and observe download speed.
3. Apply Terraform provisioning `aws_vpc_endpoint` with type `Gateway`.
4. Inspect route table: confirm `pl-xxxx` (S3 Prefix List) target is `vpce-xxxx`.
5. Repeat download: observe increased throughput and verify in VPC Flow Logs that traffic bypasses the NAT Gateway.
