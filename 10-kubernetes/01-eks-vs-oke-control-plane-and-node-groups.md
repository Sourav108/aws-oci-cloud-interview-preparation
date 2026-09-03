# 01. EKS vs. OKE Control Plane & Node Groups

## 1. Problem
Managing Kubernetes control planes manually (using tools like `kubeadm` on raw virtual machines) is notoriously difficult in production. SRE teams must configure multi-master etcd consensus clustering, manage TLS certificates across API servers, handle automated control-plane failover across availability zones, and execute perilous version upgrades without losing cluster state. A single split-brain etcd failure or expired CA certificate can take down an entire corporate application fleet. Cloud-managed Kubernetes platforms—Amazon Elastic Kubernetes Service (EKS) and Oracle Cloud Infrastructure Container Engine for Kubernetes (OKE)—eliminate this burden by providing managed, SLA-backed, highly available control planes.

## 2. Cloud Concept
### Managed Kubernetes Control Plane Architecture
In both AWS EKS and OCI OKE, the cloud provider owns and operates the Kubernetes control plane:
- **API Server (`kube-apiserver`)**: Fronted by a managed cloud load balancer. Scaled horizontally across multiple Availability Zones / Fault Domains.
- **etcd Cluster**: Highly available, 3-node or 5-node distributed consensus cluster deployed across independent failure domains. Automatic continuous snapshots and disaster recovery backups managed by the cloud platform.
- **Controller Manager & Scheduler**: Continuously monitored and auto-healed.
- **Control Plane Endpoint Access**: Configurable as *Public*, *Public & Private*, or *Private Only* (where the Kubernetes API server is accessible strictly through an internal VPC/VCN private endpoint).

### Worker Node Abstractions
1. **Self-Managed Worker Nodes**:
   - Standard EC2 or OCI Compute instances running the `kubelet` and container runtime (containerd).
   - *Customer Responsibility*: Bootstrapping, joining instances to the cluster, managing AMI updates, configuring Auto Scaling Groups, and executing OS patches.
2. **Managed Node Groups (MNG)**:
   - Automated by the cloud service. Cloud platform provisions and updates the underlying Auto Scaling Group / Instance Pool.
   - Handles automated node draining (`kubectl drain`) and rolling AMI updates with zero downtime.
3. **Serverless / Virtual Nodes (AWS Fargate & OKE Virtual Nodes)**:
   - Completely eliminates worker VMs. Pods run directly on isolated microVMs / bare-metal serverless infrastructure.
   - *Zero Node Maintenance*: No node patching, no kubelet tuning, and no capacity over-provisioning `[Doc: OKE Virtual Nodes, checked 2026-09-03]`.

## 3. Mental Model
Think of cloud-managed Kubernetes as operating a modern passenger train service:
- **Self-Hosted Kubernetes (`kubeadm`)** is building your own locomotive from scratch, laying the physical steel tracks, maintaining the electrical signals, and shoveling the coal into the engine yourself.
- **Cloud-Managed Kubernetes (EKS / OKE)** is buying a first-class ticket on a high-speed electric bullet train. The railway operator (AWS/OCI) manages the locomotives, tracks, power lines, and track switches (etcd, API servers, master nodes). You simply decide which passenger cars (workload pods) attach to the train and where they travel.

