# 01. Cloud Service Models & The Shared Responsibility Model

## 1. Problem
In distributed production systems, catastrophic security breaches and operational outages frequently stem from a misunderstanding of operational ownership. Engineering teams often operate under the dangerous assumption that deploying workloads to a "managed cloud" shifts all operational liability—patching, encryption, high availability, backup verification, and DDoS mitigation—to the cloud service provider. When a database is left unencrypted, an open security group exposes a port, or an operating system kernel vulnerability goes unpatched, systems are compromised not because the cloud failed, but because the boundary of responsibility was misunderstood.

## 2. Cloud Concept
Cloud computing reorganizes traditional computing infrastructure into three primary service models, each shifting the division of operational and security ownership:

1. **Infrastructure as a Service (IaaS)**: The provider virtualizes physical compute, storage, and networking hardware. The customer provisions virtual machines, attaches virtual block storage, configures virtual routing, and retains 100% operational ownership over the guest operating system, runtime libraries, middleware, and application code.
2. **Platform as a Service (PaaS)**: The provider abstracts the operating system, hardware provisioning, and runtime patching. The customer provides application code or container images and configures identity, access rules, and business logic.
3. **Software as a Service (SaaS)**: The provider manages the entire application stack from hardware to user interface. The customer manages user identities, data access permissions, and configuration toggles.

The **Shared Responsibility Model** formalizes this boundary:
- **Security OF the Cloud**: Physical facilities, bare-metal hardware, hypervisor virtualization layers, core physical networking, and environmental controls. Owned exclusively by the cloud provider.
- **Security IN the Cloud**: Customer data encryption, Identity & Access Management (IAM), network firewall rules, guest OS configuration, patch management, and application code. Owned exclusively by the customer.

## 3. Mental Model
Think of the Shared Responsibility Model as an apartment tenancy contract:
$$\text{Total System Risk} = \text{Provider Risk (Foundation + Frame + Core Plumbing)} + \text{Tenant Risk (Locks + Windows + Valuables)}$$

- In **IaaS**, the landlord provides four bare walls, electricity, and a water line. If you leave the front door unlocked or forget to extinguish a candle, the landlord is not liable when your belongings are stolen or damaged.
- In **PaaS**, the landlord provides a furnished hotel room with maintenance staff changing lightbulbs and servicing the HVAC. You remain responsible for whom you invite into the room and securing your passport in the room safe.
- In **SaaS**, you are a patron in a restaurant. You choose what to eat and pay the bill, but the restaurant cleans the kitchen and prepares the food.

## 4. Architecture Diagram
```text
┌────────────────────────────────────────────────────────────────────────┐
│                   SHARED RESPONSIBILITY BOUNDARY                       │
├──────────────────────────┬──────────────────────┬──────────────────────┤
│      IaaS (EC2 / OCI)    │   PaaS (RDS / ADB)   │  Serverless (Lambda) │
├──────────────────────────┼──────────────────────┼──────────────────────┤
│ [Customer] Data & IAM    │ [Customer] Data & IAM│ [Customer] Data & IAM│
│ [Customer] App Code      │ [Customer] App Code  │ [Customer] App Code  │
│ [Customer] OS & Runtime  │ ── ── ── ── ── ── ── │ ── ── ── ── ── ── ── │
│ ── ── ── ── ── ── ── ──  │ [Provider] OS/Patch  │ [Provider] Runtime/OS│
│ [Provider] Hypervisor    │ [Provider] Hypervisor│ [Provider] Hypervisor│
│ [Provider] Hardware & DC │ [Provider] Hardware  │ [Provider] Hardware  │
└──────────────────────────┴──────────────────────┴──────────────────────┘
```

## 5. AWS Implementation
In AWS, the shared responsibility boundary varies dynamically depending on the compute and data abstraction tier:

### Compute Tiers
- **Amazon EC2 (IaaS)**: AWS owns the physical data center, server hardware, and Nitro/Xen hypervisor `[Doc: AWS Shared Responsibility Model, checked 2026-09-03]`. The customer is strictly responsible for installing security patches on the Linux/Windows guest OS, configuring firewall rules via Security Groups and NACLs, managing SSH key pairs, and maintaining host-level intrusion detection systems.
- **Amazon ECS / EKS with Fargate (PaaS / CaaS)**: AWS owns the underlying container host OS, hypervisor microVMs (Firecracker), and cluster orchestration control plane. The customer owns container image vulnerabilities, pod security standards, RBAC, and network ingress policies.
- **AWS Lambda (Serverless)**: AWS manages the entire execution environment, language runtimes, scaling mechanisms, and underlying server fleet. The customer is responsible exclusively for function code, function IAM execution roles, and event payload validation.

