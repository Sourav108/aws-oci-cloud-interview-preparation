# 01. IAM vs. OCI Identity Domains & Compartment Hierarchy

## 1. Problem
In modern enterprise cloud environments, securing thousands of cloud resources across hundreds of development teams is fraught with operational danger. Naive security setups either grant overly permissive administrative access (`AdministratorAccess` or `manage all-resources`), creating severe blast-radius vulnerabilities where a single leaked API key can delete production databases, or enforce fragmented, conflicting policy rules that block critical automated deployments. Furthermore, organizations migrating between AWS and OCI frequently stumble over fundamental structural differences: attempting to isolate projects in OCI by creating separate tenancies (like AWS accounts) instead of leveraging OCI's native hierarchical **Compartments**, or misunderstanding how cloud policy evaluation engines adjudicate conflicting permissions.

## 2. Cloud Concept
### Identity & Tenancy Primitives
- **AWS Multi-Account Architecture**:
  - In AWS, the **Account** is the fundamental blast-radius and security boundary.
  - Multi-account governance is orchestrated via **AWS Organizations**:
    - **Organizational Units (OUs)**: Logical trees grouping accounts.
    - **Service Control Policies (SCPs)**: Centralized JSON guardrails applied at the OU or Account level that set the maximum allowable permissions. SCPs act as filters; they never grant permissions, only bound them.
  - **AWS IAM**:
    - *IAM User*: Long-lived credentials (passwords, access keys). Strongly discouraged for machines and humans in modern architectures.
    - *IAM Role*: Identity assumed by humans (via SSO) or compute services (EC2, Lambda) granting temporary, auto-expiring security tokens via AWS Security Token Service (STS).
- **OCI Tenancy & Identity Domains Architecture**:
  - In OCI, an enterprise owns a single top-level **Tenancy** (the root container mapped to a cloud contract) `[Doc: OCI IAM Overview, checked 2026-09-04]`.
  - **OCI IAM Identity Domains**:
    - A container for managing users, groups, and authentication policies (MFA, SSO, federation).
    - Identity Domains can be created per environment (e.g., Development Domain, Production Domain) or centralized in a Primary Domain.
  - **OCI Compartments (The Architectural Masterpiece)**:
    - Compartments are logical partitions within a single tenancy used to organize, isolate, and control access to cloud resources.
    - **Hierarchical Nesting**: Compartments can be nested inside other compartments **up to 6 levels deep** `[Doc: OCI Compartment Limits, checked 2026-09-04]`.
    - **Policy Inheritance**: Policies written at a parent compartment automatically cascade down to all child and sub-compartments!
    - **Resource Isolation**: A resource (VCN, Compute VM, Autonomous DB) belongs to **exactly one compartment**. Moving a resource between compartments updates its governing policies dynamically without destroying the resource.

### The Policy Evaluation Engine: AWS vs. OCI
The authorization engines of AWS and OCI evaluate API requests using fundamentally different rule mechanisms:

```text
AWS IAM EVALUATION ENGINE:
1. Default: IMPLICIT DENY
2. Check SCPs (AWS Organizations): Is action permitted by all parent SCPs?
   * NO ──► EXPLICIT DENY (Request dropped!)
3. Check Resource-Based Policies (S3, KMS, SQS): Does an explicit ALLOW exist?
4. Check IAM Permissions Boundaries: Is action within the boundary?
5. Check Identity-Based Policies: Does an explicit ALLOW exist?
6. Check for ANY EXPLICIT DENY across ALL policies:
   * If even ONE explicit DENY matches ──► FINAL DECISION = DENIED!
   * If ALLOW exists and NO DENY ───────► FINAL DECISION = ALLOWED!
```

```text
OCI DECLARATIVE VERB HIERARCHY:
1. Default: NO ACCESS (Implicit Deny)
2. Evaluate Compartment Policy Statements:
   "Allow group <GroupName> to <VERB> <RESOURCE-TYPE> in compartment <CompartmentName> where <Conditions>"
3. Verb Inheritance Cascade:
   * INSPECT : List resources and metadata without viewing user data or contents.
   * READ    : INSPECT + view resource configuration and download contents (e.g., read object).
   * USE     : READ + update existing resources (e.g., start/stop VM, attach volume); CANNOT create/delete!
   * MANAGE  : Full administrative control: Create, Read, Update, Delete, Move across compartments.
4. If ANY policy grants the required verb ──► FINAL DECISION = ALLOWED!
   (OCI does not use AWS-style "Deny" statements; permissions are strictly additive).
```

