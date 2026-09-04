# 03. IAM Least Privilege, Permissions Boundaries & Access Governance

## 1. Problem
Enforcing the Principle of Least Privilege across large engineering organizations is one of the most difficult challenges in cloud engineering. Over time, developers request broad permissions to unblock urgent deployments, accumulating "permission creep." Developers rarely remove permissions once a project launches, leaving hundreds of roles with latent administrative privileges. If a machine identity or employee credential is compromised, attackers exploit subtle **Privilege Escalation Vectors** (e.g., `iam:CreatePolicyVersion`, `iam:PassRole`, or `iam:AttachUserPolicy`) to grant themselves full administrative control over the entire cloud infrastructure.

## 2. Cloud Concept: Automated Governance & Guardrails
```text
PERMISSIONS BOUNDARY ENFORCEMENT (DELEGATED ADMINISTRATION):

[Junior Cloud Engineer] ──► Can create IAM Roles for microservices
                                 │
                                 ├──► Attempts to attach: "AdministratorAccess"
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│ AWS IAM PERMISSIONS BOUNDARY / OCI COMPARTMENT POLICY                  │
│                                                                        │
│   Maximum Allowable Permissions: S3, SQS, DynamoDB ONLY!               │
│                                                                        │
│   EFFECTIVE PERMISSIONS = Intersection(Attached Policy, Boundary)      │
│   * The role CANNOT terminate EC2, CANNOT touch IAM, CANNOT touch KMS! │
│   * Administrative privilege escalation is mathematically impossible!  │
└────────────────────────────────────────────────────────────────────────┘
```

- **The Principle of Least Privilege**: Identities must be granted only the minimum permissions necessary to perform their specific job functions, for the minimum duration required.
- **Automated Policy Generation**: Instead of guessing required permissions, cloud security engines analyze actual historical API traffic captured in audit logs and generate mathematically precise least-privilege policies.
- **Permissions Boundaries**: Advanced IAM guardrails that define the maximum allowable permissions an identity can possess. Even if an administrator accidentally attaches `AdministratorAccess`, the effective permissions are clipped to the boundary.

## 3. AWS Least-Privilege Engineering
In AWS:
- **AWS IAM Access Analyzer**:
  - Uses automated formal reasoning to analyze resource-based policies across S3, KMS, SQS, and IAM roles.
  - Flags any policy that allows access from outside the AWS account or organization.
  - **Policy Generation from CloudTrail**: Analyzes 90 days of CloudTrail API activity for a specific developer or service and automatically outputs a minimal JSON policy containing only the exact actions actually executed.
- **Permissions Boundaries for Delegated Administration**:
  - Allows senior DevOps engineers to safely delegate role creation to application teams.
  - The developer's IAM policy requires them to attach a specific Permissions Boundary to any role they create:
    ```json
    "Condition": {
      "StringEquals": { "iam:PermissionsBoundary": "arn:aws:iam::123:policy/AppBoundary" }
    }
    ```

## 4. OCI Policy Governance & Compartment Quotas
In OCI:
- **OCI Compartment Quotas**:
  - Policy-based limits applied to compartments to prevent resource exhaustion and bound cost.
  - Syntax:
    ```sql
    set compute quota vm-standard-e5-flex-count to 20 in compartment Development
    zero compute quota bm-gpu-count in compartment Sandbox
    ```
- **OCI IAM Policy Advisor & Security Zones**:
  - **OCI Security Zones**: Strictly enforces security recipes on compartments. If a developer attempts to create an unencrypted bucket or launch a public database in a Security Zone, OCI physically blocks the API call at the platform level!

## 5. Production Failure Modes: Privilege Escalation Vectors
- **The `iam:PassRole` Escalation Trap**: A developer role lacks administrative permissions, but possesses `iam:PassRole` and `ec2:RunInstances`. The developer launches a new EC2 instance, passes a highly privileged IAM Role (e.g., `SuperAdminRole`) to the instance profile, and SSHs into the instance to steal the superadmin credentials from the Instance Metadata Service!
  - *Fix*: Strictly scope `iam:PassRole` to specific target service ARNs.
- **Dormant Access Key Exploitation**: A developer leaves an unused access key active on their account. Two years later, the key is leaked via a personal laptop backup and exploited.
  - *Fix*: Enforce automated key retirement after 90 days.

## 6. Troubleshooting & Auditing Commands
1. **Identify Unused Credentials via AWS Credential Report**:
   ```bash
   aws iam generate-credential-report
   aws iam get-credential-report --output text --query "Content" | base64 --decode
   ```
   Filter for access keys where `access_key_1_last_used_date` is $> 90\text{ days}$.
2. **Audit Open S3 Buckets via Access Analyzer**:
   ```bash
   aws accessanalyzer list-findings --analyzer-arn <arn> --filter '{"status":{"eq":["ACTIVE"]}}'
   ```

## 7. Senior Interview Question & Defense
**Question**: *Your enterprise wants to enable application developers to create their own IAM roles for AWS Lambda functions and ECS tasks without filing tickets to the security team. How do you architect this self-service model while preventing developers from escalating their own privileges to AdministratorAccess?*

**Staff-Level Defense**:
> "We implement **Delegated Administration with Cryptographically Enforced Permissions Boundaries**:
>
> 1. **The Vulnerability We Must Prevent**:
>    - If we grant developers `iam:CreateRole` and `iam:AttachRolePolicy`, a developer can simply create an IAM role with `AdministratorAccess` and assume it, bypassing all organizational controls.
>
> 2. **The Permissions Boundary Architectural Guardrail**:
>    - We define a central **Permissions Boundary policy** (`DevWorkloadBoundary`) that limits permissions strictly to application resources:
>      - Allows reading/writing to application S3 buckets, DynamoDB tables, and SQS queues.
>      - **Explicitly denies** any IAM actions (`iam:*`), KMS key deletion (`kms:ScheduleKeyDeletion`), and security modifications.
>
> 3. **The Developer IAM Policy with Condition Gates**:
>    - We grant developers permissions to create roles **only if they attach the boundary during creation**:
>      ```json
>      {
>        "Effect": "Allow",
>        "Action": ["iam:CreateRole", "iam:AttachRolePolicy", "iam:PutRolePolicy"],
>        "Resource": "arn:aws:iam::*:role/app-*",
>        "Condition": {
>          "StringEquals": {
>            "iam:PermissionsBoundary": "arn:aws:iam::123:policy/DevWorkloadBoundary"
>          }
>        }
>      }
>      ```
>    - We add an explicit Deny preventing developers from modifying or deleting the `DevWorkloadBoundary` itself, and restrict `iam:PassRole` exclusively to roles possessing the boundary.
>
> 4. **The Security Guarantee**:
>    - Even if a developer attaches `AdministratorAccess` to their new role, the **Effective Permissions are the mathematical intersection between the attached policy and the Permissions Boundary**.
>    - The new role can never perform any action outside the boundary, empowering developers to move fast with zero security ticket overhead while making privilege escalation mathematically impossible."