### Data & Storage Tiers
- **Amazon S3**: AWS guarantees data durability ($99.999999999\%$) across physical facilities `[Doc: Amazon S3 SLA, checked 2026-09-03]`. The customer is 100% responsible for bucket access policies, preventing public ACL leaks, configuring Server-Side Encryption (SSE-KMS), and setting lifecycle policies.
- **Amazon RDS / Aurora**: AWS manages OS installation, database engine binary patching, hardware failover, and physical storage volume replication. The customer is responsible for schema design, SQL query indexing, database connection pooling, user credential rotation, and firewall access rules.

## 6. OCI Implementation
Oracle Cloud Infrastructure enforces an equivalent shared responsibility framework, with unique architectural delineations derived from its off-box virtualization and enterprise governance model:

### Compute Tiers
- **OCI Compute (IaaS - Bare Metal & Virtual Machines)**:
  - *Bare Metal*: OCI provides physical servers with zero Oracle virtualization software on the host. Network virtualization is offloaded to custom SmartNIC cards (off-box virtualization) `[Doc: OCI Architecture Overview, checked 2026-09-03]`. The customer owns 100% of the OS, kernel, hypervisor (if nesting), and local storage encryption.
  - *Virtual Machines*: OCI manages the hypervisor on the physical host. The customer manages the guest OS, yum/apt security updates, and network packet filtering via Network Security Groups (NSGs) and Security Lists.
- **OCI Container Engine for Kubernetes (OKE)**: OCI manages the Kubernetes control plane nodes at zero cost for Basic Clusters `[Doc: OKE Pricing, checked 2026-09-03]`. The customer selects worker node shapes (Bare Metal or VM) and is responsible for worker node OS patching (via node pool recycling) and Kubernetes NetworkPolicies.
- **OCI Functions (Serverless)**: Built on the open-source Fn Project. OCI provisions and manages the underlying Docker container execution engine. The customer packages function logic as container images and assigns OCI IAM policies to dynamic groups.

### Data & Governance Tiers
- **OCI Autonomous Database (PaaS)**: Represents a significantly higher level of provider responsibility than standard RDS. OCI automates database tuning, security patch application without downtime, automatic index generation, and automated backup schedules `[Doc: Autonomous Database Technical Overview, checked 2026-09-03]`. The customer manages data schemas, user privilege grants, and network access whitelists.
- **Compartments & Security Zones**: OCI enforces hierarchical resource isolation via **Compartments**. Furthermore, **OCI Security Zones** allow security teams to enforce non-negotiable policies: if an environment is declared a Security Zone, OCI will programmatically reject any customer API request that attempts to assign a public IP, disable encryption, or attach unencrypted storage volumes `[Doc: OCI Security Zones, checked 2026-09-03]`.

## 7. Configuration
Comparing an IaaS VM deployment where the customer owns network access controls:

### AWS Security Group Configuration (Terraform)
```hcl
# In IaaS, customer is responsible for restricting ingress
resource "aws_security_group" "app_tier" {
  name        = "app-tier-sg"
  description = "Restrict access strictly to private ALB"
  vpc_id      = var.vpc_id

  ingress {
    description     = "Allow HTTP from internal ALB only"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [var.alb_security_group_id] # Least-privilege reference
  }

  egress {
    description = "Allow outbound to NAT Gateway for package updates"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### OCI Network Security Group Configuration (Terraform)
```hcl
# OCI NSG attached directly to instance VNIC
resource "oci_core_network_security_group" "app_tier" {
  compartment_id = var.compartment_id
  vcn_id         = var.vcn_id
  display_name   = "app-tier-nsg"
}