## 4. Architecture Diagram
```text
CLOUD-MANAGED KUBERNETES TOPOLOGY (AWS EKS & OCI OKE):

┌────────────────────────────────────────────────────────────────────────┐
│ CLOUD-MANAGED CONTROL PLANE (Hidden in Provider VPC / Tenancy)        │
│                                                                        │
│   [kube-apiserver AZ-1]  [kube-apiserver AZ-2]  [kube-apiserver AZ-3]  │
│   └──────────────────────┬──────────────────────┴───────────────────┘  │
│                          │ Raft Consensus                              │
│   [etcd Node 1] ◄────────┼────────► [etcd Node 2] ◄────────► [etcd 3]  │
│   (Continuous Automated Snapshots to S3 / OCI Object Storage)          │
└──────────────────────────┬─────────────────────────────────────────────┘
                           │ Private Cross-VPC / VCN ENI Endpoint
                           ▼
┌────────────────────────────────────────────────────────────────────────┐
│ CUSTOMER WORKSPACE VCN / VPC (Private Worker Subnets)                 │
│                                                                        │
│   MANAGED NODE GROUP 1 (AZ-1 / AD-1)    MANAGED NODE GROUP 2 (AZ-2)    │
│   ┌────────────────────────────────┐    ┌──────────────────────────┐   │
│   │ Worker Node: EC2 / OCI VM      │    │ Worker Node: EC2 / OCI VM│   │
│   │ ┌──────────────┐┌────────────┐ │    │ ┌───────────┐┌─────────┐ │   │
│   │ │ Pod: Orders  ││ Pod: Users │ │    │ │Pod: Orders││Pod: Pay │ │   │
│   │ └──────────────┘└────────────┘ │    │ └───────────┘└─────────┘ │   │
│   │ containerd / kubelet           │    │ containerd / kubelet     │   │
│   └────────────────────────────────┘    └──────────────────────────┘   │
│                                                                        │
│   SERVERLESS VIRTUAL NODES (AWS Fargate / OKE Virtual Nodes)           │
│   ┌────────────────────────────────┐    ┌──────────────────────────┐   │
│   │ Isolated MicroVM: Pod Analytics│    │ Isolated MicroVM: Batch  │   │
│   └────────────────────────────────┘    └──────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS Amazon EKS:
- **Control Plane SLA & Pricing**:
  - AWS charges **\$0.10 per hour per EKS cluster** ($\approx \$72.00/\text{month}$) `[Doc: Amazon EKS Pricing, checked 2026-09-03]`.
  - Backed by an uptime SLA of **99.95%**.
- **Managed Node Groups (MNG)**:
  - Automates EC2 provisioning using Launch Templates and Auto Scaling Groups.
  - Native integration with **Karpenter**: an open-source, high-performance Kubernetes node autoscaler that bypasses rigid Auto Scaling Groups, observing pending unschedulable pods and launching right-sized EC2 instances in under 45 seconds.
- **EKS Extended Support**:
  - AWS charges an additional **\$0.60 per cluster-hour** ($\approx \$432/\text{month}$) for clusters running older Kubernetes versions that have entered the 12-month Extended Support window `[Doc: Amazon EKS Extended Support, checked 2026-09-03]`. This creates a massive financial incentive to keep clusters updated.

## 6. OCI Implementation
In Oracle Cloud Infrastructure Container Engine for Kubernetes (OKE):
- **Control Plane Tiers & Financial Advantage**:
  - **Basic Clusters**: **100% Free**. OCI charges **\$0.00 for the control plane**. Customers pay strictly for the worker compute instances.
  - **Enhanced Clusters**: Billed at **\$0.10 per hour** ($\approx \$72.00/\text{month}$), providing a financially backed **99.95% SLA**, Virtual Node support, and automated add-on management `[Doc: OKE Pricing and SLA, checked 2026-09-03]`.
- **OKE Virtual Nodes**:
  - OCI's native serverless Kubernetes node abstraction.
  - Pods run on fully isolated, hypervisor-hardened virtual nodes managed entirely by Oracle.
  - Sized dynamically based on pod resource requests; billed on per-second OCPU/memory consumption.
- **Self-Healing Node Pools**:
  - OKE node pools monitor worker node health automatically. If a node fails kubelet heartbeats, OKE drains the node and provisions a fresh worker instance on a healthy Fault Domain.
- **Native OCI Add-on Management**:
  - OKE provides centralized console and Terraform management for Kubernetes add-ons: CoreDNS, kube-proxy, OCI VCN CNI, and CSI storage drivers.

## 7. Configuration
Comparing managed Kubernetes cluster provisioning in Terraform across AWS and OCI:

### AWS EKS Cluster with Managed Node Group (Terraform)
```hcl
# AWS EKS Cluster
resource "aws_eks_cluster" "prod_eks" {
  name     = "prod-eks-cluster"
  role_arn = aws_iam_role.eks_cluster_role.arn
  version  = "1.30"

  vpc_config {
    subnet_ids              = var.private_subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = true # Controlled via CIDR whitelist
  }

  depends_on = [aws_iam_role_policy_attachment.eks_cluster_policy]
}

