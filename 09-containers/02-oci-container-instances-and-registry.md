# 02. OCI Container Instances & Cloud Container Registries (ECR vs. OCIR)

## 1. Problem
In many enterprise organizations, developers provision heavy, expensive multi-node Kubernetes clusters (EKS/OKE) simply to run periodic background batch processors, machine learning inference containers, or CI/CD test runners. This incurs high baseline cluster management fees, creates idle compute waste, and demands ongoing control-plane maintenance. Furthermore, container registries are frequently left unmanaged, accumulating thousands of untagged legacy image layers that cost thousands of dollars in monthly storage waste and harbor critical unpatched Common Vulnerabilities and Exposures (CVEs).

## 2. Cloud Concept
### Serverless Container Instances (No Orchestrator Required)
- **OCI Container Instances**:
  - A fully managed compute service that enables customers to run containers directly on OCI high-performance compute infrastructure without managing servers, clusters, or container orchestrators (like Kubernetes or Nomad) `[Doc: OCI Container Instances Overview, checked 2026-09-03]`.
  - A Container Instance is provisioned inside your private VCN, attaching a native VNIC with dedicated private IP addressing and NSG security protection.
  - Sized flexibly: you choose exact OCPUs and memory down to fractions of a core.
  - Startup latency: typically **under 5 seconds**, making it ideal for event-driven batch jobs and elastic worker fleets.

### Managed Container Registries: AWS ECR vs. OCI OCIR
A container registry is an enterprise, private, Docker v2 and OCI-compliant image storage repository:
- **AWS Elastic Container Registry (ECR)**:
  - Regional or cross-region replicated container repository.
  - Deep IAM policy integration for repository access control.
  - **Automated Vulnerability Scanning**: Basic scanning (powered by Clair) and Enhanced scanning (powered by Amazon Inspector) that scans images continuously against CVE databases.
  - **Lifecycle Policies**: Enforces retention limits (e.g., keep only the latest 30 tagged images, delete untagged images older than 14 days) to prevent storage bloat.
- **OCI Registry (OCIR)**:
  - A standards-compliant Docker Registry integrated directly into OCI Compartments and Identity Domains.
  - Supports private and public repositories.
  - **Built-in Vulnerability Scanning**: Integrates with the OCI Vulnerability Scanning Service to automatically inspect layers for known security vulnerabilities upon push.
  - **Cross-Region Replication**: Supports automated asynchronous replication of container images across OCI regions for disaster recovery and geo-distributed latency minimization.

## 3. Mental Model
Think of container runtime and registry services as modern manufacturing logistics:
- **The Container Registry (ECR / OCIR)** is the central blueprint library and secure warehouse where tested machine parts (container images) are cataloged, scanned for manufacturing defects (CVE scanning), and stored.
- **OCI Container Instances** is on-demand 3D printing. When an order arrives, a dedicated printer instantly spins up in your private workshop, executes the task in seconds, and disappears. You don't need to build a permanent, sprawling factory floor (Kubernetes) just to print a single widget.

