# Lab 01: Dual-Cloud VPC & VCN Topology Implementation

---

## 1. Goal & Core Concept

The objective of this hands-on lab is to architect, deploy, and statically validate an enterprise-grade, highly available 3-tier network topology across both **Amazon Web Services (AWS)** and **Oracle Cloud Infrastructure (OCI)**.

### Core Architectural Concepts Tested
- **Network Boundaries**: AWS Virtual Private Cloud (VPC) vs. OCI Virtual Cloud Network (VCN) address space allocation.
- **Subnet Tiers**: Segregation of Public Ingress (ALB / OCI LB), Private Compute (Microservices / Worker VMs), and Isolated Data (Databases / Cache) tiers across multiple failure domains (3 AZs / 3 Fault Domains).
- **Gateway Mechanics**:
  - AWS Internet Gateway (IGW) and NAT Gateway vs. OCI Internet Gateway (IGW), NAT Gateway (NGW), and Service Gateway (SGW).
- **Stateful vs. Stateless Filtering**: AWS Security Groups (stateful) and Network ACLs (stateless) vs. OCI Network Security Groups (NSGs, stateful) and Security Lists (stateful/stateless).

---

## 2. Cost Warning & Resource Estimate

> [!NOTE]
> **ESTIMATED HOURLY RUN COST: `< $0.05 / hour`**
>
> | Cloud Provider | Resource Component | Quantity | Hourly Rate | Run Cost (2 Hours) |
> | :--- | :--- | :---: | :---: | :---: |
> | **AWS** | VPC, Internet Gateway, Subnets, Route Tables | 1 VPC | $0.00 (Free) | $0.00 |
> | **AWS** | NAT Gateway (Single-AZ for lab testing) `[Doc: VPC Pricing, checked 2026]` | 1 | $0.045 / hr | $0.09 |
> | **OCI** | VCN, Subnets, Route Tables, NSGs | 1 VCN | $0.00 (Free) | $0.00 |
> | **OCI** | OCI NAT Gateway & Service Gateway `[Doc: OCI Networking, checked 2026]` | 1 NGW + 1 SGW | $0.00 (Free) | $0.00 |
>
> *Total Estimated Lab Cost for a 2-hour session: **$0.09 USD**.*
> *In No-Live-Account Fallback Mode, cost is **$0.00**.*

---

## 3. Architecture Diagram

```
========================================================================================================================
                                   DUAL-CLOUD 3-TIER NETWORK TOPOLOGY
========================================================================================================================

  AWS VPC: 10.0.0.0/16                                      OCI VCN: 10.1.0.0/16
  ┌──────────────────────────────────────────────────┐      ┌──────────────────────────────────────────────────┐
  │ [ Internet Gateway (IGW) ]                       │      │ [ Internet Gateway (IGW) ]                       │
  │                     │                            │      │                     │                            │
  │                     ▼                            │      │                     ▼                            │
  │ PUBLIC SUBNETS (Ingress Tier - 10.0.0.0/24)      │      │ PUBLIC SUBNETS (Ingress Tier - 10.1.0.0/24)      │
  │ - Default route: 0.0.0.0/0 -> igw-xxxx           │      │ - Default route: 0.0.0.0/0 -> igw-xxxx           │
  │ - Hosts: AWS NAT Gateway (10.0.0.10)             │      │ - Hosts: Public Flexible LB VIP                  │
  │                     │                            │      │                                                  │
  │ ────────────────────┼─────────────────────────── │      │ ──────────────────────────────────────────────── │
  │                     ▼                            │      │ [ OCI NAT Gateway (NGW) ] [ OCI Service GW (SGW)]│
  │ PRIVATE COMPUTE SUBNETS (10.0.10.0/24)           │      │                     │                            │
  │ - Default route: 0.0.0.0/0 -> nat-xxxx           │      │ PRIVATE COMPUTE SUBNETS (10.1.10.0/24)           │
  │ - Stateful SG: Allow ingress port 8080 from ALB  │      │ - Default route: 0.0.0.0/0 -> nat-gw             │
  │                     │                            │      │ - OCI Service route: all-services -> service-gw  │
  │ ────────────────────┼─────────────────────────── │      │ ──────────────────────────────────────────────── │
  │                     ▼ (DB Port 5432 Only)        │      │                     ▼ (DB Port 1521/5432 Only)   │
  │ ISOLATED DATA SUBNETS (10.0.20.0/24)             │      │ ISOLATED DATA SUBNETS (10.1.20.0/24)             │
  │ - NO route to Internet or NAT Gateway            │      │ - NO route to Internet Gateway or NAT Gateway    │
  │ - Accessible only via Private Compute SG         │      │ - Accessible only via Private Compute NSG        │
  └──────────────────────────────────────────────────┘      └──────────────────────────────────────────────────┘
========================================================================================================================
```