## 3. Mental Model
Think of cloud authorization structures as commercial real estate:
- **AWS Multi-Account Architecture** is a corporate campus made up of 50 separate, independent physical office buildings (Accounts). Each building has its own locked front door and dedicated security guards. If you want an employee from Building A to inspect a server in Building B, you must issue them a special temporary visitor badge (Cross-Account STS AssumeRole).
- **OCI Compartment Hierarchy** is a 50-story skyscraper (Tenancy). The building has a single central security desk. Each floor or wing is a Compartment (Floor 10 = Finance, Floor 11 = Engineering). The building owner sets a policy: *"All security guards can access all floors"* (Parent Policy Inheritance). A manager on Floor 10 can create sub-rooms (child compartments) and give access keys to specific team members without building a new skyscraper.

## 4. Architecture Diagram
```text
AWS MULTI-ACCOUNT SCP VS. OCI COMPARTMENT HIERARCHY:

AWS ORGANIZATIONS TOPOLOGY:
┌────────────────────────────────────────────────────────────────────────┐
│ ROOT ORGANIZATION                                                      │
│   └── OU: Production (Enforces SCP: Deny leaving us-east-1)            │
│         ├──► AWS Account 1111: Network Shared Services                 │
│         └──► AWS Account 2222: Payments Microservices                  │
│   └── OU: Development (Enforces SCP: Deny expensive GPU instances)     │
│         └──► AWS Account 3333: Sandbox Workloads                       │
└────────────────────────────────────────────────────────────────────────┘

OCI TENANCY COMPARTMENT HIERARCHY (Single Tenancy, Zero Account Sprawl):
┌────────────────────────────────────────────────────────────────────────┐
│ ROOT TENANCY: EnterpriseCorp (Compartment OCID: ocid1.tenancy.oc1...)  │
│ Policy: "Allow group SecurityAdmins to inspect all-resources in tenancy"│
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ COMPARTMENT: Production (Level 1)                              │   │
│   │ Policy: "Allow group ProdDevs to use all-resources in comp Prod"│  │
│   │                                                                │   │
│   │   ┌──────────────────────────┐    ┌──────────────────────────┐ │   │
│   │   │ Sub-Comp: Network (L2)   │    │ Sub-Comp: Database (L2)  │ │   │
│   │   │ * VCNs, DRG v2, FastConn │    │ * Autonomous DB, Exadata │ │   │
│   │   └──────────────────────────┘    └──────────────────────────┘ │   │
│   └────────────────────────────────────────────────────────────────┘   │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ COMPARTMENT: Non-Production (Level 1)                          │   │
│   │   └── Sub-Comp: Dev-Sandbox (Level 2)                          │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS:
- **Service Control Policies (SCPs)**:
  - Attached to Root, OUs, or Accounts.
  - Classic security guardrail: Region Restriction SCP. Prevents developers from accidentally launching resources in unauthorized foreign regions:
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "DenyUnauthorizedRegions",
          "Effect": "Deny",
          "NotAction": ["iam:*", "organizations:*", "route53:*", "cloudfront:*"],
          "Resource": "*",
          "Condition": {
            "StringNotEquals": { "aws:RequestedRegion": ["us-east-1", "us-west-2"] }
          }
        }
      ]
    }
    ```
- **Permissions Boundaries**:
  - An advanced IAM control used to delegate administrative privileges safely.
  - A senior engineer allows junior developers to create IAM roles for Lambda, but attaches a **Permissions Boundary** to the developer role. Even if the developer attempts to attach `AdministratorAccess` to their new Lambda role, the effective permissions are clipped to the boundary.