## 4. Architecture Diagram
```text
┌────────────────────────────────────────────────────────────────────────┐
│                   ENTERPRISE CONTAINER PIPELINE                       │
│                                                                        │
│   [Developer / CI-CD Runner]                                           │
│               │                                                        │
│               ▼ 1. docker push (TLS 1.3)                               │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ CLOUD CONTAINER REGISTRY (AWS ECR / OCI OCIR)                  │   │
│   ├────────────────────────────────────────────────────────────────┤   │
│   │ * Automated Image Vulnerability Scan (CVE Detection)           │   │
│   │ * Lifecycle Rule: Prune untagged images > 14 days old          │   │
│   │ * Cross-Region Replication (Primary Region ──► DR Region)      │   │
│   └───────────────────────┬────────────────────────────────────────┘   │
│                           │                                            │
│                           ▼ 2. Pulls Scanned Image                     │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ OCI CONTAINER INSTANCE (Private VCN Subnet)                    │   │
│   ├────────────────────────────────────────────────────────────────┤   │
│   │ * Instant Serverless Launch (< 5s)                             │   │
│   │ * Attached to Customer Private VCN Subnet (Native VNIC)        │   │
│   │ * Sized via Flexible Shape: 2 OCPUs, 8 GB RAM                  │   │
│   │ * ZERO Kubernetes Cluster Overhead                             │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS:
- **Amazon ECR Features**:
  - **Cross-Region and Cross-Account Replication**: Configured at the private registry level. Images pushed to `us-east-1` automatically replicate to `eu-west-1` and disaster recovery regions asynchronously.
  - **KMS Encryption**: Repositories encrypted at rest using AWS KMS Customer Managed Keys (CMKs).
  - **ECR Immutable Tags**: Prevents tag overwriting. Tagging an image with `v1.2.0` permanently locks that tag, blocking accidental deployment of unvetted builds or supply-chain attacks.
  - **Pull-Through Cache**: ECR can cache upstream public registries (Docker Hub, GitHub Packages, Quay.io) directly within your private VPC, eliminating rate limits and external internet dependencies.

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **OCI Container Instances Deep Dive**:
  - **No Control Plane Tax**: OCI charges **zero fees** for the Container Instance service itself. You pay only for the underlying OCPU and Memory consumed at standard OCI Compute rates `[Doc: OCI Container Instances Pricing, checked 2026-09-03]`.
  - **Container Groups**: A Container Instance can host multiple co-located containers that share local loopback networking (`localhost`) and shared memory volumes (identical to the Kubernetes Pod specification).
  - **Restart Policies**: Configurable restart behaviors: `ALWAYS`, `ON_FAILURE`, or `NEVER` (ideal for one-shot batch tasks).
  - **Native IAM Instance Principals**: Containers can authenticate directly to OCI APIs (e.g., reading Object Storage or Vault) using OCI Instance Principals, completely eliminating hardcoded API keys.
- **OCI Registry (OCIR)**:
  - Supports Docker CLI authentication using OCI Auth Tokens.
  - Granular access control mapped to OCI Compartments and IAM policies (e.g., `Allow group Developers to read repos in compartment Dev`).
  - Native integration with OKE and OCI Container Instances for seamless image pulls over the internal Oracle backbone with zero NAT data costs.

## 7. Configuration
Comparing registry and serverless container configuration in Terraform:

### AWS ECR Repository with Lifecycle Policies & Scanning (Terraform)
```hcl
# Secure AWS ECR Repository
resource "aws_ecr_repository" "api_repo" {
  name                 = "enterprise-api"
  image_tag_mutability = "IMMUTABLE" # Prevents tag poisoning!

  image_scanning_configuration {
    scan_on_push = true # Automated CVE scanning!
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = var.kms_key_arn
  }
}

# Lifecycle Policy: Delete untagged images older than 14 days
resource "aws_ecr_lifecycle_policy" "prune_policy" {
  repository = aws_ecr_repository.api_repo.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Delete untagged images older than 14 days"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 14
        }
        action = { type = "expire" }
      },
      {
        rulePriority = 2
        description  = "Retain maximum of 30 tagged production images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v", "prod"]
          countType     = "imageCountMoreThan"
          countNumber   = 30
        }
        action = { type = "expire" }
      }
    ]
  })
}
```

### OCI Container Instance Deployment (Terraform)
```hcl
# OCI Container Instance running a batch processing job
resource "oci_container_instances_container_instance" "batch_job" {
  compartment_id      = var.compartment_id
  availability_domain = var.availability_domain
  display_name        = "nightly-settlement-job"
  shape               = "CI.Standard"

  shape_config {
    ocpus         = 2
    memory_in_gbs = 8
  }

  vnics {
    subnet_id        = var.private_subnet_id # Attached to private VCN!
    assign_public_ip = false
  }

  containers {
    display_name = "settlement-processor"
    image_url    = "${var.ocir_region}.ocir.io/${var.tenancy_name}/batch:v2.4"

    environment_variables = {
      ENVIRONMENT = "production"
      BATCH_DATE  = "2026-09-03"
    }

    resource_config {
      vcpus_limit   = 2
      memory_in_gbs = 8
    }
  }

  # Once complete, do not restart!
  container_restart_policy = "NEVER"
}
```

## 8. Data Flow
```text
Secure Image Push, Scan, and Deployment Pipeline:
1. CI/CD Runner builds image: docker build -t app:v1.0.0
2. CI/CD authenticates via temporary OAuth2 token to ECR / OCIR.
3. Runner pushes layers: docker push registry/app:v1.0.0
4. Registry intercepts push:
   * Layers written to encrypted Object Storage backend.
   * Automated Vulnerability Scanner initiates static layer analysis.
   * Compares package manifests (rpm, deb, npm, pip) against national CVE databases.
   * If CRITICAL vulnerability found (e.g., CVSS score >= 9.0), EventBridge / OCI Event triggers webhook to block deployment.
