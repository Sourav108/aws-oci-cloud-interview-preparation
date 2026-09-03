# 02. Cluster Networking CNI & Pod Identity Federation

## 1. Problem
In traditional self-hosted Kubernetes, pod networking relies on software overlay networks (like Flannel or Calico) using VXLAN/IPIP encapsulation. While overlays hide pod IP addressing from the physical network, they introduce double-encapsulation CPU overhead, obscure pod traffic from cloud network firewalls, and make direct pod-to-cloud-database communication impossible without Complex SNAT. In hyperscale public clouds, cloud providers integrate pod networking directly into the virtual cloud network fabric (AWS VPC CNI and OCI VCN CNI). However, this creates a severe architectural problem: **Pod IP Address Exhaustion**. If a cluster runs 2,000 pods and each pod consumes a real, routable private IP from your subnet, subnets exhaust their address space in hours. Furthermore, running workloads without cryptographically bound pod identity forces teams to dangerously share monolithic EC2/Compute node IAM roles across all pods.

## 2. Cloud Concept
### Native Cloud CNI vs. Overlay Networking
- **Native Cloud CNI (AWS VPC CNI & OCI VCN CNI)**:
  - Pods are **first-class citizens** on the VPC/VCN.
  - The CNI attaches Elastic Network Interfaces (ENIs/VNICs) to worker nodes and assigns real, private IPv4/IPv6 addresses from the subnet CIDR block directly to each pod's network namespace.
  - *Benefits*: Zero encapsulation CPU overhead; line-rate network performance; Security Groups and NSGs can be applied directly to individual pods; VPC Flow Logs capture pod traffic natively.
  - *The Architectural Trade-off*: Severe pressure on subnet IPv4 address space.

### Pod Density & The IP Exhaustion Problem
- Every EC2 instance type has a hard hardware ceiling on:
  $$\text{Max Pods per Node} = (\text{Number of ENIs} \times (\text{IPv4 Addresses per ENI} - 1)) + 2$$
  On an `m5.large` (3 ENIs, 10 IPs per ENI), the host can run at most $(3 \times 9) + 2 = \mathbf{29\text{ pods}}$, regardless of how much spare CPU or memory remains!
- **AWS Solution: ENI Prefix Delegation**:
  - Instead of allocating individual `/32` IP addresses to an ENI slot, AWS allocates an entire **/28 IPv4 prefix** (16 consecutive IP addresses) to each slot `[Doc: Amazon VPC CNI Prefix Delegation, checked 2026-09-03]`.
  - Dramatically increases pod density (an `m5.large` jumps from 29 pods to **110 pods**) while reducing IP allocation API calls to the EC2 control plane.

### Pod Identity Federation: IRSA & OCI Workload Identity
In production Kubernetes, container pods must never share the node's underlying IAM role. If Pod A is a public web frontend and Pod B processes financial transactions, Pod A must not have permissions to read financial databases:
- **IAM Roles for Service Accounts (IRSA in AWS)**:
  - Uses an OpenID Connect (OIDC) identity provider associated with the EKS cluster.
  - Kubernetes injects a cryptographically signed JSON Web Token (JWT) into the pod's filesystem.
  - The AWS SDK calls `sts:AssumeRoleWithWebIdentity`, presenting the JWT. AWS STS verifies the token against the cluster's public OIDC keys and returns temporary, scoped IAM credentials.
- **EKS Pod Identity (Modern Alternative)**:
  - Deploys a native daemon agent on the node that handles token interception directly, simplifying IAM trust policies and eliminating OIDC URL maintenance.
- **OCI Workload Identity**:
  - The direct OCI equivalent: OKE clusters federate Kubernetes Service Account tokens with the OCI IAM service via OIDC `[Doc: OKE Workload Identity, checked 2026-09-03]`.
  - Enables pods to assume OCI IAM policies directly without storing API signing keys or user tokens inside containers.

