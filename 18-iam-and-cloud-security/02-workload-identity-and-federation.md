# 02. Workload Identity, Cross-Account Access & Enterprise Federation

## 1. Problem
Hardcoding static cloud credentials (such as AWS Access Key IDs, Secret Access Keys, or OCI API Signing Keys) inside application code, Docker images, or configuration files is the single leading cause of catastrophic cloud security breaches. When developers accidentally commit a private API key to a public GitHub repository, automated bot scanners discover and exploit it in under 60 seconds, launching thousands of unauthorized GPU instances or exfiltrating production databases. Furthermore, enterprise IT environments require centralizing identity management in enterprise Identity Providers (Okta, Microsoft Entra ID, Ping Identity) rather than manually creating duplicate user accounts in every cloud account. Scalable cloud security demands **ephemeral, token-based machine workload identity and standards-based federation**.

## 2. Cloud Concept
### Machine Workload Identity: Eliminating Long-Lived Secrets
Workload Identity enables running software components (Kubernetes pods, virtual machines, serverless functions, CI/CD pipelines) to authenticate directly to the cloud control plane without storing static secrets:
- **AWS Instance Profiles & Service Roles**: EC2 instances and Lambda functions assume an IAM Role automatically. The hypervisor injects temporary credentials (valid for 1 to 6 hours) into the local Instance Metadata Service (IMDS).
- **AWS IAM Roles for Service Accounts (IRSA)**: Uses OpenID Connect (OIDC) federation between the EKS cluster and AWS IAM. The kubelet projects a cryptographically signed JSON Web Token (JWT) into the pod. The AWS SDK calls `sts:AssumeRoleWithWebIdentity` to exchange the JWT for short-lived AWS credentials.
- **AWS EKS Pod Identity**: Modern daemon-based alternative that eliminates maintaining OIDC issuer URLs in IAM trust policies.
- **OCI Dynamic Groups**:
  - Groups OCI compute instances, Autonomous Databases, or API Gateway instances based on matching rules (e.g., `ALL {instance.compartment.id = 'ocid1.compartment.oc1..abc'}`).
  - Applications call OCI SDKs using **Instance Principals**: the SDK automatically signs API requests using the instance's private key without any credential configuration `[Doc: OCI Instance Principals, checked 2026-09-04]`.
- **OCI Workload Identity**: Federates Kubernetes Service Accounts in OKE directly with OCI IAM policies via OIDC.

### Cross-Account vs. Cross-Tenancy Access
- **AWS Cross-Account Access (STS `AssumeRole`)**:
  - Account A (Security Tooling) needs to scan Account B (Production Workload).
  - Account B defines an IAM Role with a Trust Policy trusting Account A:
    `"Principal": { "AWS": "arn:aws:iam::AccountA:root" }`.
  - To prevent the **Confused Deputy Attack**, Account B enforces a mandatory condition:
    `"StringEquals": { "sts:ExternalId": "SecretClientUUID" }`.
- **OCI Cross-Tenancy Access (`Define`, `Endorse`, `Admit`)**:
  - In OCI, different enterprises or business units reside in separate **Tenancies**.
  - OCI coordinates cross-tenancy access through declarative cryptographic handshakes:
    1. Both tenancies define a reference alias: `Define tenancy <TargetTenancy> as <TargetOCID>`.
    2. The source tenancy **Endorses** its group:
       `Endorse group DataAuditors to read all-resources in tenancy PartnerTenancy`.
    3. The destination tenancy **Admits** the remote group:
       `Admit group DataAuditors of tenancy AuditorTenancy to read all-resources in compartment Production`.

### Enterprise Federation: SAML 2.0 & OIDC
- Centralizes user lifecycle management in corporate IdPs (Microsoft Entra ID, Okta, Ping Identity).
- Users log in via single sign-on (SSO); the IdP asserts identity claims (groups, roles) using SAML 2.0 XML or OIDC JSON Web Tokens.
- Cloud platforms (AWS IAM Identity Center / OCI IAM Identity Domains) map corporate IdP groups directly to cloud permission sets and compartment policies.

## 3. Mental Model
Think of workload identity and cross-account access as international diplomatic travel:
- **Static API Keys** is carrying \$100,000 in cash in an unzipped backpack through an airport. If someone pickpockets you, your money is gone instantly.
- **Workload Identity (STS / Instance Principals)** is a biometric passport: the border control scanner checks your facial scan and issues a 1-day electronic visitor visa that expires automatically at midnight.
- **External ID in Cross-Account Trust** is a secret diplomatic password exchanged in advance. If a foreign diplomat arrives claiming to represent Company X, the guard checks both their passport AND the secret password to prevent identity spoofing.

