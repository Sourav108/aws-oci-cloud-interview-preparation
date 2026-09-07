# Production Cloud Reference Architecture Projects

---

## 1. Catalog Overview & Purpose

This directory houses **8 enterprise-grade production reference architecture projects** designed for Senior, Staff, and Principal Cloud Engineers and Enterprise Solutions Architects.

While whiteboard system designs demonstrate theoretical modeling, these reference projects bridge the gap between design theory and production implementation. Each project provides an end-to-end blueprint containing:
- **Dual-Cloud System Topology**: Side-by-side architectures for Amazon Web Services (AWS) and Oracle Cloud Infrastructure (OCI).
- **Production Infrastructure as Code (IaC)**: Hardened, modular HashiCorp Terraform (HCL) configurations adhering to the 12-factor cloud-native methodology.
- **Zero-Trust Security & Workload Identity**: Credential-less IAM architectures using EKS Pod Identity / AWS IRSA and OCI Workload Identity / Dynamic Groups with hardware KMS envelope encryption.
- **Production SRE Observability**: Golden signal instrumentation, OpenTelemetry daemonset exporters, and multi-burn-rate error budget alerts.
- **FinOps Unit Economics**: Comprehensive Bill of Materials (BOM) for Development, Staging, and Production environments with cost-saving guardrails.
- **Failure Injection & Game Day Runbooks**: Concrete chaos engineering procedures to validate resilience under simulated infrastructure failures.

---

## 2. Project Directory Roadmap

```
projects/
├── README.md                                             # This catalog index, deployment rules, and cost governance
├── 01-ha-web-tier-rest-api/README.md                     # Project 1: High-Availability Tier-1 REST API
├── 02-event-driven-ecommerce/README.md                   # Project 2: Asynchronous Event-Driven Order Processing
├── 03-global-media-streaming-pipeline/README.md          # Project 3: Global Media Streaming & Transcoding Pipeline
├── 04-multi-region-disaster-recovery/README.md           # Project 4: Multi-Region Active-Passive Disaster Recovery
├── 05-realtime-iot-telemetry-lakehouse/README.md         # Project 5: Real-Time IoT Telemetry & Streaming Lakehouse
├── 06-enterprise-multi-account-landing-zone/README.md    # Project 6: Enterprise Multi-Account / Multi-Compartment Landing Zone
├── 07-zero-trust-kubernetes-platform/README.md           # Project 7: Zero-Trust Multi-Tenant Kubernetes Platform (EKS + OKE)
└── 08-multi-tenant-saas-data-isolation/README.md         # Project 8: Multi-Tenant SaaS Data Isolation Architecture
```

---

## 3. Master Reference Architecture Matrix

| # | Reference Project | Architectural Archetype | AWS Production Stack | OCI Production Stack | Target Industry / SLA |
| :-: | :--- | :--- | :--- | :--- | :--- |
| **01** | **HA Web Tier REST API** | Synchronous, Low-Latency Microservices | Route 53 + CloudFront + ALB + EKS + Aurora PG + ElastiCache | OCI DNS + OCI CDN + OCI LB + OKE + Autonomous DB + OCI Cache | Fintech / SaaS (99.999% SLA, P99 < 35ms) |
| **02** | **Event-Driven E-Commerce** | Asynchronous Decoupled Saga Mesh | API GW + SQS FIFO + Step Functions + Lambda/EKS + DynamoDB | OCI API GW + OCI Queue + OCI Events + OCI Functions + OCI NoSQL | E-Commerce Retail (100k TPS burst, RPO=0) |
| **03** | **Global Media Streaming** | Zero-Proxy Chunked Ingest & Video Transcoding | S3 Transfer Acceleration + Pre-Signed URLs + Batch GPU + CloudFront | OCI FastConnect Edge + PAR + OKE NVIDIA GPU Shapes + OCI CDN | Media & Entertainment (500 TB/day ingest) |
| **04** | **Multi-Region Disaster Recovery** | Automated Regional Failover & WORM Compliance | Route 53 ARC + Aurora Global DB + S3 CRR + Backup Vault Lock | OCI DNS Steering + Full Stack DR + Active Data Guard + Immutable S3 | Core Banking / Healthcare (RTO < 15m, RPO < 1m) |
| **05** | **Real-Time IoT Lakehouse** | High-Velocity Streaming & Iceberg Lakehouse | Kinesis Data Streams + EMR Flink + S3 Apache Iceberg + Athena | OCI Streaming + OCI Data Flow + Object Storage Lakehouse + ADW | Industrial IoT / Connected Fleet (1M msgs/sec) |
| **06** | **Enterprise Landing Zone** | Multi-Account & Multi-Compartment Governance | AWS Organizations + Control Tower + IAM Identity Center + TGW | OCI Identity Domains + Compartments + Dynamic Groups + DRG v2 | Regulated Enterprise / Shared Services Hub |
| **07** | **Zero-Trust Kubernetes** | Secure Container Platform with mTLS Service Mesh | EKS + Cilium eBPF + Istio mTLS + AWS KMS Secret Encryption | OKE + OCI VCN-Native CNI + Istio Service Mesh + OCI Vault KMS | Healthcare / Defense (Strict Microsegmentation) |
| **08** | **Multi-Tenant SaaS Isolation** | Hybrid Pool vs. Silo Multi-Tenant Data Platform | API GW + Cognito / Lambda Authorizer + Aurora Tenant Schemas | OCI API GW + Identity Domains + Autonomous DB Pluggable DBs (PDB) | B2B SaaS Enterprise (Tenant Data Segregation) |

---

## 4. Deployment Guidelines & Safety Guardrails

### 4.1 Zero Billable Apply Policy
In adherence to the repository safety policies:
- **Do Not Execute Blind Live Applies**: Never run `terraform apply` against live cloud accounts without explicit architectural approval, sandbox boundaries, and pre-calculated billing estimates.
- **Use Static Linting & Dry-Run Validation**:
  ```bash
  terraform init -backend=false
  terraform fmt -check
  terraform validate
  tflint
  checkov -d .
  ```

### 4.2 Credential & Workload Identity Guardrail
- Hardcoded API keys, access keys, or database passwords in source code or Terraform variable definitions (`terraform.tfvars`) are strictly forbidden.
- All implementations utilize **EKS Pod Identity / AWS IRSA** or **OCI Workload Identity / Dynamic Groups** to dynamically retrieve short-lived STS or security tokens at runtime.

### 4.3 FinOps Cost Protection Rules
- Every reference project includes an explicit FinOps Bill of Materials.
- In non-production testing environments, developers must:
  1. Provision minimal or flexible shapes (e.g., `t4g.small` on AWS, `VM.Standard.E4.Flex` with 1 OCPU / 8 GB on OCI).
  2. Implement automated scheduled shutdown policies (e.g., stopping non-production clusters outside business hours).
  3. Enforce lifecycle policies on storage buckets (auto-deleting test objects after 24 hours).