5. If scan passes: OCI Container Instance / ECS pulls image across private VPC endpoint / Service Gateway.
```

## 9. Security
- **Image Tag Immutability**:
  - Always enforce `image_tag_mutability = "IMMUTABLE"`.
  - Without immutability, a malicious developer or compromised CI pipeline can overwrite the `latest` or `v1.0.0` tag with a backdoor-injected image, poisoning production deployments without altering deployment manifests.
- **Private Registry Endpoint Access**:
  - Prevent container nodes from pulling images over the public internet.
  - In AWS, provision **ECR Interface VPC Endpoints** (`com.amazonaws.region.ecr.api` and `com.amazonaws.region.ecr.dkr`).
  - In OCI, route image pulls through the **Service Gateway**, completely eliminating internet traversal and NAT fees.

## 10. Reliability
- **Cross-Region Registry Replication**:
  - If an entire cloud region experiences an outage, local deployment pipelines fail if container images are stored exclusively in that region.
  - Configure automated cross-region replication so that every pushed image is mirrored in a secondary disaster recovery region within minutes.

## 11. Scaling
- **Avoiding Docker Hub Rate Limits via Pull-Through Cache**:
  - Public Docker Hub enforces rate limits (100 pulls per 6 hours for anonymous users). In an auto-scaling cluster with 500 nodes, instances will fail to pull base images (e.g., `alpine:latest` or `python:3.11`).
  - Both ECR and OCIR support **Pull-Through Cache**, automatically caching upstream public images in your private repository and serving future requests internally at gigabit speeds.

## 12. Observability
- **ECR / OCIR Metrics**:
  - Track `RepositorySize` and `ImageCount` to prevent unbounded storage growth.
  - Monitor scan findings: `ImageScanFindingsSummary` in AWS Security Hub or OCI Cloud Guard.

## 13. Cost
- **Container Registry Billing**:
  - ECR and OCIR charge **zero repository creation fees**.
  - Storage is billed at standard Object Storage rates ($\approx \$0.10/\text{GB/month}$ in AWS ECR, and $\approx \$0.025/\text{GB/month}$ in OCI OCIR) `[Doc: AWS ECR Pricing, checked 2026-09-03]`.
  - In an enterprise pushing 50 GB of unpruned build artifacts per day, storage costs reach **\$1,500/month** within a year unless Lifecycle Policies are enforced!

## 14. Failure Modes
- **The OOM Container Kill in Serverless Instances**: A container instance is allocated 4 GB RAM. A memory leak causes resident set size (RSS) to hit 4,001 MB. The underlying Linux cgroup memory controller terminates the container with `ExitCode 137`. Because the restart policy is set to `NEVER`, the batch pipeline abruptly stops without completing transactions.
- **The Untagged Image Storage Cost Creep**: A CI/CD pipeline pushes images tagged with git commit hashes (`sha-12345`). Over 18 months, 40,000 images accumulate. The company pays over \$4,000/month for orphaned image layers until an automated lifecycle policy is applied to purge images older than 30 days.

## 15. Troubleshooting
When container instances fail to pull images from private registries:
1. **Verify Token Expiration**:
   - ECR authorization tokens expire after **12 hours**. If a custom script runs `docker login`, it must re-authenticate periodically via `aws ecr get-login-password`.
2. **Inspect Private Network Routing**:
   - Does the private subnet have a VPC Interface Endpoint for ECR (or an OCI Service Gateway route)?
3. **Verify Security Group / NSG on Endpoints**:
   - Ensure the private endpoint's security group allows inbound port 443 from the container instance subnet.

## 16. Common Mistakes
- **Failing to Configure ECR Lifecycle Policies**: Allowing every single pull-request build image to persist indefinitely, creating terabytes of zombie image layers.
- **Using Kubernetes When Container Instances Suffice**: Architecting a full OKE or EKS cluster for a team whose only requirement is running three cron-triggered nightly Python scripts.

## 17. Trade-offs
| Feature | OCI Container Instances | Kubernetes (OKE / EKS) | AWS ECS (Fargate) |
| :--- | :--- | :--- | :--- |
| **Startup Speed** | Ultra-Fast (< 5 seconds) | Moderate (15–45 seconds) | Fast (10–20 seconds) |
| **Cluster Management** | Zero (No cluster concept) | High (Control plane & nodes) | Low (Serverless cluster abstraction) |
| **Networking** | Native VCN VNIC per instance | CNI plugin (Overlays/ENIs) | `awsvpc` ENI per task |
| **Best Workloads** | Batch jobs, CI/CD runners, cron jobs | Complex microservice graphs | Production enterprise microservices |

## 18. Interview Questions
1. *Why should an enterprise enforce 'Image Tag Immutability' on private container registries? What attack vectors does this prevent?*
2. *When would you recommend OCI Container Instances over a full Kubernetes (OKE) cluster, and what are the cost and operational implications?*
3. *How do you eliminate Docker Hub rate-limiting errors during large-scale autoscaling events in private cloud networks?*

## 19. Interview Answer
**Exemplary Answer to Question 2**:
> "I recommend **OCI Container Instances** over a full Kubernetes (OKE) cluster when the workload requirements center on **independent, ephemeral, or asynchronous batch computing** rather than complex microservice service-to-service orchestration:
>
> 1. **Operational Overhead Elimination**:
>    - Managing a Kubernetes cluster (even managed OKE) introduces significant operational toil: configuring control-plane upgrades, tuning CoreDNS, managing CNI IP pools, updating kubelet versions on worker nodes, and maintaining ingress controllers.
>    - OCI Container Instances eliminates 100% of cluster management. There is no control plane, no worker nodes, and no Kubernetes API to maintain.
>
> 2. **Financial Efficiency**:
>    - A Kubernetes cluster requires running baseline worker nodes 24/7 to maintain system pods (CoreDNS, kube-proxy, CSI drivers), generating constant idle compute costs.
>    - OCI Container Instances has **zero service markup and zero idle cluster costs**. You pay standard compute rates strictly for the exact seconds the container executes.
>
> 3. **Startup Latency & VCN Integration**:
>    - OCI Container Instances launch in under 5 seconds, attaching directly to your private VCN via native VNICs with full NSG microsegmentation.
>
> 4. **Decision Boundary**:
>    - If the application is a long-running graph of 50 decoupled microservices requiring service discovery, mutual TLS, and complex traffic ingress, **OKE is mandatory**.
>    - If the workload consists of event-driven batch jobs, nightly data ETL processors, CI/CD runners, or simple standalone webhooks, **OCI Container Instances is vastly superior** in simplicity, cost, and maintenance."

## 20. Hands-on Exercise
**Objective**: Provision an ECR repository with automated vulnerability scanning and deploy an image lifecycle policy using Terraform.

### Verification Steps
1. Deploy an ECR repository with `image_tag_mutability = "IMMUTABLE"` and `scan_on_push = true`.
2. Build and push a lightweight Docker image with an intentionally outdated package (e.g., old OpenSSL version).
3. Attempt to push an updated build with the exact same tag: verify that ECR rejects the push with `ImageTagAlreadyExistsException`.
4. Check the ECR console / AWS CLI for scan results:
   ```bash
   aws ecr describe-image-scan-findings --repository-name <repo> --image-id imageTag=<tag>
   ```
5. Confirm that the automated vulnerability scanner detects and catalogs the CVE severity breakdown.