---

## 4. Prerequisites

1. **CLI Tools**:
   - `terraform` (v1.8.0 or later).
   - `awscli` (v2.15+ configured with `AWS_DEFAULT_REGION="us-east-1"`).
   - `oci-cli` (v3.37+ configured with user tenancy OCID).
2. **IAM Permissions**:
   - AWS: `AmazonVPCFullAccess` or granular policy allowing `ec2:*Vpc*`, `ec2:*Subnet*`, `ec2:*Gateway*`.
   - OCI: `manage virtual-network-family in compartment <lab-compartment>`.

---

## 5. Infrastructure Code (Terraform HCL)

### 5.1 AWS Network Implementation (`aws_network.tf`)

```hcl
# AWS 3-Tier VPC Architecture
terraform {
  required_version = ">= 1.8.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.50"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# 1. VPC Core
resource "aws_vpc" "lab_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "lab01-enterprise-vpc"
    Environment = "lab"
  }
}

# 2. Gateways
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.lab_vpc.id
  tags   = { Name = "lab01-igw" }
}

resource "aws_eip" "nat_eip" {
  domain = "vpc"
  tags   = { Name = "lab01-nat-eip" }
}

resource "aws_nat_gateway" "nat_gw" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public_subnet.id
  tags          = { Name = "lab01-nat-gw" }

  depends_on = [aws_internet_gateway.igw]
}

# 3. Subnets
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.lab_vpc.id
  cidr_block              = "10.0.0.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
  tags                    = { Name = "lab01-public-subnet" }
}

resource "aws_subnet" "private_compute_subnet" {
  vpc_id            = aws_vpc.lab_vpc.id
  cidr_block        = "10.0.10.0/24"
  availability_zone = "us-east-1a"
  tags              = { Name = "lab01-private-compute-subnet" }
}

resource "aws_subnet" "isolated_data_subnet" {
  vpc_id            = aws_vpc.lab_vpc.id
  cidr_block        = "10.0.20.0/24"
  availability_zone = "us-east-1a"
  tags              = { Name = "lab01-isolated-data-subnet" }
}

# 4. Route Tables & Associations
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.lab_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = { Name = "lab01-public-rt" }
}

resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}

resource "aws_route_table" "private_compute_rt" {
  vpc_id = aws_vpc.lab_vpc.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat_gw.id
  }

  tags = { Name = "lab01-private-compute-rt" }
}

resource "aws_route_table_association" "private_compute_assoc" {
  subnet_id      = aws_subnet.private_compute_subnet.id
  route_table_id = aws_route_table.private_compute_rt.id
}

# Isolated data tier has a local-only route table (no default 0.0.0.0/0 route)
resource "aws_route_table" "isolated_data_rt" {
  vpc_id = aws_vpc.lab_vpc.id
  tags   = { Name = "lab01-isolated-data-rt" }
}

resource "aws_route_table_association" "isolated_data_assoc" {
  subnet_id      = aws_subnet.isolated_data_subnet.id
  route_table_id = aws_route_table.isolated_data_rt.id
}
```

### 5.2 OCI Network Implementation (`oci_network.tf`)

