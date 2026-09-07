# Reference Project 06: Enterprise Multi-Account & Multi-Compartment Landing Zone

---

## 1. Executive Summary & Architecture Overview

This production reference architecture implements an enterprise-grade, multi-account (AWS) and multi-compartment (OCI) cloud foundation landing zone designed to enforce organizational governance, security guardrails, centralized hub-and-spoke networking, identity federation, and cost attribution across hundreds of application teams.

Key Architectural Capabilities:
- **Hierarchical Governance Structure**: Organizes cloud resources using AWS Organizations / Control Tower Organizational Units (OUs) and OCI Nested Compartment Hierarchies.
- **Centralized Hub-and-Spoke Networking**: Implements AWS Transit Gateway (TGW) and OCI Dynamic Routing Gateway (DRG v2) with inspection firewalls routing all inter-VPC/VCN and on-premises traffic.
- **Enterprise Identity & SSO Federation**: Centralized identity federation via AWS IAM Identity Center (SAML 2.0 / SCIM) and OCI IAM Identity Domains with Okta / Azure AD.
- **Automated Security Guardrails**: Enforces Service Control Policies (SCPs) on AWS and Compartment IAM Policies on OCI to prevent security violations before they occur.

---

## 2. Dual-Cloud System Topology

```
========================================================================================================================
                          ENTERPRISE LANDING ZONE HIERARCHICAL TOPOLOGY
========================================================================================================================

  AWS ORGANIZATIONS / OCI ROOT TENANCY
  │
  ├── [ CORE / FOUNDATION OU / COMPARTMENT ]
  │   ├── Security Account / Security Compartment
  │   │   - Centralized AWS Security Hub / OCI Cloud Guard
  │   │   - GuardDuty / OCI Vulnerability Scanning Service
  │   │   - Security Operations IAM Roles & Auditing
  │   │
  │   ├── Log Archive Account / Logging Compartment
  │   │   - Centralized CloudTrail / OCI Audit Log Bucket
  │   │   - Immutable WORM Object Retention Rules (7-Year Lock)
  │   │
  │   └── Network Hub Account / Network Transit Compartment
  │       - AWS Transit Gateway (TGW) / OCI Dynamic Routing Gateway (DRG v2)
  │       - Network Firewall (AWS Network Firewall / OCI Network Firewall)
  │       - Direct Connect / FastConnect On-Premises Interconnects
  │
  ├── [ WORKLOADS / APPLICATION OUs / COMPARTMENTS ]
  │   ├── Production OU / Production Compartment
  │   │   - Production Application Accounts / Spoke VCNs
  │   │   - Strict SCPs: Deny disabling logging, deny public S3, deny unapproved regions
  │   │
  │   ├── Staging OU / Staging Compartment
  │   │   - Pre-production mirrored workloads
  │   │
  │   └── Development / Sandbox OU / Sandbox Compartments
  │       - Developer experimentation with strict monthly budget caps and auto-reap
========================================================================================================================
```

---

## 3. Dual-Cloud Component Mapping Matrix

| Architectural Domain | AWS Native Primitive | OCI Native Primitive | Engineering Comparison |
| :--- | :--- | :--- | :--- |
| **Governance Boundary** | AWS Organizations + AWS Control Tower `[Doc: Control Tower, checked 2026]` | OCI IAM Tenancy + Hierarchical Compartments `[Doc: OCI IAM, checked 2026]` | AWS isolates blast radius via distinct 12-digit accounts; OCI achieves identical isolation within a single tenancy via nested compartments. |
| **Preventive Guardrails** | Service Control Policies (SCPs) | OCI Compartment IAM Policies | AWS SCPs apply permission boundaries to root/OU/account; OCI policies grant explicit access to dynamic groups within target compartments. |
| **Transit Networking** | AWS Transit Gateway (TGW) | OCI Dynamic Routing Gateway (DRG v2) | TGW scales to 5,000 VPC attachments; DRG v2 connects up to 300 VCNs across compartments with native routing tables. |
| **Identity Federation** | AWS IAM Identity Center (formerly AWS SSO) | OCI IAM Identity Domains (Identity Cloud Service) | Both support SAML 2.0 and SCIM automated user provisioning from enterprise IdPs (Okta, Azure AD). |
| **Audit & Log Vault** | AWS CloudTrail + S3 Object Lock Vault | OCI Audit Service + Object Storage Retention Rules | Aggregates all control-plane API calls into an immutable, tamper-proof repository in an isolated account/compartment. |
| **Centralized Inspection**| AWS Network Firewall | OCI Network Firewall (Palo Alto powered) | Statefully inspects East-West (VPC-to-VPC) and North-South (Internet egress) traffic. |

---

## 4. Production Infrastructure as Code (Terraform HCL)