## 4. Architecture Diagram
```text
WORKLOAD IDENTITY FEDERATION & CONFUSED DEPUTY MITIGATION:

CROSS-ACCOUNT STS ASSUMEROLE WITH EXTERNAL ID:
┌────────────────────────────────────────────────────────────────────────┐
│ THIRD-PARTY SAAS MONITORING ACCOUNT (Account ID: 111122223333)         │
│   [SaaS Worker Application]                                            │
│   Executes: sts:AssumeRole(RoleArn="arn:aws:iam::444455556666:role/Scan",│
│                          ExternalId="cust_uuid_99420")                 │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ Present Role ARN + External ID
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│ CUSTOMER AWS PRODUCTION ACCOUNT (Account ID: 444455556666)             │
│                                                                        │
│   IAM ROLE: SecurityScannerRole                                        │
│   TRUST POLICY:                                                        │
│   {                                                                    │
│     "Principal": { "AWS": "arn:aws:iam::111122223333:root" },          │
│     "Action": "sts:AssumeRole",                                        │
│     "Condition": {                                                     │
│       "StringEquals": { "sts:ExternalId": "cust_uuid_99420" }          │
│     }                                                                  │
│   }                                                                    │
│                                                                        │
│   * If malicious customer attempts to trick SaaS into assuming role    │
│     without knowing 'cust_uuid_99420' ──► STS REJECTS ACCESS!          │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS:
- **The Modern EKS Pod Identity Agent**:
  - Eliminates the operational complexity of managing IAM OIDC Identity Providers for each EKS cluster.
  - Deploys an Amazon EKS Pod Identity Agent DaemonSet on worker nodes.
  - Pods obtain temporary credentials via an internal link-local daemon endpoint, simplifying trust policies to:
    ```json
    "Principal": { "Service": "pods.eks.amazonaws.com" }
    ```
- **Cross-Account Role Assumption**:
  - STS `AssumeRole` returns temporary credentials: `AccessKeyId`, `SecretAccessKey`, and `SessionToken`.
  - Configurable session duration from **15 minutes up to 12 hours**.
  - Role chaining (assuming Role A to assume Role B) caps maximum session duration at **1 hour**.
- **AWS IAM Identity Center (Successor to AWS SSO)**:
  - Centralized portal integrating with Microsoft Entra ID / Okta via SCIM (System for Cross-domain Identity Management) and SAML 2.0.
  - Automatically provisions users and assigns multi-account Permission Sets.

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **OCI Instance Principals (Zero-Secret Compute Security)**:
  - In OCI, an instance can be authenticated as a first-class principal `[Doc: OCI Instance Principals, checked 2026-09-04]`.
  - Architecture:
    1. Create a **Dynamic Group** with a rule:
       ```sql
       ANY { instance.id = 'ocid1.instance.oc1..abc' }
       ```
    2. Write an IAM policy granting the Dynamic Group permissions:
       ```sql
       Allow dynamic-group AppServers to manage object-family in compartment Media
       ```
    3. Application code initializes the SDK with `InstancePrincipalsAuthenticationDetailsProvider`.
    4. The SDK queries the local instance metadata endpoint (`http://169.254.169.254/opc/v2/identity/cert.pem`) to obtain the hypervisor-signed X.509 certificate and private key, automatically signing all REST API calls!
- **OCI Cross-Tenancy Policies**:
  - Completely eliminates the need to maintain cross-account STS tokens.
  - Both tenancies establish mutual trust statements (`Define`, `Endorse`, `Admit`).
  - Users authenticate once to their home tenancy and directly manage authorized compartments in the partner tenancy with zero credential hopping.
- **OCI Identity Domains Federation**:
  - Identity Domains feature native, built-in support for SAML 2.0 and OIDC Identity Providers.
  - Supports automated user provisioning and group mapping from Microsoft Entra ID and Okta via SCIM v2.0.

## 7. Configuration
Comparing workload identity and cross-account policies in Terraform across AWS and OCI:

### AWS Cross-Account IAM Role with External ID (Terraform)
```hcl
# AWS IAM Role with External ID condition to prevent Confused Deputy
resource "aws_iam_role" "cross_account_scanner" {
  name = "CrossAccountSecurityScannerRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::111122223333:root" # Trusted Partner Account
        }
        Action = "sts:AssumeRole"
        Condition = {
          StringEquals = {
            # Cryptographic External ID shared exclusively between customer and SaaS!
            "sts:ExternalId" = var.saas_customer_external_id
          }
        }
      }
    ]
  })
}

# Attach read-only security inspection policy
resource "aws_iam_role_policy_attachment" "security_read" {
  role       = aws_iam_role.cross_account_scanner.name
  policy_arn = "arn:aws:iam::aws:policy/SecurityAudit"
}
```