# AWS Managed Node Group
resource "aws_eks_node_group" "general_nodes" {
  cluster_name    = aws_eks_cluster.prod_eks.name
  node_group_name = "general-workload-mng"
  node_role_arn   = aws_iam_role.eks_node_role.arn
  subnet_ids      = var.private_subnet_ids

  scaling_config {
    desired_size = 3
    max_size     = 10
    min_size     = 2
  }

  instance_types = ["m6i.xlarge"]
  capacity_type  = "ON_DEMAND"

  update_config {
    max_unavailable = 1 # Ensures zero downtime rolling upgrades!
  }
}
```

### OCI OKE Enhanced Cluster with Node Pool (Terraform)
```hcl
# OCI OKE Enhanced Cluster
resource "oci_containerengine_cluster" "prod_oke" {
  compartment_id     = var.compartment_id
  name               = "prod-oke-cluster"
  kubernetes_version = "v1.30.1"
  vcn_id             = var.vcn_id

  # Enhanced cluster type for 99.95% SLA
  type = "ENHANCED_CLUSTER"

  endpoint_config {
    is_public_ip_enabled = false # Strict private API endpoint!
    subnet_id            = var.control_plane_subnet_id
  }

  options {
    service_lb_subnet_ids = [var.public_lb_subnet_id]
  }
}

# OCI OKE Managed Node Pool
resource "oci_containerengine_node_pool" "app_pool" {
  cluster_id         = oci_containerengine_cluster.prod_oke.id
  compartment_id     = var.compartment_id
  name               = "app-worker-pool"
  kubernetes_version = "v1.30.1"
  node_shape         = "VM.Standard.E5.Flex"

  node_shape_config {
    ocpus         = 2
    memory_in_gbs = 16
  }

  node_config_details {
    size = 3
    placement_configs {
      availability_domain = var.availability_domain
      subnet_id           = var.worker_subnet_id # Regional Subnet!
    }
  }
}
```

## 8. Data Flow
```text
The Step-by-Step Rolling Cluster Upgrade Protocol:
1. SRE updates cluster version: v1.29 ──► v1.30 via Terraform/Console.
2. Cloud Control Plane Upgrade:
   * AWS/OCI upgrades etcd schema.
   * Rolling restart of kube-apiserver, controller-manager, and scheduler.
   * Workload pods on worker nodes continue running uninterrupted!
3. Add-on Upgrades: CoreDNS, kube-proxy, and CNI drivers upgraded to match v1.30.
4. Managed Node Group Rolling Update:
   * Node Pool launches NEW v1.30 Worker Node (Node D).
   * Node Pool issues 'kubectl cordon Node A' (blocks new pods).
   * Node Pool issues 'kubectl drain Node A --ignore-daemonsets --delete-emptydir-data'.
   * Pods evict gracefully, rescheduling onto Node D according to PodDisruptionBudgets.
   * Node A terminated; process repeats for Node B and C with ZERO customer downtime.