resource "oci_core_network_security_group_security_rule" "allow_alb_ingress" {
  network_security_group_id = oci_core_network_security_group.app_tier.id
  direction                 = "INGRESS"
  protocol                  = "6" # TCP

  source_type = "NETWORK_SECURITY_GROUP"
  source      = var.lb_nsg_id

  tcp_options {
    destination_port_range {
      min = 8080
      max = 8080
    }
  }
}
```

## 8. Data Flow
```text
Client Request ──> [Cloud Edge / Anti-DDoS: AWS Shield / OCI DDoS Protection] (Provider Owned)
                          │
                          ▼
                   [Internet Gateway / VCN IGW] (Provider Managed Primitive)
                          │
                          ▼
                   [Customer VPC/VCN Subnet Route Table] (Customer Configured)
                          │
                          ▼
                   [Security Group / OCI NSG] (Customer Enforced Firewall)
                          │
                          ▼
                   [EC2 / OCI Compute Guest OS & App] (Customer Patched & Maintained)
```

## 9. Security
- **Identity Isolation**: In AWS, identity access boundaries are governed by IAM policies and Organization Service Control Policies (SCPs). In OCI, identity is scoped to Compartments and tenancy policy inheritance.
- **Workload Identity**: In both clouds, never store long-lived credentials inside VMs or containers. In AWS, bind IAM Roles to EC2 Instance Profiles or EKS ServiceAccounts (IRSA). In OCI, match instance OCIDs inside a **Dynamic Group** and grant policies to the Dynamic Group.
- **Data Protection Boundary**: In IaaS, encrypting local swap space, temporary directories, and in-flight traffic is the customer's duty. In PaaS, the provider handles transparent data encryption (TDE) at rest by default.

## 10. Reliability
Under IaaS, high availability across multiple physical failure domains is entirely the customer's architectural responsibility:
- **AWS**: If you deploy an EC2 instance to a single Availability Zone (`us-east-1a`), AWS provides zero architectural failover if that physical data center loses utility power. The customer must configure Auto Scaling Groups spanning multiple AZs.
- **OCI**: In single-AD regions, the customer must deliberately distribute compute instances across **3 Fault Domains** (`FAULT-DOMAIN-1`, `FAULT-DOMAIN-2`, `FAULT-DOMAIN-3`) to prevent a single server rack or top-of-rack switch failure from taking down the cluster.

## 11. Scaling
- **IaaS Scaling**: Scaling requires the customer to configure auto-scaling rules, health-check grace periods, golden AMI / custom image build pipelines, and warm-up times.
- **Serverless / PaaS Scaling**: Scaling is managed by the provider based on incoming requests (e.g., Lambda concurrency scaling up to regional burst limits; Autonomous DB auto-scaling compute up to 3x base allocation without downtime).

## 12. Observability
The division of observability responsibility follows the boundary:
- **Provider Telemetry**: Hypervisor health, physical network packet drops, disk hardware telemetry (hidden or surfaced via coarse metrics like AWS `StatusCheckFailed_System` or OCI `InstanceStatus`).
- **Customer Telemetry**: OS CPU utilization, memory pressure, disk I/O wait, application garbage collection pauses, and application HTTP 5xx error logs. These must be collected via customer-installed agents (AWS CloudWatch Agent, OCI Unified Monitoring Agent, OpenTelemetry Collector).

## 13. Cost
- **IaaS Economics**: Billed for provisioned capacity regardless of utilization. An idle `m6i.4xlarge` or `VM.Standard.E5.Flex` costs the exact same whether it runs at 1% CPU or 99% CPU.
- **PaaS / Serverless Economics**: Billed on consumption metrics (e.g., Lambda GB-seconds, API Gateway request counts, Aurora Serverless ACU consumption). Higher unit cost per compute cycle, but zero cost when traffic drops to zero.

## 14. Failure Modes
- **The Unpatched Kernel Vulnerability**: A team runs EC2/Compute instances for 18 months without updating base OS images. An attacker leverages an unpatched Linux kernel vulnerability (e.g., Dirty COW / Dirty Pipe) to gain root access. *Liability: 100% Customer.*
- **The Public Bucket Data Leak**: An engineer changes an S3 bucket or OCI Object Storage bucket visibility to "Public" to test an image upload. Sensitive customer records are scraped by public search engines. *Liability: 100% Customer.*
- **The Single-AZ/AD Blackhole**: A major electrical substation failure knocks out an entire AWS AZ. A company's critical payment API goes offline for 6 hours because its RDS instance was provisioned in Single-AZ mode to save cost. *Liability: 100% Customer architectural choice.*

## 15. Troubleshooting
When investigating whether an incident is a cloud platform issue or a customer responsibility issue:
1. **Check Provider Status**: Inspect AWS Health Dashboard or OCI System Status.
2. **Check System vs. Instance Health**:
   - AWS: If `StatusCheckFailed_System` is 1, hypervisor hardware is degraded (AWS responsibility). Remediate by stopping and starting the instance to trigger migration to a new physical host.
   - AWS: If `StatusCheckFailed_Instance` is 1, the guest OS has crashed, kernel panicked, or run out of memory (Customer responsibility).
   - OCI: Review OCI Compute Instance Status and inspect Console History / VNC Console logs to debug kernel boot hangs.

## 16. Common Mistakes
- **Assuming Automated Backups Mean Point-in-Time Recovery**: Cloud providers offer automated snapshot capabilities, but configuring snapshot schedules, cross-region replication, and regular restore validation is entirely the customer's duty.
- **Treating Default Cloud VPCs as Production Ready**: Default VPCs in AWS feature public subnets with internet gateways attached. Deploying databases into default VPCs without modification exposes them directly to public internet routing.

## 17. Trade-offs
| Dimension | Infrastructure as a Service (IaaS) | Platform as a Service (PaaS) | Serverless Compute |
| :--- | :--- | :--- | :--- |
| **Control** | Complete (Kernel flags, custom drivers, filesystem) | Moderate (Runtime configs, environment parameters) | Low (Constrained to execution sandbox) |
| **Operational Overhead** | High (OS patching, scaling, log shipping) | Low (Managed maintenance windows, automated HA) | Minimal (Focus entirely on business code) |
| **Portability** | High (Standard Linux images run anywhere) | Medium (Tied to provider database or runtime APIs) | Low (Proprietary triggers and event bindings) |
| **Failure Blast Radius**| Broad (Misconfigured OS compromises all hosted apps) | Scoped (Provider isolates managed instances) | Minimal (Ephemeral microVMs per request) |

## 18. Interview Questions
1. *In an enterprise cloud interview: If an attacker executes a ransomware payload inside your production EC2 instances and encrypts all attached EBS volumes, did AWS fail its security obligations? How would you defend this to your CTO?*
2. *How does the boundary of shared responsibility change when migrating from an unmanaged PostgreSQL database on EC2/OCI Compute to Amazon Aurora or OCI Autonomous Transaction Processing?*
3. *What mechanisms exist in OCI to prevent junior infrastructure engineers from violating the customer's side of the shared responsibility model?*

## 19. Interview Answer
**Exemplary Answer to Question 2**:
> "When migrating an unmanaged PostgreSQL instance from an IaaS VM to Amazon Aurora or OCI Autonomous Database, the customer boundary shifts upward dramatically.
>
> In an IaaS deployment on EC2 or OCI Compute, our team owns:
> 1. Operating system security patching, kernel tuning, and disk volume management.
> 2. Database binary installation, minor version upgrades, and patch maintenance.
> 3. Designing synchronous streaming replication across Availability Zones and handling failover scripts.
> 4. Managing WAL archiving, point-in-time backup consistency, and disk corruption recovery.
>
> When we migrate to Amazon Aurora or OCI Autonomous DB:
> - The cloud provider assumes complete responsibility for OS patching, hypervisor availability, automated physical storage replication (e.g., Aurora's 6-way quorum storage across 3 AZs), minor engine updates, and sub-30-second automated failovers. In OCI Autonomous Database, Oracle even assumes responsibility for automated index generation and performance tuning.
> - However, our team retains 100% ownership over: database authentication (IAM vs native DB users), client-side connection pooling (preventing connection exhaustion via RDS Proxy or PgBouncer), query optimization, table schema design, and enforcing TLS in transit. The provider gives us high availability of the database engine, but we remain responsible for ensuring our application queries do not monopolize database resources."

## 20. Hands-on Exercise
**Objective**: Demonstrate customer-side firewall enforcement by creating a private VM that refuses all public ingress while permitting outbound access to cloud package mirrors via a NAT Gateway.

### Verification Steps
1. Verify instance has no public IPv4 address assigned.
2. Attempt inbound SSH from public internet: connection must immediately time out.
3. Establish session via AWS SSM Session Manager or OCI Bastion Service.
4. Execute `curl -I https://amazonlinux.default.amazonaws.com` (AWS) or `yum check-update` (OCI): connection succeeds through the NAT Gateway.
5. Verify in VPC/VCN Flow Logs that inbound unsolicited traffic is dropped at the Security Group / NSG boundary with status `REJECT`.