### OCI Dynamic Group & Policy for Instance Principals (Terraform)
```hcl
# 1. OCI Dynamic Group matching instances in App compartment
resource "oci_identity_dynamic_group" "app_servers" {
  compartment_id = var.tenancy_ocid
  name           = "AppServersDynamicGroup"
  description    = "Matches all compute instances in the Application compartment"
  matching_rule  = "ALL {instance.compartment.id = '${var.app_compartment_id}'}"
}

# 2. Grant Dynamic Group access to write to Object Storage
resource "oci_identity_policy" "dynamic_group_policy" {
  compartment_id = var.tenancy_ocid
  name           = "app-servers-storage-policy"
  description    = "Allows instances to write backups to Object Storage"

  statements = [
    "Allow dynamic-group AppServersDynamicGroup to manage object-family in compartment Backups"
  ]
}
```

## 8. Data Flow
```text
The OCI Instance Principal Authentication Flow:
1. Application on Compute VM needs to upload file to OCI Object Storage.
2. Code initializes SDK: provider = InstancePrincipalsAuthenticationDetailsProvider()
3. SDK queries local metadata service: http://169.254.169.254/opc/v2/identity/
   - Retrieves leaf certificate, intermediate CA, and RSA private key.
4. SDK constructs HTTP REST request: PUT /n/namespace/b/bucket/o/file.png
5. SDK cryptographically signs request headers with the instance's private key (SHA-256 RSA).
6. Request hits OCI Object Storage API.
7. OCI IAM Engine:
   - Validates instance certificate against internal Oracle CA.
   - Evaluates Dynamic Group matching rules (Instance belongs to AppServersDynamicGroup).
   - Validates IAM policy grants 'manage object-family'.
8. Request permitted! File uploaded in 12ms with ZERO static credentials on disk!
```

## 9. Security
- **Hardening the Instance Metadata Service (IMDSv2)**:
  - Attackers exploiting Server-Side Request Forgery (SSRF) vulnerabilities in web applications attempt to fetch instance credentials from `http://169.254.169.254`.
  - **Enforce IMDSv2**: Requires a session-oriented `PUT` request with a token before reading metadata.
  - Set `http_put_response_hop_limit = 1` to prevent containers running in network bridge mode from accessing the host metadata endpoint.
- **Shortening STS Session Lifetimes**:
  - Configure cross-account roles with the minimum allowable session duration (e.g., 1 hour instead of 12 hours) to bound the blast radius of compromised ephemeral tokens.

## 10. Reliability
- **Automated Credential Rotation**:
  - Workload identity eliminates manual secret rotation outages.
  - In traditional setups, updating an expired API key across 200 servers often results in accidental downtime when one server is missed. With Instance Principals and IRSA, credentials rotate transparently in background threads.

## 11. Scaling
- **Avoiding STS API Rate Limiting**:
  - AWS STS enforces regional API request rate limits on `AssumeRole` (e.g., 500 requests/sec).
  - In a cluster with 5,000 microservice containers, each container must **cache STS credentials in memory** and refresh them only when 80% of the session lifetime has elapsed, rather than calling STS on every HTTP request!

## 12. Observability
- **Tracking Identity Transitions in Audit Logs**:
  - AWS CloudTrail records:
    - `AssumeRole`: Tracks source IP, assumed role name, and `externalId`.
    - `AssumedRoleIdentifier`: Identifies the exact session name that executed downstream API calls.
  - OCI Audit records:
    - `principalId`: Displays the OCID of the Dynamic Group and the specific compute instance that executed the API call.

## 13. Cost
- Workload identity mechanisms (AWS STS, OCI Instance Principals, Dynamic Groups) are **100% Free of charge**.
- Significant financial savings stem from eliminating expensive third-party secrets management software previously required to rotate static API keys on virtual machines.

## 14. Failure Modes
- **The Confused Deputy Exploit**: A company integrates with an external third-party SaaS analytics tool. The company creates a cross-account role trusting the SaaS AWS account, but **omits `sts:ExternalId`**. An attacker signs up for the same SaaS tool, enters the victim's AWS Account ID and Role ARN, and uses the SaaS tool to execute queries against the victim's private AWS account!
- **The Broken Dynamic Group Matching Rule**: An engineer configures a Dynamic Group with rule `instance.compartment.id = 'ocid1.comp..123'`. Later, an administrator moves the compute instance to a new compartment. The dynamic group matching rule evaluates to false; the instance immediately loses all IAM permissions, crashing production background workers.

## 15. Troubleshooting
When workload identity or cross-account access fails:
1. **In AWS STS**:
   - Error: `Not authorized to perform sts:AssumeRole`:
     - Inspect the trust policy on the target role: does the `Principal` ARN exactly match the caller?
     - Does the caller's identity policy explicitly grant `sts:AssumeRole` on the target role ARN?
     - Is the `sts:ExternalId` string an exact, case-sensitive match?