## 3. Mental Model
Think of cloud pod networking and identity as hotel keycards and telephone extensions:
- **Overlay Networking** is a hotel switchboard where all outside calls ring the front desk (Node), and the operator manually transfers the call to your room extension (overlay NAT).
- **Native Cloud CNI** gives every individual hotel room its own direct, public-facing telephone number connected to the city telecom grid.
- **Prefix Delegation** is the telecom company assigning the hotel block numbers in batches of 16 numbers at a time, rather than wiring one phone line at a time.
- **IRSA / Workload Identity** is a smart keycard issued to the guest that only unlocks Room 304 and the gym, preventing the guest from walking into the hotel manager's private accounting office.

## 4. Architecture Diagram
```text
┌────────────────────────────────────────────────────────────────────────┐
│                        AWS VPC CNI & IRSA ARCHITECTURE                 │
│                                                                        │
│   EC2 WORKER NODE (m6i.xlarge) - Subnet: 10.0.1.0/24                   │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ PRIMARY ENI (10.0.1.10 - Node Host IP)                         │   │
│   ├────────────────────────────────────────────────────────────────┤   │
│   │ SECONDARY ENI WITH PREFIX DELEGATION:                          │   │
│   │ Prefix Slot 1: 10.0.1.32/28 (16 IPs)                           │   │
│   │   ├──► Pod A (Orders Service): 10.0.1.33                       │   │
│   │   │    * Injected Token: /var/run/secrets/eks.amazonaws.com/   │   │
│   │   │    * ServiceAccount: orders-sa                             │   │
│   │   │    * Assumes IAM Role: arn:aws:iam::123:role/OrdersRole    │   │
│   │   │                                                            │   │
│   │   └──► Pod B (Billing Service): 10.0.1.34                      │   │
│   │        * ServiceAccount: billing-sa                            │   │
│   │        * Assumes IAM Role: arn:aws:iam::123:role/BillingRole   │   │
│   └────────────────────────────────────────────────────────────────┘   │
│                                  │                                     │
│                                  ▼ Validates JWT via OIDC              │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ AWS STS / OCI IAM FEDERATION ENGINE                            │   │
│   │ * Validates Cluster OIDC Public Keys                           │   │
│   │ * Issues 1-Hour Ephemeral Scoped Access Credentials            │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS Amazon EKS:
- **AWS VPC CNI Configuration Knobs**:
  - `WARM_IP_TARGET`: Specifies the number of free, unassigned IP addresses the CNI daemon (`aws-node`) keeps pre-allocated on the node to accelerate pod spin-up.
  - `ENABLE_PREFIX_DELEGATION = "true"`: Enables allocating `/28` IPv4 slices per ENI slot.
  - `WARM_PREFIX_TARGET`: Specifies how many spare `/28` prefixes to keep warm.
- **Custom Networking in VPC CNI**:
  - When the primary VPC subnet CIDR is running out of IP addresses, enable **Custom Networking**.
  - Assign a secondary non-routable CIDR (e.g., `100.64.0.0/16` CGNAT space) to the VPC.
  - Pods are assigned IPs strictly from the secondary `100.64.0.0/16` subnets, completely shielding corporate RFC 1918 subnets from pod address consumption.
- **Security Groups for Pods**:
  - Allows attaching an AWS Security Group **directly to a Kubernetes Pod** using the `SecurityGroupPolicy` Custom Resource Definition (CRD), achieving microsegmentation at the pod level!

## 6. OCI Implementation
In Oracle Cloud Infrastructure Container Engine for Kubernetes (OKE):
- **OCI VCN-Native Pod Networking CNI**:
  - OKE provides native VCN pod networking where every pod receives an IP address directly from an OCI VCN Regional Subnet `[Doc: OKE VCN-Native Networking, checked 2026-09-03]`.
  - **Regional Subnet Advantage**: Pod IPs belong to a Regional Subnet spanning all Availability Domains and Fault Domains. Pods in AD-1 and AD-2 can communicate with zero inter-AD routing barriers.
  - **Network Security Groups (NSGs) for Pods**: Pods can be assigned to OCI NSGs directly, enforcing hardware-offloaded SmartNIC firewall filtering on pod traffic.
- **OCI Flannel Overlay CNI (Alternative)**:
  - For organizations with strict, unchangeable IPv4 subnet constraints, OKE provides the native **Flannel Overlay CNI**.
  - Under Flannel, pods use a private overlay address space (e.g., `10.244.0.0/16`) that never consumes VCN subnet IPs. Outbound traffic to the VCN or internet is translated via host-level SNAT.
- **OCI Workload Identity Deep Dive**:
  - Works by linking an OCI IAM Policy directly to a Kubernetes Service Account namespace and name.
  - Syntax:
    ```sql
    Allow any-user to manage objects in compartment Prod
    where ALL {
      request.principal.type = 'workload',
      request.principal.service_account = 'data-writer-sa',
      request.principal.namespace = 'production'
    }
    ```
  - Eliminates the need to maintain AWS-style IAM Role ARNs or complex trust relationships; OCI evaluates service account claims natively in its authorization engine.

## 7. Configuration
Comparing pod identity configuration in Terraform across AWS and OCI:

### AWS IAM Roles for Service Accounts (IRSA) (Terraform)
```hcl
# 1. IAM Role with OIDC AssumeRole Trust Policy
resource "aws_iam_role" "orders_app" {
  name = "orders-service-irsa-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = var.cluster_oidc_provider_arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            # Strictly locks role to the specific K8s Service Account!
            "${var.cluster_oidc_issuer}:sub" = "system:serviceaccount:production:orders-sa"
            "${var.cluster_oidc_issuer}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })
}