- **Resource-Based Policies vs. Identity Policies**:
  - *Identity Policy*: Attached to User/Role; defines what they can access.
  - *Resource Policy*: Attached directly to S3 buckets, SQS queues, or KMS keys; can grant access to identities in **different AWS accounts** without the target role needing to assume a role.

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **The Elegance of OCI Policy Syntax**:
  - OCI rejected complex, verbose 50-line JSON documents in favor of human-readable, declarative English statements:
    $$\text{Allow } \langle\text{Subject}\rangle \text{ to } \langle\text{Verb}\rangle \text{ } \langle\text{Resource-Type}\rangle \text{ in } \langle\text{Location}\rangle \text{ where } \langle\text{Conditions}\rangle$$
  - *Subject*: `group <GroupName>`, `dynamic-group <DGName>`, or `any-user`.
  - *Verb*: `inspect`, `read`, `use`, or `manage`.
  - *Resource-Type*: `all-resources`, `virtual-network-family`, `instance-family`, `database-family`, `object-family`.
  - *Location*: `tenancy` or `compartment <CompartmentName>`.
- **Compartment Hierarchical Inheritance**:
  - A policy written at the Root Tenancy level:
    ```sql
    Allow group SecurityAuditors to inspect all-resources in tenancy
    ```
    automatically grants read-only inspection access across every single existing and future compartment in the entire organization!
- **OCI Network Sources**:
  - A native security construct that defines an IP whitelist (e.g., corporate office public IPs or a private VCN CIDR).
  - Can be embedded directly into IAM policies:
    ```sql
    Allow group CorporateAdmins to manage all-resources in tenancy
    where request.networkSource.name = 'CorpHeadquartersNetwork'
    ```
    If an admin's credentials are stolen, the attacker cannot execute API calls from the public internet!
- **Dynamic Groups**:
  - Groups OCI compute instances, Autonomous Databases, or functions based on matching rules (e.g., `ALL {instance.compartment.id = 'ocid1.compartment.oc1..abc'}`).
  - Eliminates AWS-style instance profile provisioning; all compute instances in the compartment automatically inherit the dynamic group's policies.

## 7. Configuration
Comparing identity policies in Terraform across AWS and OCI:

### AWS IAM Policy with Condition Keys (Terraform)
```hcl
# AWS IAM Policy restricting access to a specific private VPC CIDR
resource "aws_iam_policy" "s3_restricted_policy" {
  name        = "S3RestrictedAccessPolicy"
  description = "Allows S3 access strictly from corporate network CIDR"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid      = "AllowS3Actions"
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
        Resource = ["arn:aws:s3:::corporate-finance-vault", "arn:aws:s3:::corporate-finance-vault/*"]
        Condition = {
          IpAddress = {
            "aws:SourceIp" = ["198.51.100.0/24"] # Corporate Office IP
          }
        }
      }
    ]
  })
}
```

### OCI IAM Policy with Compartments & Network Sources (Terraform)
```hcl
# 1. OCI Network Source Definition
resource "oci_core_network_source" "corp_network" {
  compartment_id = var.tenancy_ocid
  name           = "CorpNetworkSource"
  description    = "Corporate headquarters IP range"
  public_source_list = ["198.51.100.0/24"]
}

# 2. OCI Human-Readable Policy Statement
resource "oci_identity_policy" "finance_policy" {
  compartment_id = var.tenancy_ocid
  name           = "finance-compartment-policy"
  description    = "Grants Finance team access to Finance compartment from corporate IP"

  statements = [
    "Allow group FinanceTeam to manage object-family in compartment Finance where request.networkSource.name = 'CorpNetworkSource'",
    "Allow group FinanceTeam to read virtual-network-family in compartment Finance"
  ]
}
```

## 8. Data Flow
```text
The AWS Explicit Deny vs. OCI Verb Authorization Flow:

AWS API REQUEST: ec2:TerminateInstances
1. IAM Engine evaluates SCPs: Allow ec2:*? YES.
2. IAM Engine evaluates Identity Policy: Allow ec2:TerminateInstances? YES.
3. IAM Engine evaluates Tag Policy: Effect = Deny if Environment != "Dev".
4. Tag is "Prod" ──► EXPLICIT DENY TRIGGERED!
5. Final Evaluation: Explicit Deny overrides all allows ──► HTTP 403 Access Denied.

OCI API REQUEST: LaunchInstance
1. IAM Engine searches policies matching User's Groups.
2. Finds statement: "Allow group Developers to use instance-family in compartment Dev"
3. Evaluates Verb: Does 'use' permit creating a new instance?
   * Check verb table: 'use' allows reboot/stop, but CANNOT create!
4. Engine checks for higher verb: No 'manage' statement found.
5. Final Evaluation: No matching permission ──► HTTP 404/403 Not Authorized.
```

