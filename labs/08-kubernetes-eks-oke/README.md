# Lab 08: Managed Kubernetes Ingress, Networking & Pod Security (EKS & OKE)

---

## 1. Goal & Core Concept

The objective of this hands-on lab is to deploy, configure, and troubleshoot managed Kubernetes container workloads across **Amazon Elastic Kubernetes Service (EKS)** and **Oracle Cloud Infrastructure Container Engine for Kubernetes (OKE)**.

### Core Architectural Concepts Tested
- **Container Network Interface (CNI)**: AWS VPC CNI (secondary IP allocation) vs. OCI VCN-Native CNI (direct VNIC per pod).
- **Ingress Controller Automation**: AWS Load Balancer Controller (Target Type `ip`) vs. OCI Native Ingress Controller.
- **Microsegmentation with Kubernetes NetworkPolicy**: Enforcing default-deny egress and ingress rules.
- **NodePort vs. LoadBalancer vs. Ingress**: Selecting optimal ingress patterns for throughput and cost.

---

## 2. Cost Warning & Resource Estimate

> [!NOTE]
> **ESTIMATED HOURLY RUN COST: `< $0.35 / hour`**
>
> | Cloud Provider | Resource Component | Quantity | Hourly Rate | Run Cost (2 Hours) |
> | :--- | :--- | :---: | :---: | :---: |
> | **AWS** | EKS Cluster Control Plane `[Doc: EKS Pricing, checked 2026]` | 1 Cluster | $0.10 / hr | $0.20 |
> | **AWS** | EC2 Worker Node (`t4g.small` or `t3.medium`) | 2 Nodes | $0.0336 / hr | $0.067 |
> | **OCI** | OKE Cluster Control Plane `[Doc: OKE Pricing, checked 2026]` | 1 Cluster | $0.00 (Basic Free) | $0.00 |
> | **OCI** | Worker Node (`VM.Standard.E5.Flex` 1 OCPU / 8 GB) | 2 Nodes | $0.04 / hr | $0.08 |
>
> *Total Estimated Lab Cost for a 2-hour session: **$0.347 USD**.*
> *In No-Live-Account Fallback Mode, cost is **$0.00**.*

---

## 3. Architecture Diagram

```
========================================================================================================================
                          MANAGED KUBERNETES NETWORKING & INGRESS TOPOLOGY
========================================================================================================================

                                        [ External Client Request ]
                                                     │
                                                     ▼
                             [ Ingress Load Balancer: ALB / OCI Flexible LB ]
                                                     │
                                                     ▼ (Target Type: IP / Direct Pod CNI Routing)
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  KUBERNETES CLUSTER (EKS / OKE) - 10.0.0.0/16
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   INGRESS CONTROLLER (AWS LB Controller / OCI Ingress Controller)
   ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
   │ Pod: Nginx Web App (IP: 10.0.10.45)               │ Pod: Backend API App (IP: 10.0.20.78)                        │
   │ Namespace: frontend                               │ Namespace: backend                                           │
   │                                                   │                                                              │
   │ [ NetworkPolicy: Default Deny All ]               │ [ NetworkPolicy: Allow Ingress from Frontend Pods Only ]     │
   │ - Egress allowed only to Backend on port 8080     │ - Egress to public internet strictly blocked                 │
   └───────────────────────────────────────────────────┴──────────────────────────────────────────────────────────────┘
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
```

---

## 4. Prerequisites

1. `kubectl` CLI installed and configured.
2. Terraform CLI v1.8+.

---

## 5. Infrastructure Code (Terraform & Manifests)

### 5.1 AWS EKS Node Group Configuration (`aws_eks.tf`)

```hcl
# AWS Reference Implementation: EKS Managed Node Group
resource "aws_eks_node_group" "lab_nodes" {
  cluster_name    = var.eks_cluster_name
  node_group_name = "lab08-worker-nodes"
  node_role_arn   = var.node_role_arn
  subnet_ids      = [var.private_subnet_az1, var.private_subnet_az2]

  scaling_config {
    desired_size = 2
    max_size     = 4
    min_size     = 1
  }

  instance_types = ["t4g.small"]
  capacity_type  = "ON_DEMAND"
}
```

### 5.2 Kubernetes Ingress & NetworkPolicy (`app_manifests.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: frontend
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend-api
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
    ports:
    - protocol: TCP
      port: 8080
```

---

## 6. Step-by-Step Deployment Guide

```bash
terraform init -backend=false
terraform validate
terraform plan
```

---

## 7. Expected Validation Results

```text
[Statically validated — not applied to a live account]

Verifying CNI Pod IP Allocation:
$ kubectl get pods -o wide
NAME                        READY   STATUS    IP           NODE
frontend-69f8c6b75f-abcde   1/1     Running   10.0.10.45   ip-10-0-10-12.ec2.internal
backend-54b9d887dc-fghij    1/1     Running   10.0.20.78   ip-10-0-20-34.ec2.internal

Note: Pod IPs reside directly within the VPC/VCN CIDR space (Zero overlay NAT).
```

---

## 8. Failure Injection Drill: NetworkPolicy Ingress Blockade

### The Scenario
Attempt to connect to the backend API pod from an unauthorized namespace (e.g., `default`), testing NetworkPolicy microsegmentation.

### The Injection
```bash
kubectl run test-pod -n default --image=curlimages/curl --rm -it -- curl --connect-timeout 3 http://backend-service.backend:8080/
```

### Manifested Symptoms
```text
curl: (28) Connection timed out after 3000 milliseconds
```
The Linux kernel (eBPF / iptables) silently drops the SYN packet because the source namespace lacks the `frontend` label.

---

## 9. Debugging Walkthrough

```text
====================================================================================================
                        KUBERNETES CNI IP EXHAUSTION DIAGNOSIS
====================================================================================================

Symptom: Pod stuck in 'ContainerCreating' or 'Pending' with event:
'FailedCreatePodSandBox: failed to assign an IP address to container'
Step 1: Inspect Node Network Interface Count
  $ aws ec2 describe-network-interfaces --filters "Name=attachment.instance-id,Values=i-xxxx"
Step 2: Check AWS VPC CNI Subnet Available IPs
  $ aws ec2 describe-subnets --subnet-ids subnet-xxxx --query "Subnets[0].AvailableIpAddressCount"
  Output: 0
Root Cause: Subnet ran out of available private IP addresses!
Remediation: Expand VPC secondary CIDR block or enable CNI prefix delegation (assigns /28 CIDRs per ENI).
====================================================================================================
```

---

## 10. Teardown & Verification

```bash
terraform destroy -auto-approve
```

---

## 11. Senior Interview Defense & Takeaways

> **Interviewer**: *"Why configure the AWS Load Balancer Controller with `target-type: ip` instead of `target-type: instance`?"*

**Candidate Defense**:
*"In traditional Kubernetes `target-type: instance` routing, the Application Load Balancer routes traffic to an arbitrary worker node on an ephemeral NodePort. That worker node uses `kube-proxy` iptables NAT rules to forward the packet across the cluster network to the actual pod on another node.*

*This 'instance' routing introduces two severe penalties: it adds 2-4ms of extra network latency due to the second network hop, and it masks the real client IP address (SNAT). By contrast, `target-type: ip` leverages the AWS VPC CNI / OCI VCN-Native CNI to route traffic directly from the Load Balancer to the Pod's private IP in a single hop, preserving client IPs and maximizing throughput."*