# 2. Kubernetes Service Account with IRSA Annotation
resource "kubernetes_service_account" "orders_sa" {
  metadata {
    name      = "orders-sa"
    namespace = "production"
    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.orders_app.arn
    }
  }
}
```

### OCI Workload Identity Policy (Terraform)
```hcl
# OCI Workload Identity: Linking K8s Service Account directly to OCI IAM Policy
resource "oci_identity_policy" "workload_policy" {
  compartment_id = var.compartment_id
  name           = "orders-workload-identity-policy"
  description    = "Grants Orders Pod access to Object Storage"

  statements = [
    "Allow any-user to manage objects in compartment Production where ALL { request.principal.type = 'workload', request.principal.cluster_id = '${var.oke_cluster_id}', request.principal.namespace = 'production', request.principal.service_account = 'orders-sa' }"
  ]
}
```

## 8. Data Flow
```text
IRSA / Workload Identity Token Exchange Sequence:
1. Kubernetes Pod starts with spec.serviceAccountName: orders-sa
2. Kubelet projects signed OIDC JWT token into container volume:
   /var/run/secrets/eks.amazonaws.com/serviceaccount/token
3. Application code initializes AWS/OCI SDK:
   - Reads JWT token from disk.
   - Executes STS / IAM call: AssumeRoleWithWebIdentity(Token, RoleARN).
4. Cloud STS / OCI IAM verifies token:
   - Queries EKS / OKE OIDC discovery document (JWKS keys).
   - Validates signature, audience, and expiration.
   - Validates namespace and service account name match IAM policy condition.
5. STS returns temporary AWS credentials (AccessKey, SecretKey, SessionToken).
6. Application queries DynamoDB / S3 using ephemeral credentials.
```

## 9. Security
- **Eliminating Node-Level IAM Sharing**:
  - Before IRSA, pods inherited the IAM role attached to the worker EC2 instance. Any pod on that node could execute any action granted to any other pod on that node.
  - IRSA and OCI Workload Identity enforce true cryptographic microsegmentation: **every pod has its own unique, isolated identity**.
- **Blocking IMDSv1 on Worker Nodes**:
  - If a container queries the host Instance Metadata Service (`http://169.254.169.254/latest/meta-data/`), it might attempt to steal the worker node's host credentials.
  - Enforce **IMDSv2** with `http_put_response_hop_limit = 1`. Because the container runtime is a network hop away from the host network namespace, the hypervisor drops the IMDS request, preventing credential theft.