## 9. Security
- **Eliminating AWS Root Account Usage**:
  - The AWS Root user possesses unrestricted access that cannot be limited by IAM policies. Lock Root with hardware MFA, delete access keys, and use it exclusively for account creation and billing plan changes.
  - In OCI, the default tenancy administrator account should similarly be protected by FIPS-compliant MFA and reserved for emergency break-glass scenarios.
- **Enforcing Least-Privilege Verbs in OCI**:
  - Never use the `manage` verb when `use` or `read` suffices:
    - If a monitoring tool only needs to list servers, grant `inspect instance-family`.
    - If a CI/CD pipeline deploys code to existing servers, grant `use instance-family`.
    - Reserve `manage` strictly for Terraform infrastructure provisioning pipelines.

## 10. Reliability
- **Compartment Deletion Safeguards**:
  - In OCI, you cannot accidentally delete a compartment that contains active resources!
  - All compute instances, databases, and volumes inside the compartment must be permanently terminated before OCI permits compartment deletion, preventing catastrophic accidental corporate wipes.

## 11. Scaling
- **Managing Multi-Account Sprawl vs. Compartment Nesting**:
  - In AWS, large enterprises routinely accumulate 500 to 2,000 separate AWS accounts, requiring complex tooling (AWS Control Tower, Transit Gateways) to manage network connectivity and cross-account billing.
  - In OCI, a single Tenancy scales to support the entire enterprise by organizing teams into hierarchical nested compartments (e.g., `Root` $\longrightarrow$ `Division` $\longrightarrow$ `Department` $\longrightarrow$ `Environment`), drastically simplifying billing and centralized policy management.

## 12. Observability
- **Audit Logging**:
  - AWS CloudTrail: Records every IAM policy alteration, `sts:AssumeRole` call, and API execution.
  - OCI Audit Service: Automatically records every API call across all compartments in an immutable JSON event stream retained for **365 days by default** `[Doc: OCI Audit Service, checked 2026-09-04]`.

## 13. Cost
- Both AWS IAM and OCI IAM are **100% Free core platform services**. There is zero cost to create users, groups, roles, compartments, or identity domains.
- Savings stem from governance: using OCI Compartment Quotas or AWS SCPs to block developers from accidentally launching \$30/hour GPU instances in sandbox environments.

## 14. Failure Modes
- **The Confused Deputy Vulnerability**: A third-party monitoring SaaS asks you to assume an IAM Role in your AWS account. If the trust policy only checks the SaaS account ID without enforcing an **External ID**, an attacker can trick the SaaS into accessing your AWS account using the SaaS service's credentials! *Remediation: Mandatory `sts:ExternalId` in trust policies.*
- **The Phantom Explicit Deny Lockout in AWS**: An engineer adds an SCP with `"Effect": "Deny", "Action": "*"` with a condition excluding their own admin role. However, they forgot that SCPs apply to the **entire account, including the administrator**! All engineers, CI/CD pipelines, and automated healing scripts are instantly locked out of the account.

## 15. Troubleshooting
When an API call returns `AccessDenied` or `NotAuthorizedOrNotFound`:
1. **In AWS**:
   - Inspect the decoded authorization failure message using AWS CLI:
     ```bash
     aws sts decode-authorization-message --encoded-message <error-token>
     ```
   - Shows precisely which policy (SCP, Permissions Boundary, or Identity Policy) caused the Deny.
2. **In OCI**:
   - Remember: OCI returns `404 Not Found` instead of `403 Forbidden` if the user lacks the `inspect` verb on the resource, deliberately hiding the existence of confidential resources!
   - Verify: Does the policy statement target the exact compartment where the resource lives, or a parent compartment?

## 16. Common Mistakes
- **Assuming AWS Deny Exists in OCI**: Trying to write an OCI policy statement with `Deny group Developers to...`. OCI policy syntax has no `Deny` keyword! To restrict access in OCI, simply omit the `Allow` statement or use negative `where` condition clauses.
- **Using Wildcards (`*`) Everywhere in AWS**: Granting `s3:*` on `*` because it's easier during initial development. An attacker compromising that role can download, overwrite, or delete every S3 bucket across the entire enterprise.