```

## 9. Security
- **API Server Endpoint Hardening**:
  - Never leave the Kubernetes API server publicly accessible to `0.0.0.0/0`.
  - Set `endpoint_private_access = true` and `endpoint_public_access = false`. SREs and CI/CD runners access the cluster through an internal Bastion, AWS VPN, or OCI Bastion Service over private network endpoints.
- **Control Plane Audit Logging**:
  - Enable all EKS control plane log types: `api`, `audit`, `authenticator`, `controllerManager`, `scheduler`. Stream logs directly to CloudWatch or OCI Logging to detect unauthorized RBAC privilege escalation.

## 10. Reliability
- **Pod Disruption Budgets (PDB)**:
  - Rolling node upgrades can evict pods rapidly.
  - Enforce `PodDisruptionBudget` manifests on all production deployments:
    ```yaml
    apiVersion: policy/v1
    kind: PodDisruptionBudget
    metadata:
      name: orders-pdb
    spec:
      minAvailable: 2
      selector:
        matchLabels:
          app: orders
    ```
    This physically prevents the cloud node drainer from evicting a pod if doing so would drop available healthy replicas below 2.

## 11. Scaling
- **Karpenter vs. Cluster Autoscaler**:
  - The legacy **Cluster Autoscaler** scales by modifying EC2 Auto Scaling Groups, which is slow (2–5 minutes) and locked to predetermined instance shapes.
  - **Karpenter** acts directly within the Kubernetes scheduler: it analyzes pending pod CPU/memory/GPU requests, evaluates current EC2 Spot/On-Demand spot market pricing, launches the exact right-sized instance, and binds the node in **under 45 seconds**.

## 12. Observability
- **Prometheus & OpenTelemetry Integration**:
  - AWS Managed Service for Prometheus (AMP) and Managed Grafana (AMG).
  - OCI Application Performance Monitoring (APM) and OCI Container Monitoring Agent.

## 13. Cost
- **EKS vs. OKE Cluster Economics (Annual Spend)**:
  - 10 EKS Clusters (AWS): $10 \times \$0.10/\text{hr} \times 8,760\text{ hrs} = \mathbf{\$8,760/\text{year}}$ purely for control planes.
  - 10 OKE Basic Clusters (OCI): **\$0.00/year** (100% Free control planes!).
  - 10 OKE Enhanced Clusters (OCI): **\$8,760/year** with 99.95% SLA.
- **The EKS Extended Support Trap**: Failing to upgrade an EKS cluster past 14 months increases the hourly fee from \$0.10/hr to **\$0.70/hr** (\$504/month per cluster!).

## 14. Failure Modes
- **The Aggressive Node Drain Deadlock**: An engineer triggers a node group upgrade on a cluster running a single replica of a legacy service without a PodDisruptionBudget. The node drainer forcefully kills the pod. Traffic to that microservice drops to 100% error rate until the new node initializes 3 minutes later.
- **Subnet IP Exhaustion Blocking Node Scaling**: An EKS cluster attempts to scale out from 10 to 50 nodes. Because each worker node consumes dozens of IPs for its pods via the AWS VPC CNI, the worker subnet exhausts all available IPv4 addresses. New EC2 instances launch but fail to obtain an IP, hanging in the `NotReady` state.

## 15. Troubleshooting
When worker nodes fail to join an EKS or OKE cluster:
1. **Verify Outbound API Connectivity**:
   - Can the worker node reach the API server private endpoint on TCP port 443?
2. **Inspect Node Bootstrap Logs**:
   - Connect via SSM/Serial Console: check `/var/log/cloud-init-output.log` and `/var/log/messages`.
   - Look for `kubelet` crash logs: `failed to run Kubelet: failed to get node info: node "ip-10-0-1-50" not found`.
3. **Verify IAM / Identity Authentication**:
   - In EKS: Check the `aws-auth` ConfigMap in `kube-system` (or modern EKS Access Entries). Does the worker node IAM role have permission to join the cluster?

## 16. Common Mistakes
- **Neglecting Cluster Version Upgrades**: Treating a Kubernetes cluster like a long-lived database server. Kubernetes releases minor versions every 4 months, and cloud providers deprecate versions after 14 months. Cluster version upgrades must be automated into quarterly engineering roadmaps.
- **Running Ingress on Public Master Endpoints**: Deploying public load balancers into private control plane subnets instead of dedicated public worker subnets.

## 17. Trade-offs
| Feature | AWS EKS | OCI OKE |
| :--- | :--- | :--- |
| **Control Plane Cost** | \$0.10/hr (Always billed) | Free (Basic) or \$0.10/hr (Enhanced) |
| **Serverless Pods** | AWS Fargate (MicroVMs) | OKE Virtual Nodes |
| **Node Autoscaling** | Karpenter & Cluster Autoscaler | OCI Cluster Autoscaler & Flexible Shapes |
| **VCN/VPC CNI Model** | ENI Secondary IPs & Prefix Delegation | Native VCN CNI & Flannel Overlay |

## 18. Interview Questions
1. *Walk me through the exact step-by-step procedure for executing a zero-downtime Kubernetes version upgrade on an EKS or OKE cluster serving live production traffic.*
2. *Why does AWS charge \$0.60/hr extra for older EKS versions, and how do you architect an organizational CI/CD platform to prevent falling into Extended Support billing?*
3. *Compare the control plane architecture, pricing, and SLA guarantees of AWS EKS versus OCI OKE.*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "Executing a zero-downtime Kubernetes cluster version upgrade on AWS EKS or OCI OKE requires a strictly phased, 4-stage operational protocol:
>
> 1. **Phase 1: Pre-Upgrade API Compatibility Verification**:
>    - Before touching infrastructure, I use tools like `pluto` or `kubent` to scan all Helm charts, GitOps manifests, and custom controllers for deprecated API versions that will be removed in the target Kubernetes version (e.g., `autoscaling/v2beta1` to `autoscaling/v2`).
>    - I verify that all deployments enforce a calibrated **PodDisruptionBudget (PDB)** ensuring at least $N-1$ replicas remain available during node evictions.
>
> 2. **Phase 2: Managed Control Plane Upgrade**:
>    - I trigger the control plane version upgrade (e.g., v1.29 to v1.30) via Terraform or CLI.
>    - AWS EKS / OCI OKE executes a rolling upgrade of the `kube-apiserver` and underlying etcd schema across three availability zones.
>    - Crucially: **existing worker nodes and running pods continue processing customer traffic completely uninterrupted** during the control plane upgrade.
>
> 3. **Phase 3: Managed Add-on Upgrades**:
>    - Once the API server is healthy on v1.30, I upgrade the cluster core add-ons to version-matched releases:
>      1. `kube-proxy`
>      2. CoreDNS
>      3. Cloud CNI plugin (AWS VPC CNI / OCI VCN CNI)
>      4. Storage CSI drivers
>
> 4. **Phase 4: Worker Node Group Rolling Upgrade**:
>    - If using Managed Node Groups, I update the Launch Template AMI ID to the v1.30 optimized AMI and initiate a rolling update with `max_unavailable = 1`.
>    - The cloud control plane provisions a new v1.30 worker node. Once healthy, it issues `kubectl cordon` on an old v1.29 node, followed by `kubectl drain`.
>    - Pods gracefully terminate according to their `terminationGracePeriodSeconds` and reschedule onto the new node.
>    - This cycle repeats sequentially across all worker nodes, delivering a 100% seamless upgrade with zero dropped customer requests."

## 20. Hands-on Exercise
**Objective**: Deploy an OKE or EKS cluster in Terraform and inspect managed control plane health via CLI.

### Verification Steps
1. Provision an EKS / OKE cluster using Terraform with private endpoint access enabled.
2. Configure local `kubectl` credentials:
   ```bash
   # AWS EKS
   aws eks update-kubeconfig --region us-east-1 --name prod-eks-cluster
   # OCI OKE
   oci ce cluster create-kubeconfig --cluster-id <cluster-id> --file ~/.kube/config
   ```
3. Verify control plane component status:
   ```bash
   kubectl get nodes -o wide
   kubectl get pods -n kube-system
   ```
4. Verify that worker nodes display `STATUS: Ready` and report the correct cloud provider kernel and containerd versions.
