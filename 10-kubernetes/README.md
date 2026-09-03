# Module 10: Kubernetes (Cloud-Managed Lens: EKS vs. OKE)

> **Architectural Objective**: *Master cloud-managed Kubernetes control planes, node group lifecycle engineering, cloud CNI networking, and workload identity federation. Deconstruct the cloud integration layer of Amazon EKS and Oracle Cloud Infrastructure Container Engine for Kubernetes (OKE), evaluate CNI pod networking architectures (AWS VPC CNI vs. OCI VCN-Native CNI), and master cloud ingress and storage CSI integrations.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. EKS vs. OKE Control Plane & Node Groups](01-eks-vs-oke-control-plane-and-node-groups.md)** | Managed Control Plane SLA/Pricing, Managed Node Groups, Virtual Nodes (Fargate vs. OKE Virtual Nodes), Zero-Downtime Cluster Version Upgrades | Full 20-Section Deep Dive (~2,500 words) |
| **[02. Cluster Networking CNI & Pod Identity](02-cluster-networking-cni-and-pod-identity.md)** | AWS VPC CNI (Secondary IPs, Prefix Delegation) vs. OCI VCN-Native CNI, IAM Roles for Service Accounts (IRSA / Pod Identity) vs. OCI Workload Identity | Full 20-Section Deep Dive (~2,500 words) |
| **[03. Ingress Controllers & Storage CSI Drivers](03-ingress-controllers-and-storage-csi.md)** | AWS Load Balancer Controller (TargetGroupBinding) vs. OCI LB Controller, EBS/EFS CSI vs. OCI Block Volume / FSS CSI Driver | Abbreviated Kubernetes Integration Guide (~950 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **The Cloud-Managed Kubernetes Trade-off**: Why enterprises adopt EKS/OKE instead of self-hosted `kubeadm`, and how cloud providers manage etcd quorum, control plane scaling, and master node patching.
2. **CNI Pod IP Exhaustion & Prefix Delegation**: How AWS VPC CNI consumes private subnet IPs for every pod, the mechanics of ENI prefix delegation (`/28` IPv4 slices), and how OCI VCN-Native CNI leverages regional subnets.
3. **Pod Identity Federation**: How IAM Roles for Service Accounts (IRSA) and OCI Workload Identity use OpenID Connect (OIDC) federation to inject short-lived cloud credentials directly into Kubernetes service accounts without long-lived secrets.
