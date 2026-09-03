# Cloud Version & Grounding Reference Matrix

This repository maintains strict factual grounding against official cloud provider documentation and tool releases. All version-sensitive data, service quotas, CLI flags, default values, and architectural capabilities must cite their verification status using the grounding tags established below.

---

## 🏷️ Grounding Policy & Citation Taxonomy

Every factual assertion subject to change over time (service limits, default timeouts, pricing units, IAM action names, API versioning) carries one of three explicit tags:

1. `[Doc: <service/tool>, checked <YYYY-MM-DD>]` — Directly looked up and validated in official vendor documentation or changelog in the current session.
2. `[Inference]` — Deduced from established, invariant architectural principles (e.g., CAP theorem, TCP handshake behavior, stateless vs stateful packet inspection).
3. `[Approximation — verify before relying on this]` — Educated estimate or standard baseline when direct documentation lookup is unavailable or subject to frequent regional variations.

---

## 🛠️ Infrastructure & Platform Tooling Versions

| Tooling / Ecosystem | Version / Baseline | Status & Grounding Citation |
| :--- | :---: | :--- |
| **AWS CLI** | `v2.36.38` | `[Doc: aws-cli/v2 CHANGELOG.rst, checked 2026-09-03]` |
| **OCI CLI** | `v3.89.1` | `[Doc: oracle/oci-cli releases, checked 2026-09-03]` |
| **HashiCorp Terraform** | `v1.16.1` | `[Doc: hashicorp/terraform releases, checked 2026-09-03]` |
| **Kubernetes** | `v1.37.0` | `[Doc: kubernetes.io releases, checked 2026-09-03]` |
| **Docker Engine / CLI** | `v27.x / v28.x` | `[Approximation — verify before relying on this]` |
| **Helm** | `v3.16.x` | `[Approximation — verify before relying on this]` |
| **Terraform AWS Provider** | `~> 5.80` | `[Doc: registry.terraform.io/providers/hashicorp/aws, checked 2026-09-03]` |
| **Terraform OCI Provider** | `~> 6.20` | `[Doc: registry.terraform.io/providers/oracle/oci, checked 2026-09-03]` |
| **AWS SDK for Go (v2)** | `v1.36.x` | `[Approximation — verify before relying on this]` |
| **AWS SDK for Java (v2)** | `v2.30.x` | `[Approximation — verify before relying on this]` |
| **OCI Go SDK** | `v65.x` | `[Approximation — verify before relying on this]` |
| **OCI Java SDK** | `v3.50.x` | `[Approximation — verify before relying on this]` |

---

## ☁️ Cloud Service Versioning & Invariant Baselines

### 1. Compute & Containers
- **AWS EC2**: Nitro Hypervisor architecture `[Inference]`; IMDSv2 mandatory security posture `[Doc: EC2 User Guide, checked 2026-09-03]`.
- **OCI Compute**: Flexible shapes (`VM.Standard.E5.Flex`, `VM.Standard3.Flex`) with decoupled OCPU and Memory provisioning `[Doc: OCI Compute Shapes, checked 2026-09-03]`.
- **AWS EKS**: Kubernetes version support follows upstream Kubernetes + AWS extended support window `[Doc: EKS User Guide, checked 2026-09-03]`.
- **OCI OKE**: Managed Kubernetes control plane with Native Pod Networking via OCI VCN CNI plugin `[Doc: OCI Container Engine, checked 2026-09-03]`.

### 2. Networking & Traffic Routing
- **AWS VPC**: Maximum VPC CIDR block size `/16`, minimum `/28` `[Doc: Amazon VPC Quotas, checked 2026-09-03]`; Security Groups stateful `[Inference]`; NACLs stateless `[Inference]`.
- **OCI VCN**: Maximum VCN CIDR block size `/16`, minimum `/30` `[Doc: OCI VCN Overview, checked 2026-09-03]`; Network Security Groups (NSG) attached to VNICs (stateful or stateless) `[Doc: OCI NSG Docs, checked 2026-09-03]`; Security Lists attached to Subnets `[Doc: OCI Security Lists, checked 2026-09-03]`.
- **AWS ALB**: HTTP/2, HTTP/3 (QUIC), gRPC, WebSockets, L7 path/host routing `[Doc: AWS ALB User Guide, checked 2026-09-03]`.
- **OCI Load Balancer**: Flexible bandwidth load balancer (10 Mbps to 8000 Mbps min/max shape) `[Doc: OCI Load Balancing Service, checked 2026-09-03]`.

### 3. Managed Storage & Databases
- **AWS S3**: Strong read-after-write consistency for PUTs and DELETEs `[Doc: Amazon S3 Documentation, checked 2026-09-03]`; 3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD requests per second per prefix `[Doc: Amazon S3 Performance, checked 2026-09-03]`.
- **OCI Object Storage**: Strong consistency; Standard and Archive storage tiers; Auto-tiering support `[Doc: OCI Object Storage FAQ, checked 2026-09-03]`.
- **AWS Aurora**: Multi-AZ storage with 6-way replication across 3 AZs; quorum model: 4 of 6 for writes, 3 of 6 for reads `[Doc: Aurora Storage Architecture, checked 2026-09-03]`.
- **OCI Autonomous Database**: Serverless and Dedicated infrastructure modes; automated patching, tuning, and auto-scaling `[Doc: OCI Autonomous DB Overview, checked 2026-09-03]`.

### 4. IAM & Governance
- **AWS IAM**: Policy evaluation default deny; explicit deny overrides explicit allow; resource-based policies vs identity-based policies `[Doc: AWS IAM Evaluation Logic, checked 2026-09-03]`.
- **OCI IAM**: Compartment hierarchy; policy syntax `Allow <group|dynamic-group> to <verb> <resource-type> in <location> where <conditions>` `[Doc: OCI IAM Policy Syntax, checked 2026-09-03]`. Verbs: `inspect`, `read`, `use`, `manage` `[Doc: OCI IAM Verbs, checked 2026-09-03]`.