## 10. Reliability
- **Avoiding Prefix Delegation Subnet Fragmentation**:
  - When Prefix Delegation is enabled, AWS allocates contiguous `/28` blocks.
  - If a subnet is heavily fragmented with individual `/32` IP allocations, the CNI cannot find a contiguous 16-IP block, causing node startup to fail with `InsufficientFreePrefixes`.
  - *Reliability Rule*: Enable Prefix Delegation on **newly created, unfragmented subnets**.

## 11. Scaling
- **VPC CNI Warm IP Tuning**:
  - In a 500-node cluster, if `WARM_IP_TARGET = 10`, the cluster holds 5,000 idle IP addresses hostage in the subnet!
  - For large clusters, set `WARM_IP_TARGET` to small numbers (e.g., 2) or use `WARM_PREFIX_TARGET = 1` to prevent wasting subnet address capacity.

## 12. Observability
- **Monitoring CNI Metrics**:
  - Prometheus metrics exposed by `aws-node` daemon on port 61678:
    - `awscni_assigned_ip_addresses`: Number of IPs currently in use by pods.
    - `awscni_total_ip_addresses`: Total allocated IPs held by the node.
    - `awscni_eni_allocated`: Current number of attached ENIs.

## 13. Cost
- Both AWS VPC CNI and OCI VCN CNI are free software components.
- Cost savings stem from **Prefix Delegation**: increasing pod density per node from 29 to 110 pods allows organizations to run fewer, larger EC2 instances, reducing total compute spend by 20%–35%.

## 14. Failure Modes
- **The "No Space Left on Device" IP Exhaustion Panic**: A cluster scales to 1,500 pods. The `/21` worker subnet ($2,043$ usable IPs) runs out of addresses. The CNI fails to allocate pod interfaces; pods hang indefinitely in `ContainerCreating` with events: `Failed to allocate network for pod: no free IP addresses available`.
- **The Expired OIDC Provider Thumbprint**: An administrator updates the TLS certificate on an enterprise EKS OIDC provider but forgets to update the IAM OIDC Provider Thumbprint in AWS IAM. Every single pod across the entire cluster using IRSA fails STS authentication simultaneously, throwing `InvalidIdentityToken` and collapsing production.

## 15. Troubleshooting
When pods fail to acquire IP addresses or fail IRSA authentication:
1. **Check CNI DaemonSet Logs**:
   ```bash
   kubectl logs -n kube-system -l k8s-app=aws-node --tail=100
   ```
   Look for `Failed to allocate IP` or EC2 API rate limit errors (`RequestLimitExceeded`).
2. **Inspect Projected Service Account Token**:
   - Exec into pod: inspect `/var/run/secrets/eks.amazonaws.com/serviceaccount/token`.
   - Decode JWT on `jwt.io`: verify the `sub` claim matches `system:serviceaccount:<namespace>:<serviceaccount>`.
3. **Verify IAM Role Trust Policy**: Does the condition block exactly match the OIDC issuer URL and Service Account name?

## 16. Common Mistakes
- **Forgetting OIDC Issuer URL Prefix**: In IRSA trust policies, omitting the `:sub` or `:aud` suffix on the condition key. The condition evaluates to false, silently rejecting STS requests.
- **Running Out of Secondary IPs Without Prefix Delegation**: Buying large `c6i.32xlarge` instances with 128 vCPUs but forgetting to enable Prefix Delegation. The host is capped at ~50 pods due to ENI slot limits, wasting 70% of the instance's compute power.

## 17. Trade-offs
| CNI Strategy | Network Performance | Subnet IP Consumption | Security Posture | Best Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **AWS VPC CNI (Default)** | Line-rate (Zero overlay) | High (1 IP per pod) | High (Pod SGs supported) | Standard AWS enterprise clusters |
| **AWS VPC CNI (Prefix Del)**| Line-rate (Zero overlay) | Low (Batches of 16 IPs) | High (High pod density) | Large-scale EKS clusters |
| **OCI VCN-Native CNI** | Line-rate (Zero overlay) | Moderate (Regional subnet)| High (Native OCI NSGs) | Standard OKE enterprise clusters |
| **Flannel Overlay (OCI)** | Moderate (Overlay CPU tax) | **Zero VCN IPs** | Moderate (Host NAT) | Clusters in tight legacy RFC 1918 subnets |