2. **In OCI Instance Principals**:
   - Error: `404 Not Authorized Or Not Found`:
     - Verify local metadata service access: `curl -H "Authorization: Bearer Oracle" http://169.254.169.254/opc/v2/instance/`.
     - Inspect the Dynamic Group matching rule: does it match the instance's current OCID or compartment OCID?

## 16. Common Mistakes
- **Embedding AWS Access Keys in EC2 User Data**: Writing `export AWS_ACCESS_KEY_ID=AKIA...` in `cloud-init` scripts. User data is readable in plaintext by anyone with `ec2:DescribeInstances` access. Always use **IAM Instance Profiles** or **OCI Instance Principals**.
- **Failing to Enforce IMDSv2**: Leaving EC2 instances on legacy IMDSv1, leaving the host vulnerable to SSRF credential theft.

## 17. Trade-offs
| Machine Identity Model | Security Posture | Operational Overhead | Portability |
| :--- | :--- | :--- | :--- |
| **Static API Keys** | **Terrible (High leak risk)**| High (Manual rotation) | Runs anywhere |
| **IAM Instance Profiles / Dynamic Groups**| **High (Zero secrets on disk)**| Low (Automatic rotation) | Bound to AWS/OCI compute |
| **IRSA / OCI Workload Identity**| **Highest (Cryptographic pod token)**| Moderate (OIDC setup) | Kubernetes native |
| **Cross-Account STS (External ID)**| High (Confused deputy immune)| Moderate | Multi-account standard |

## 18. Interview Questions
1. *What is the Confused Deputy problem in cross-account cloud authorization, and how does the `sts:ExternalId` condition mathematically prevent it in AWS IAM?*
2. *Explain how OCI Instance Principals and Dynamic Groups work under the hood. How does a compute instance authenticate to OCI APIs without any static credentials or configuration files?*
3. *A Kubernetes pod needs to access an Amazon S3 bucket. Compare the security architecture of passing AWS credentials via Kubernetes Secrets versus using IAM Roles for Service Accounts (IRSA).*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "The **Confused Deputy Problem** is a foundational security flaw in multi-tenant systems where an authorized intermediary service (the 'deputy') is tricked by an attacker into performing unauthorized actions against a victim's resources:
>
> 1. **The Vulnerability Scenario**:
>    - Suppose a third-party SaaS company (ExampleCorp, AWS Account `111122223333`) provides automated cloud cost optimization.
>    - To let ExampleCorp scan your resources, your company (VictimCorp, AWS Account `444455556666`) creates an IAM Role `CostScannerRole` with a trust policy allowing `arn:aws:iam::111122223333:root` to call `sts:AssumeRole`.
>    - An attacker registers their own account on ExampleCorp. In the ExampleCorp dashboard, the attacker inputs **VictimCorp's Account ID and Role ARN** (`arn:aws:iam::444455556666:role/CostScannerRole`).
>    - When ExampleCorp's backend runs, it uses its own AWS credentials to assume VictimCorp's role. Because the trust policy only checks ExampleCorp's Account ID, **AWS STS grants access!** ExampleCorp (the confused deputy) scans VictimCorp's private infrastructure and displays the confidential data on the attacker's dashboard.
>
> 2. **The Cryptographic Solution: `sts:ExternalId`**:
>    - To eliminate this exploit, VictimCorp generates a unique, unguessable, cryptographically secure identifier (e.g., `VictimCorp-UUID-4921`) and shares it exclusively with ExampleCorp during onboarding.
>    - VictimCorp updates the IAM Role trust policy to enforce a mandatory condition:
>      ```json
>      "Condition": {
>        "StringEquals": { "sts:ExternalId": "VictimCorp-UUID-4921" }
>      }
>      ```
>    - When ExampleCorp calls `sts:AssumeRole`, it must supply this specific customer's `ExternalId` parameter.
>    - If the attacker attempts to configure VictimCorp's Role ARN in the attacker's ExampleCorp dashboard, ExampleCorp passes the *attacker's* external ID, STS evaluates the condition as false, and rejects the request with `AccessDenied`, completely closing the confused deputy vulnerability."

## 20. Hands-on Exercise
**Objective**: Configure and verify an OCI Dynamic Group with Instance Principals using OCI CLI inside a running VM.

### Verification Steps
1. Create a Dynamic Group matching the instance OCID.
2. Attach an IAM policy granting the Dynamic Group read access to Object Storage.
3. SSH into the compute instance. Do NOT configure `~/.oci/config` or any API keys.
4. Run an OCI CLI command using the `--auth instance_principal` flag:
   ```bash
   oci os ns get --auth instance_principal
   ```
5. Confirm that the command returns the tenancy namespace successfully, proving that the instance authenticated directly to the OCI control plane via hardware-bound X.509 certificates with zero static secrets.
