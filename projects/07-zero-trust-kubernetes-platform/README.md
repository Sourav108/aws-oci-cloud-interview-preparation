# Reference Project 07: Zero-Trust Multi-Tenant Kubernetes Platform (EKS + OKE)

---

## 1. Executive Summary & Architecture Overview

This production reference architecture delivers a hardened, multi-tenant Zero-Trust Kubernetes platform utilizing Amazon Elastic Kubernetes Service (EKS) and Oracle Cloud Infrastructure Container Engine for Kubernetes (OKE). The architecture enforces complete east-west microsegmentation, cryptographic pod identity, hardware-backed secret encryption, and strict multi-tenant boundary isolation.

Key Architectural Capabilities:
- **eBPF-Powered Kernel Microsegmentation**: Replaces standard iptables with Cilium eBPF for wire-speed Layer 3/4/7 network security policies and deep observability.
- **Mutual TLS (mTLS) Service Mesh**: Istio service mesh enforces automatic cryptographic mTLS with SPIFFE/SPIRE identities across all inter-service container traffic.
- **Hardware-Enforced Envelope Encryption**: Kubernetes secrets are encrypted at rest inside `etcd` using AWS KMS Customer Managed Keys (CMK) / OCI Vault Dedicated KMS Keys.
- **Credential-less Workload Identity**: Eliminates static credentials using AWS EKS Pod Identity / IRSA and OCI Workload Identity with short-lived token projection.

---

## 2. Dual-Cloud System Topology

```
========================================================================================================================
                          ZERO-TRUST KUBERNETES PLATFORM TOPOLOGY (EKS & OKE)
========================================================================================================================

                                  [ Public Internet / B2B Clients ]
                                                  │
                                                  ▼
                          [ Ingress WAF + TLS Termination (Edge Ingress) ]
                                                  │
                                                  ▼
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  KUBERNETES CLUSTER PERIMETER (Private API Endpoint - Zero Public Master/Worker IPs)
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   ISTIO INGRESS GATEWAY (Terminates External Ingress & Initiates East-West Mesh)
   [ Istio Ingress Gateway Pods (Multi-AZ / Multi-Fault Domain) ]
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
                                                  │ (Strict mTLS 1.3 / SPIFFE Identity)
                                                  ▼
   TENANT NAMESPACE A (e.g., namespace: payments)      TENANT NAMESPACE B (e.g., namespace: analytics)
   ┌──────────────────────────────────────────────┐    ┌──────────────────────────────────────────────┐
   │ [ Payment API Pod ]                          │    │ [ Analytics Consumer Pod ]                   │
   │  - Envoy Sidecar (mTLS Proxy)                │    │  - Envoy Sidecar (mTLS Proxy)                │
   │  - ServiceAccount: payments-sa               │    │  - ServiceAccount: analytics-sa              │
   │  - Projected Workload Identity Token         │    │  - Projected Workload Identity Token         │
   │                                              │    │                                              │
   │ [ Cilium eBPF L7 Network Policy ]:           │    │ [ Cilium eBPF L7 Network Policy ]:           │
   │  - ALLOW Ingress from Ingress Gateway only   │    │  - DENY ALL Ingress from Namespace Payments  │
   │  - DENY ALL East-West to Namespace Analytics │    │  - ALLOW Egress to Data Lakehouse only       │
   └──────────────────────┬───────────────────────┘    └──────────────────────┬───────────────────────┘
                          │                                                   │
                          ▼                                                   ▼
   ETCD STORAGE ENVELOPE ENCRYPTION                    CREDENTIAL-LESS CLOUD WORKLOAD ACCESS
   [ AWS KMS CMK / OCI Vault Dedicated Key ]           [ AWS EKS Pod Identity / OCI Workload Identity ]
    - Hardware FIPS 140-2 Level 3 HSM                   - STS / Instance Principal Token Projection
    - Envelope encrypts all k8s secret objects          - Grants direct access to S3, Vault, and RDS
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
```

---

## 3. Dual-Cloud Component Mapping Matrix

| Architectural Layer | AWS Native Primitive | OCI Native Primitive | Architecture Notes |
| :--- | :--- | :--- | :--- |
| **Managed Kubernetes Control Plane**| Amazon EKS (Private Cluster Endpoint) `[Doc: EKS, checked 2026]` | OCI Container Engine for Kubernetes (OKE) `[Doc: OKE, checked 2026]` | Disables public API server endpoints; cluster accessible strictly via private bastion / VPN. |
| **Container Networking Interface** | AWS VPC CNI / Cilium CNI chaining | OCI VCN-Native CNI / Cilium eBPF | Direct private IP assignment to pods, eliminating double-NAT overlay routing bottlenecks. |
| **Service Mesh & mTLS** | Istio Service Mesh (Ambient / Sidecar) | Istio on OKE / OCI Service Mesh | Automatic mutual TLS 1.3 encryption with cryptographic SPIFFE/SPIRE x509 cert rotation. |
| **etcd Secrets Encryption** | AWS KMS Customer Managed Key | OCI Vault KMS Key | Protects `k8s-secrets` in etcd with envelope encryption before writing to disk. |
| **Workload Identity** | EKS Pod Identity / AWS IRSA | OCI Workload Identity / Dynamic Groups | Dynamic STS/principal token exchange; pods make cloud API calls without credentials. |
| **Kernel Microsegmentation** | Cilium eBPF Network Policies | Cilium eBPF / OCI NSGs | High-throughput kernel-level packet filtering; enforces Layer 7 HTTP path security rules. |