```hcl
# OCI 3-Tier VCN Architecture
terraform {
  required_providers {
    oci = {
      source  = "oracle/oci"
      version = "~> 5.40"
    }
  }
}

variable "compartment_ocid" {
  type        = string
  description = "Target OCI Compartment OCID"
}

# 1. VCN Core
resource "oci_core_vcn" "lab_vcn" {
  compartment_id = var.compartment_ocid
  cidr_blocks    = ["10.1.0.0/16"]
  display_name   = "lab01-enterprise-vcn"
  dns_label      = "lab01vcn"
}

# 2. Gateways (IGW, NAT GW, Service GW)
resource "oci_core_internet_gateway" "igw" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.lab_vcn.id
  display_name   = "lab01-igw"
  enabled        = true
}

resource "oci_core_nat_gateway" "nat_gw" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.lab_vcn.id
  display_name   = "lab01-nat-gw"
}

data "oci_core_services" "all_services" {
  filter {
    name   = "name"
    values = ["All .* Services In Oracle Services Network"]
    regex  = true
  }
}

resource "oci_core_service_gateway" "service_gw" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.lab_vcn.id
  display_name   = "lab01-service-gw"

  services {
    service_id = data.oci_core_services.all_services.services[0].id
  }
}

# 3. Route Tables
resource "oci_core_route_table" "public_rt" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.lab_vcn.id
  display_name   = "lab01-public-rt"

  route_rules {
    destination       = "0.0.0.0/0"
    destination_type  = "CIDR_BLOCK"
    network_entity_id = oci_core_internet_gateway.igw.id
  }
}

resource "oci_core_route_table" "private_compute_rt" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.lab_vcn.id
  display_name   = "lab01-private-compute-rt"

  # Egress to Internet via NAT Gateway
  route_rules {
    destination       = "0.0.0.0/0"
    destination_type  = "CIDR_BLOCK"
    network_entity_id = oci_core_nat_gateway.nat_gw.id
  }

  # Direct Egress to OCI Object Storage/Yum via Service Gateway (No NAT charge)
  route_rules {
    destination       = data.oci_core_services.all_services.services[0].cidr_block
    destination_type  = "SERVICE_CIDR_BLOCK"
    network_entity_id = oci_core_service_gateway.service_gw.id
  }
}

# 4. Subnets
resource "oci_core_subnet" "public_subnet" {
  compartment_id             = var.compartment_ocid
  vcn_id                     = oci_core_vcn.lab_vcn.id
  cidr_block                 = "10.1.0.0/24"
  display_name               = "lab01-public-subnet"
  route_table_id             = oci_core_route_table.public_rt.id
  prohibit_public_ip_on_vnic = false
}

resource "oci_core_subnet" "private_compute_subnet" {
  compartment_id             = var.compartment_ocid
  vcn_id                     = oci_core_vcn.lab_vcn.id
  cidr_block                 = "10.1.10.0/24"
  display_name               = "lab01-private-compute-subnet"
  route_table_id             = oci_core_route_table.private_compute_rt.id
  prohibit_public_ip_on_vnic = true
}

resource "oci_core_subnet" "isolated_data_subnet" {
  compartment_id             = var.compartment_ocid
  vcn_id                     = oci_core_vcn.lab_vcn.id
  cidr_block                 = "10.1.20.0/24"
  display_name               = "lab01-isolated-data-subnet"
  prohibit_public_ip_on_vnic = true
}
```

---

## 6. Step-by-Step Deployment Guide

```bash
# 1. Initialize working directory without live credentials
terraform init -backend=false

# 2. Format and validate syntax
terraform fmt -check
terraform validate

# 3. Generate execution plan (simulated dry-run)
terraform plan -out=tfplan.binary
```

---

## 7. Expected Validation Results