## 18. Interview Questions
1. *What is the Pod IP Address Exhaustion problem in AWS EKS? How does ENI Prefix Delegation solve it, and what are the operational risks of enabling it on existing subnets?*
2. *Explain how IAM Roles for Service Accounts (IRSA) works at the cryptographic protocol level. Walk me through the token exchange from the pod to AWS STS.*
3. *How does OCI Workload Identity compare to AWS IRSA in terms of configuration complexity and IAM policy syntax?*

## 19. Interview Answer
**Exemplary Answer to Question 2**:
> "IAM Roles for Service Accounts (IRSA) uses **OpenID Connect (OIDC) identity federation** to deliver least-privilege, ephemeral cloud credentials directly to Kubernetes pods without long-lived secrets:
>
> 1. **Cluster OIDC Establishment**:
>    - When an EKS cluster is provisioned, AWS hosts a public OIDC discovery endpoint and JSON Web Key Set (JWKS) containing the cluster's public cryptographic signing keys.
>    - We create an IAM OIDC Identity Provider in AWS IAM referencing this cluster URL.
>
> 2. **Token Injection via Projected Service Account Volume**:
>    - When a pod specifies a ServiceAccount annotated with `eks.amazonaws.com/role-arn`, the Kubernetes API server generates a signed JSON Web Token (JWT).
>    - The kubelet injects this token into the pod's filesystem as a projected volume at `/var/run/secrets/eks.amazonaws.com/serviceaccount/token` and injects two environment variables: `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE`.
>
> 3. **The STS Token Exchange (`AssumeRoleWithWebIdentity`)**:
>    - The AWS SDK inside the container automatically detects these environment variables and calls AWS STS:
>      ```text
>      sts:AssumeRoleWithWebIdentity(RoleArn, WebIdentityToken, RoleSessionName)
>      ```
>    - AWS STS intercepts the call, fetches the public JWKS keys from the cluster's OIDC endpoint, and cryptographically validates the token's signature, expiration, and audience (`sts.amazonaws.com`).
>    - STS inspects the token's subject claim (`sub`):
>      `system:serviceaccount:<namespace>:<serviceaccount-name>`.
>    - STS compares this subject claim against the condition block in the target IAM Role's trust policy.
>
> 4. **Ephemeral Credential Issuance**:
>    - If the signature is valid and the condition matches, STS issues temporary, scoped AWS credentials (Access Key ID, Secret Access Key, and Session Token) valid for 1 hour.
>    - The SDK caches these credentials and auto-refreshes them before expiration.
>
> This architecture ensures that container credentials never persist on disk, never share node-level permissions, and are automatically revoked when the pod is deleted."

## 20. Hands-on Exercise
**Objective**: Configure AWS VPC CNI Prefix Delegation and verify pod IP allocation density.

### Verification Steps
1. On an active EKS cluster, enable Prefix Delegation in the `aws-node` DaemonSet:
   ```bash
   kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
   kubectl set env daemonset aws-node -n kube-system WARM_PREFIX_TARGET=1
   ```
2. Inspect worker node ENI allocations via AWS CLI:
   ```bash
   aws ec2 describe-network-interfaces --filters "Name=attachment.instance-id,Values=<node-id>" \
     --query "NetworkInterfaces[*].Ipv4Prefixes[*].Ipv4Prefix"
   ```
3. Confirm that the node has been assigned contiguous `/28` CIDR blocks (e.g., `10.0.1.32/28`).
4. Deploy 30 test pods on an `m5.large` instance and confirm that all 30 pods enter `Running` status without breaching legacy node pod limits.
