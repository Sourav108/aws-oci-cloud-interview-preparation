# Module 11: Serverless (AWS Lambda vs. OCI Functions)

> **Architectural Objective**: *Master event-driven Function-as-a-Service (FaaS) architectures, microVM virtualization, concurrency scaling models, and cold start mitigation. Deconstruct the mechanics of AWS Lambda (powered by Firecracker microVMs) against OCI Functions (powered by the open-source Fn Project), evaluate concurrency throttling and provisioning models, and engineer robust API Gateway and asynchronous event-driven pipelines.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. Lambda vs. OCI Functions Architecture](01-lambda-vs-oci-functions-architecture.md)** | Firecracker MicroVMs vs. Fn Project Container Runtime, Execution Lifecycle (Init/Invoke/Shutdown), Ephemeral Disk (`/tmp`), Memory & Timeout Limits | Full 20-Section Deep Dive (~2,400 words) |
| **[02. Concurrency, Cold Starts & Provisioning](02-concurrency-cold-starts-and-provisioning.md)** | Cold Start Anatomy, VPC ENI Attachment Latency, Reserved Concurrency vs. Provisioned Concurrency, Account-Level Throttling Cascades | Full 20-Section Deep Dive (~2,200 words) |
| **[03. API Gateway & Event-Driven Integrations](03-api-gateway-and-event-driven-integrations.md)** | AWS API Gateway (REST vs. HTTP APIs) vs. OCI API Gateway, Asynchronous Event Triggers (S3, Streaming), Dead-Letter Queues (DLQ) | Abbreviated Serverless Guide (~900 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **Firecracker vs. Fn Project Under the Hood**: How AWS Lambda achieves sub-5ms microVM isolation using the open-source Rust-based Firecracker hypervisor, contrasted with OCI Functions running containerized workloads on managed OKE clusters.
2. **The Concurrency Throttling Cascade**: Why setting unreserved concurrency on an unthrottled worker function can exhaust the entire account's pool of 1,000 concurrent executions, taking down customer-facing production APIs.
3. **Provisioned Concurrency Economics**: How to mathematically calculate when to migrate from on-demand cold-start execution to Provisioned Concurrency or standard container compute (Fargate / Container Instances).