---

## 4. Production Infrastructure as Code (Terraform & Manifests)

### 4.1 AWS EKS Terraform Configuration (`aws_zero_trust_eks.tf`)

```hcl
# AWS Reference Implementation: Private EKS Cluster with KMS Secret Encryption
resource "aws_eks_cluster" "zero_trust_cluster" {
  name     = "prod-zero-trust-eks"
  role_arn = aws_iam_role.eks_cluster_role.arn
  version  = "1.30"

  vpc_config {
    subnet_ids              = [aws_subnet.private_az1.id, aws_subnet.private_az2.id, aws_subnet.private_az3.id]
    endpoint_private_access = true
    endpoint_public_access  = false # Air-gapped private control plane
  }

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks_etcd_key.arn
    }
    resources = ["secrets"]
  }

  tags = {
    Environment = "production"
    Security    = "zero-trust"
  }
}
```

### 4.2 Cilium Layer-7 Zero-Trust Network Policy (`cilium_policy.yaml`)

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "strict-payments-isolation"
  namespace: "payments"
spec:
  endpointSelector:
    matchLabels:
      app: payment-engine
  ingress:
  - fromEndpoints:
    - matchLabels:
        "k8s:io.kubernetes.pod.namespace": "istio-system"
        app: istio-ingressgateway
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "POST"
          path: "/api/v1/charge"
  egress:
  - toEndpoints:
    - matchLabels:
        "k8s:io.kubernetes.pod.namespace": "database-tier"
        app: pgbouncer
    toPorts:
    - ports:
      - port: "5432"
        protocol: TCP
```

---

## 5. Security, Workload Identity & Encryption

### 5.1 Workload Identity Token Projection
- Pods mount projected service account tokens:
```yaml
spec:
  containers:
  - name: api
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: workload-identity-token
  volumes:
  - name: workload-identity-token
    projected:
      sources:
      - serviceAccountToken:
          audience: "sts.amazonaws.com" # or "https://identity.oraclecloud.com"
          expirationSeconds: 3600
          path: token
```
- Cloud SDKs read this projected token to authenticate with AWS STS or OCI Identity, retrieving temporary credentials refreshed automatically every hour.

---

## 6. Observability, SLIs/SLOs & Alerting Runbook

1. **mTLS Handshake Failures**: Alert if Istio reports certificate verification failure spikes.
2. **Cilium Network Policy Drops**: Real-time eBPF packet drops (`cilium_drop_count_total`). Alert if $> 10$ drops/sec.
3. **KMS Decrypt Latency**: P99 etcd decryption latency. Target $< 5\text{ms}$.

---

## 7. Deployment & Verification Runbook

```bash
# Verify Cilium eBPF Status
cilium status --wait

# Verify Strict Mutual TLS Enforcement in Mesh
istioctl authn tls-check $(kubectl get pods -n payments -l app=payment-engine -o jsonpath='{.items[0].metadata.name}') payment-engine.payments.svc.cluster.local
```

---

## 8. FinOps Cost Breakdown & Sizing Economics

```text
====================================================================================================
                        FINOPS COST BREAKDOWN: ZERO-TRUST K8S PLATFORM
====================================================================================================

COMPONENT                         AWS MONTHLY COST        OCI MONTHLY COST
----------------------------------------------------------------------------------------------------
Kubernetes Control Plane          $73 (EKS $0.10/hour)    $0.00 (OKE Basic cluster free)
Compute Worker Nodes (30 Nodes)   $3,840                  $2,450 (Flexible shapes)
KMS Key & HSM Cryptographic Ops   $250                    $180
Observability (Hubble / OTel)     $450                    $220
----------------------------------------------------------------------------------------------------
TOTAL MONTHLY RUN-RATE            $4,613 / month          $2,850 / month
====================================================================================================
```

---

## 9. Failure Mode Drills & Chaos Engineering Runbook

### 9.1 Game Day Drill: Compromised Container Lateral Movement Test
1. **Action**: Exec into a compromised container in namespace `analytics` and attempt a TCP port scan or HTTP connection to `http://payment-engine.payments:8080`.
2. **Verification**:
   - Cilium eBPF intercepts the TCP SYN packet in the Linux kernel and drops it immediately.
   - Zero packets leave the host network interface card (NIC).
   - Cilium Hubble emits an alert: `POLICY_DENIED: Packet dropped from analytics/worker to payments/payment-engine`.
