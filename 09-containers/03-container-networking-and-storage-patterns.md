# 03. Container Networking & Storage Architecture Patterns

## 1. Problem
Containers are ephemeral by design: when a container terminates or crashes, its local writable layer is permanently discarded by the container runtime. Running stateful services (such as CMS uploads, machine learning pipelines, or legacy file processors) inside containers requires mounting shared, distributed network filesystems that survive container restarts and can be accessed concurrently by dozens of replicas across multiple Availability Domains. Furthermore, granting containers access to cloud resources without embedding permanent static credentials inside Docker images is a critical architectural requirement.

## 2. Cloud Concept: Container Storage & Identity
```text
CONTAINER MOUNT & IDENTITY ARCHITECTURE:

┌────────────────────────────────────────────────────────┐
│ CLOUD STORAGE BACKBONE (AWS EFS / OCI FSS)             │
│ Managed NFS v4.1 Filesystem (Multi-AZ / Multi-AD)      │
└───────────────────────────┬────────────────────────────┘
                            │ Read/Write Concurrent POSIX Mount
            ┌───────────────┴───────────────┐
            ▼                               ▼
┌───────────────────────┐       ┌───────────────────────┐
│ TASK REPLICA A        │       │ TASK REPLICA B        │
│ Volume: /mnt/shared   │       │ Volume: /mnt/shared   │
│ Task Role: App-IAM    │       │ Task Role: App-IAM    │
│ (awsvpc / VCN VNIC)   │       │ (awsvpc / VCN VNIC)   │
└───────────────────────┘       └───────────────────────┘
```

- **Persistent Shared Storage**:
  - Standard block storage (EBS / OCI Block Volume) is typically single-attach: it can only be mounted to one host at a time in a specific Availability Zone.
  - Multi-replica containers require **Distributed File Storage** (AWS EFS / OCI File Storage Service) supporting **NFS v4.1 with ReadWriteMany (RWX)** capabilities across multiple Availability Zones.
- **Container Identity Federation**:
  - Eliminates hardcoded API access keys (`AWS_ACCESS_KEY_ID` or OCI API PEM keys) from Dockerfiles and environment variables.
  - Leverages cloud metadata endpoints to inject short-lived, auto-rotating cryptographic tokens directly into the container namespace (AWS ECS Task Roles / OCI Instance Principals).

## 3. AWS Implementation Patterns
In AWS ECS and Fargate:
- **EFS Volumes in ECS Task Definitions**:
  - ECS supports native EFS volume mounts directly within the Task Definition JSON manifest `[Doc: Amazon ECS EFS Volumes, checked 2026-09-03]`.
  - Leverages **EFS Access Points** to enforce POSIX user IDs (`uid: 1000`, `gid: 1000`) and root directory sandboxing, preventing containers from accessing unauthorized directories on the shared filesystem.
  - Supports TLS in-transit encryption between Fargate container tasks and the EFS mount target.
- **AWS ECS Task Role Isolation**:
  - The ECS Container Agent injects a local link-local URI:
    ```bash
    curl 169.254.170.2$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI
    ```
  - The AWS SDK inside the container automatically queries this URI to retrieve temporary IAM credentials for the designated **Task Role**.

## 4. OCI Implementation Patterns
In OCI Container Instances & Compute:
- **OCI File Storage Service (FSS) Mounts**:
  - OCI FSS provides fully managed, multi-AD NFS v4.1 network storage.
  - In OCI Container Instances, persistent volumes can be attached directly from an FSS Mount Target located inside the private VCN.
  - Governed by OCI **Export Options** (IP-based access controls and root squashing).
- **OCI Instance Principals**:
  - Instead of managing API signing keys, the compute instance or container instance is placed into a **Dynamic Group** based on its OCID or compartment.
  - An IAM policy grants permissions to the dynamic group:
    ```sql
    Allow dynamic-group PaymentWorkers to manage objects in bucket FinancialRecords
    ```
  - The OCI SDK inside the container authenticates transparently using the local SmartNIC metadata service.

## 5. Failure Modes: Identity Leaks & Volume Locks
- **The Host IAM Inheritance Vulnerability**: On the EC2 Launch Type, if an engineer fails to assign an ECS Task Role, the container falls back to querying the host EC2 instance metadata (`169.254.169.254`). The container inherits the **EC2 Instance Profile** of the underlying host, gaining permissions to modify cloud infrastructure.
  - *Mitigation*: Block access to instance metadata from containers using iptables or enforce IMDSv2 with `http_put_response_hop_limit = 1`.
- **The NFS Stale File Handle Deadlock**: A container task crashes while holding an uncommitted write lock on a shared EFS / FSS file. When a replacement container launches, the stale NFS lock blocks file I/O, causing the new task to hang indefinitely.

## 6. Troubleshooting & Inspection Commands
1. **Verify In-Container IAM Identity**:
   ```bash
   # Inside AWS Container
   curl -s 169.254.170.2$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI | jq .
   # Inside OCI Container
   curl -s -H "Authorization: Bearer Oracle" http://169.254.169.254/opc/v2/instance/
   ```
2. **Test EFS / FSS Network Connectivity from Subnet**:
   ```bash
   nc -zv fs-12345.efs.us-east-1.amazonaws.com 2049
   ```
   If port 2049 (NFS) fails to connect, the security group on the EFS Mount Target is blocking ingress from the container's security group.

## 7. Senior Interview Question & Defense
**Question**: *You are migrating a legacy document-processing application to AWS Fargate or OCI Container Instances. The app requires 10 container replicas to simultaneously read and write PDF invoices to a shared directory. How do you design the storage and security architecture?*

**Staff-Level Defense**:
> "I implement this using **Managed Distributed File Storage paired with Sandboxed Identity Isolation**:
>
> 1. **Storage Layer (ReadWriteMany POSIX Storage)**:
>    - EBS and standard OCI Block Volumes are unsuitable because they are block devices that cannot be safely mounted read-write across multiple independent instances simultaneously.
>    - I provision an **Amazon EFS** filesystem (in AWS) or an **OCI File Storage Service (FSS)** filesystem (in OCI) spanning all Availability Domains.
>    - I configure an **EFS Access Point** with a defined POSIX identity (`uid: 1001`, `gid: 1001`) and lock the root path strictly to `/invoices`. This guarantees directory sandboxing and eliminates permission mismatches across containers.
>    - In the Task Definition / manifest, I mount the EFS volume directly to `/mnt/invoices` with transit encryption enabled.
>
> 2. **Network & Security Layer**:
>    - I deploy the containers in private subnets using `awsvpc` mode (or OCI native VNIC).
>    - The EFS / FSS Mount Target security group permits inbound TCP port 2049 **strictly from the Container Task Security Group ID**.
>
> 3. **Identity Layer**:
>    - I assign a dedicated **ECS Task Role** (or OCI Dynamic Group) granting minimum required permissions to read and write to the document database.
>    - This architecture delivers high-availability shared file access across 10 replicas with zero server management, zero static credentials, and complete network isolation."