```text
[Statically validated — not applied to a live account]

Execution Plan Summary:
Plan: 13 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + aws_vpc_id               = "vpc-0a1b2c3d4e5f67890"
  + aws_nat_gateway_ip       = "52.23.45.67"
  + oci_vcn_id               = "ocid1.vcn.oc1.iad.amaaaaaaxxx"
  + oci_service_gateway_id   = "ocid1.servicegateway.oc1.iad.amaaaaaayyy"

Route Table Propagation Validation:
- AWS Route Table 'lab01-private-compute-rt':
    0.0.0.0/0 -> nat-0a9b8c7d6e5f
    10.0.0.0/16 -> local (implicit)
- OCI Route Table 'lab01-private-compute-rt':
    0.0.0.0/0 -> nat-gw-ocid
    all-iad-services-in-oracle-services-network -> service-gw-ocid
```

---

## 8. Failure Injection Drill: Blackholing the NAT Route

### The Scenario
A junior engineer modifies the private route table, accidentally removing the `0.0.0.0/0 -> nat-gw` route. Private worker instances can no longer fetch security updates or call external SaaS APIs.

### The Injection
```bash
# AWS: Replace NAT route with non-existent gateway ID or drop it
aws ec2 delete-route --route-table-id rtb-0123456789abcdef0 --destination-cidr-block 0.0.0.0/0
```

### Manifested Symptoms
- Worker pods on EC2/OCI fail external DNS resolution for external APIs.
- Outbound curl attempts hang until TCP connection timeout (`Connection timed out after 30000ms`).
- Inbound traffic from ALB still succeeds because ALB forwards traffic inside the VPC/VCN private subnet boundaries.

---

## 9. Debugging Walkthrough

```text
====================================================================================================
                        STEP-BY-STEP ROOT CAUSE INVESTIGATION
====================================================================================================

Step 1: Check Ingress vs. Egress Health
  - Ingress curl to ALB responds with HTTP 200 (Load Balancer to EC2 communication is healthy).
  - Outbound curl from EC2 to https://api.github.com times out.
  - Deduction: The issue is strictly outbound egress routing.

Step 2: Inspect VPC / VCN Route Tables
  $ aws ec2 describe-route-tables --route-table-ids rtb-0123456789abcdef0       --query "RouteTables[0].Routes"
  Output:
  [
    {"DestinationCidrBlock": "10.0.0.0/16", "GatewayId": "local", "State": "active"}
  ]
  Root Cause Identified: Missing default route (0.0.0.0/0) pointing to the NAT Gateway!

Step 3: Remediate the Defect
  $ aws ec2 create-route       --route-table-id rtb-0123456789abcdef0       --destination-cidr-block 0.0.0.0/0       --nat-gateway-id nat-0a9b8c7d6e5f
  Result: Immediate restoration of outbound connectivity (< 1 second).
====================================================================================================
```

---

## 10. Teardown & Verification

```bash
# Destroy all provisioned lab resources
terraform destroy -auto-approve

# Verification Check: Confirm zero orphaned NAT gateways or Elastic IPs
aws ec2 describe-nat-gateways --filter "Name=state,Values=available,pending"
aws ec2 describe-addresses --query "Addresses[?AssociationId==null]"
oci network nat-gateway list --compartment-id ${COMPARTMENT_OCID}
```

---

## 11. Senior Interview Defense & Takeaways

> **Interviewer**: *"Why does OCI provide a Service Gateway (SGW) in addition to a NAT Gateway, whereas AWS recommends using VPC Endpoints?"*

**Candidate Defense**:
*"OCI Service Gateway provides private, non-metered routing directly into Oracle's Services Network (Object Storage, Autonomous DB, Container Registry) without traversing the public internet or incurring NAT Gateway data processing fees.*

*In AWS, routing traffic to S3 or DynamoDB privately is achieved using Gateway VPC Endpoints (which modify route tables and are completely free), whereas routing to other services requires Interface VPC Endpoints (AWS PrivateLink), which incur an hourly charge plus per-GB data processing fees. In both clouds, routing high-volume storage traffic through a NAT Gateway is a major architectural anti-pattern that inflates cloud egress and data processing bills."*