### 4.1 AWS Terraform Module (`aws_landing_zone.tf`)

```hcl
# AWS Reference Implementation: Transit Gateway and Production OU
resource "aws_organizations_organizational_unit" "workloads_prod" {
  name      = "Production-Workloads"
  parent_id = var.root_ou_id
}

resource "aws_organizations_policy" "deny_unapproved_regions_scp" {
  name        = "EnforceRegionLockSCP"
  description = "Deny all API actions outside authorized regions"
  type        = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyUnapprovedRegions"
        Effect    = "Deny"
        NotAction = [
          "iam:*",
          "organizations:*",
          "route53:*",
          "cloudfront:*",
          "support:*"
        ]
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "aws:RequestedRegion" = ["us-east-1", "us-west-2"]
          }
        }
      }
    ]
  })
}

resource "aws_ec2_transit_gateway" "hub_tgw" {
  description                     = "Central Hub Transit Gateway"
  default_route_table_association = "disable"
  default_route_table_propagation = "disable"
  dns_support                     = "enable"
  vpn_ecmp_support                = "enable"

  tags = {
    Name        = "prod-network-hub-tgw"
    Environment = "core-network"
  }
}
```

### 4.2 OCI Terraform Module (`oci_landing_zone.tf`)

```hcl
# OCI Reference Implementation: Compartment Hierarchy and DRG v2
resource "oci_identity_compartment" "workloads_prod" {
  compartment_id = var.tenancy_ocid
  name           = "Production"
  description    = "Production Workload Compartment"
  enable_delete  = false
}

resource "oci_core_drg" "hub_drg" {
  compartment_id = var.network_compartment_ocid
  display_name   = "prod-network-hub-drg-v2"
}

resource "oci_identity_policy" "enforce_compartment_guardrail" {
  compartment_id = oci_identity_compartment.workloads_prod.id
  name           = "EnforceProdIsolationPolicy"
  description    = "Strict least privilege policy for production workloads"

  statements = [
    "Allow group Tier1-DevOps to manage all-resources in compartment Production where target.resource.tag.Environment = 'Production'",
    "Deny group Tier1-DevOps to manage buckets in compartment Production where request.permission = 'BUCKET_DELETE'"
  ]
}
```

---

## 5. Security Guardrails & Policy Enforcement

### 5.1 Immutable Audit Log Pipeline
- All AWS CloudTrail logs and OCI Audit logs stream to a centralized, dedicated **Log Archive Account / Logging Compartment**.
- S3 Object Lock in **Compliance Mode** and OCI **Immutable Retention Rules** are enforced with a 7-year retention period. Even cloud root accounts cannot modify or truncate audit trails.

---

## 6. Observability, SLIs/SLOs & Alerting Runbook

1. **Security Hub / Cloud Guard Health Score**: Must remain $> 95\%$ across all accounts.
2. **Transit Gateway / DRG Packet Drops**: Alert if dropped packet rate $> 0.01\%$.
3. **GuardDuty / OCI Threat Detection**: High-severity findings trigger immediate P1 security on-call notification.

---

## 7. Deployment & Verification Runbook

```bash
# Verify SCP Policy Attachment
aws organizations list-policies-for-target   --target-id ${PROD_OU_ID}   --filter "SERVICE_CONTROL_POLICY"

# Verify Transit Gateway Route Table Propagation
aws ec2 get-transit-gateway-route-table-propagations   --transit-gateway-route-table-id ${TGW_PROD_RT_ID}
```

---

## 8. FinOps Cost Breakdown & Sizing Economics

```text
====================================================================================================
                        FINOPS COST BREAKDOWN: LANDING ZONE CORE
====================================================================================================

COMPONENT                         AWS MONTHLY COST        OCI MONTHLY COST
----------------------------------------------------------------------------------------------------
Transit Gateway / DRG v2 Base     $400 (TGW Attachments)  $0.00 (DRG v2 included free)
Network Inspection Firewall       $1,800                  $1,200
Centralized Logging & Audit Vault $450                    $220
Security Hub / GuardDuty / Guard  $650                    $380
----------------------------------------------------------------------------------------------------
TOTAL RUN-RATE                    $3,300 / month          $1,800 / month
====================================================================================================
```

---

## 9. Failure Mode Drills & Chaos Engineering Runbook

### 9.1 Game Day Drill: Rogue Administrator Guardrail Bypass Test
1. **Action**: Attempt to create a public S3 bucket or disable OCI Audit logging using an administrator IAM user.
2. **Verification**:
   - AWS SCP / OCI Policy intercepts the API call and returns `AccessDenied` / `AuthorizationFailed`.
   - Security Hub / Cloud Guard records an unauthorized configuration attempt and fires a Slack notification.
