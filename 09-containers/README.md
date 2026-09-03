# Module 09: Containers (ECS/ECR vs. Container Instances/OCIR)

> **Architectural Objective**: *Master cloud-native container execution, image lifecycle management, and container networking. Deconstruct the mechanics of Amazon Elastic Container Service (ECS) and AWS Fargate against OCI Container Instances, evaluate container registries (AWS ECR vs. OCI Registry OCIR), and engineer robust container networking and security policies.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. ECS Architecture & Launch Types](01-ecs-architecture-and-launch-types.md)** | ECS Primitives (Clusters, Task Definitions, Services), EC2 Launch Type vs. AWS Fargate, Task Networking Modes (`awsvpc` vs. `bridge`) | Full 20-Section Deep Dive (~2,200 words) |
| **[02. OCI Container Instances & Registry](02-oci-container-instances-and-registry.md)** | Serverless Container Compute in VCNs, Zero-Cluster Overhead, OCI Registry (OCIR) vs. AWS ECR (Scanning, Lifecycle, Replication) | Full 20-Section Deep Dive (~2,200 words) |
| **[03. Container Networking & Storage Patterns](03-container-networking-and-storage-patterns.md)** | Persistent Storage (EFS / FSS Volume Mounts), Task Execution Roles vs. Task IAM Roles, Instance Principals | Abbreviated Container Guide (~900 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **Serverless Containers vs. Kubernetes**: When to choose lightweight serverless container services (AWS Fargate / OCI Container Instances) over full Kubernetes clusters (EKS/OKE), and how to justify the operational trade-offs.
2. **The `awsvpc` Networking Model**: Why the `awsvpc` networking mode is the mandatory enterprise standard for microsegmentation and security, and how it impacts ENI attachment limits on container hosts.
3. **Container Identity Isolation**: The critical architectural difference between an ECS **Task Execution Role** (used by the cloud agent to pull images and write logs) and a **Task Role** (used by the application code inside the container to access databases and S3).