## 17. Trade-offs
| Governance Model | Structural Primitive | Multi-Tenancy Boundary | Policy Complexity |
| :--- | :--- | :--- | :--- |
| **AWS Organizations** | Independent AWS Accounts | Physical Account Boundary | High (JSON SCPs, Boundaries, Roles) |
| **OCI Compartments** | Nested Folders (1 to 6 levels)| Logical Compartment Boundary | **Low (English Declarative Verbs)** |
| **AWS Network Filtering**| Condition keys (`aws:SourceIp`) | Per-policy JSON evaluation | Moderate |
| **OCI Network Sources**| Centralized Named Network Source | Reusable across all policies | Low |

## 18. Interview Questions
1. *Walk me through the exact step-by-step evaluation logic executed by the AWS IAM policy engine when a user requests an API action. How does an Explicit Deny interact with SCPs, Permissions Boundaries, and Resource Policies?*
2. *How do OCI Compartments differ architecturally from AWS Accounts as a multi-tenancy isolation mechanism? How does policy inheritance work across nested compartments?*
3. *Explain the Confused Deputy problem in cross-account cloud access. How does AWS STS solve it using `sts:ExternalId`, and how does OCI handle cross-tenancy authorization?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "The AWS IAM Policy Evaluation Engine operates on a strict, deterministic sequence governed by the foundational rule: **Explicit Deny always wins, and the default is always an Implicit Deny**:
>
> 1. **Step 1: Start with Default Implicit Deny**:
>    - Every request starts unauthorized. Unless an explicit authorization is found, the final verdict is Denied.
>
> 2. **Step 2: Service Control Policies (SCPs) Evaluation**:
>    - If the account is inside an AWS Organization, IAM evaluates all SCPs attached to the account and its parent OUs.
>    - If any SCP contains an **Explicit Deny** matching the action, or if no SCP explicitly permits the action, the request is **immediately Denied**, completely bypassing local IAM policies!
>
> 3. **Step 3: Resource-Based Policy Check**:
>    - If the target resource (e.g., an S3 bucket or KMS key) has a Resource Policy that explicitly permits the request from the caller, and the request is in the same account, this can grant access. If cross-account, both the identity policy and resource policy must agree.
>
> 4. **Step 4: IAM Permissions Boundary Check**:
>    - If the calling IAM user or role has an attached Permissions Boundary, the requested action **must be explicitly allowed by the boundary**. If the boundary does not permit the action, access is denied.
>
> 5. **Step 5: Identity-Based Policies & Session Policies**:
>    - IAM evaluates all managed and inline policies attached to the user, groups, or assumed role session. At least one policy must have an `Effect: Allow` matching the requested Action and Resource.
>
> 6. **Step 6: The Universal Explicit Deny Filter**:
>    - IAM scans all applicable policies across all levels (SCPs, Boundaries, Identity Policies, Resource Policies).
>    - If **even one single policy statement** contains `"Effect": "Deny"` that matches the request, **the entire request is instantly and irreversibly rejected**, overriding any and all `Allow` statements found in any other policy.
>
> This multi-layered evaluation engine guarantees that organizational guardrails (SCPs) and security boundaries can never be overridden by individual account administrators."

## 20. Hands-on Exercise
**Objective**: Create an OCI Compartment hierarchy with inherited policy verbs using Terraform or OCI CLI.

### Verification Steps
1. Create a parent compartment `SecurityRoot` and a child compartment `NetworkProd`:
   ```bash
   oci iam compartment create --name SecurityRoot --description "Parent Compartment"
   oci iam compartment create --name NetworkProd --description "Child Network Compartment" \
     --compartment-id <SecurityRoot-OCID>
   ```
2. Apply an inherited policy at the `SecurityRoot` level:
   ```bash
   oci iam policy create --compartment-id <SecurityRoot-OCID> --name AuditPolicy \
     --description "Audit all child compartments" \
     --statements '["Allow group SecurityAuditors to inspect all-resources in compartment SecurityRoot"]'
   ```
3. Test with a user in `SecurityAuditors`:
   - Verify the user can run `oci network vcn list --compartment-id <NetworkProd-OCID>`.
   - Confirm that the user can inspect resources in the child compartment without an explicit policy written on `NetworkProd`, proving **hierarchical policy inheritance**.
