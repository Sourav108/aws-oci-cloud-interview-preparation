# Curriculum Dependency Graph & Knowledge Architecture

> **Mastery Sequence**: *Cloud infrastructure engineering is inherently hierarchical. You cannot design a secure Kubernetes cluster without mastering VPC subnets and IAM; you cannot engineer multi-region disaster recovery without understanding asynchronous database replication and global DNS routing.*

---

## 🗺️ Architectural Learning Flow

```text
[01. Foundations] ──> [02. Regions/AZs/ADs] ──> [03. Networking Fundamentals]
                                                              │
                                                              ▼
                                                   [04. VPC & VCN Topology]
                                                              │
                                                              ▼
                                                   [05. Subnets, Routing & Gateways]
                                                              │
                                                              ▼
                                                   [06. DNS & Service Discovery]
                                                              │
                                                              ▼
                                                   [07. Load Balancing (ALB/NLB)]
                                                              │
                         ┌────────────────────────────────────┴────────────────────────────────────┐
                         ▼                                                                         ▼
           [08. Compute & VMs (EC2/OCI)]                                             [12. Object Storage (S3/OCI)]
                         │                                                                         │
                         ▼                                                                         ▼
           [09. Containers & Registries]                                             [13. Block & File Storage]
                         │                                                                         │
                         ▼                                                                         ▼
           [10. Kubernetes (EKS & OKE)]                                              [14. Managed Relational DBs]
                         │                                                                         │
                         ▼                                                                         ▼
           [11. Serverless Compute]                                                  [15. NoSQL & Distributed Data]
                         │                                                                         │
                         └────────────────────────────────────┬────────────────────────────────────┘
                                                              ▼
                                            [16. In-Memory Caching (Redis)]
                                                              │
                                                              ▼
                                            [17. Messaging & Eventing (Queues/PubSub)]
                                                              │
                         ┌────────────────────────────────────┴────────────────────────────────────┐
                         ▼                                                                         ▼
           [18. IAM & Workload Identity]                                             [20. Observability & Telemetry]
                         │                                                                         │
                         ▼                                                                         ▼
           [19. Secrets & KMS (Envelope Enc)]                                        [21. Autoscaling & Capacity]
                         │                                                                         │
                         └────────────────────────────────────┬────────────────────────────────────┘
                                                              ▼
                                            [22. High Availability & Disaster Recovery]
                                                              │
                                                              ▼
                                            [23. Resilience & Fault Tolerance]
                                                              │
                                                              ▼
                                            [24. Cloud Migrations & Modernization]
                                                              │
                                                              ▼
                                            [25. Infrastructure as Code (Terraform)]
                                                              │
                                                              ▼
                                            [26. CI/CD & Deployment Pipelines]
                                                              │
                         ┌────────────────────────────────────┴────────────────────────────────────┐
                         ▼                                                                         ▼
           [27. Cloud Performance Optimization]                                      [28. Cloud Cost Optimization]
                         │                                                                         │
                         └────────────────────────────────────┬────────────────────────────────────┘
                                                              ▼
                                            [29. 500 Cloud Interview Questions]
                                                              │
                                                              ▼
                                            [30. Cloud System Design & Reference Architectures]
```

---

## 🧬 Parallel Engineering Specialization Tracks

While progressing through the linear foundation, three cross-cutting disciplines must be practiced continuously:

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        PARALLEL TRACK 1: SECURITY & ZERO TRUST                         │
│  [Sec in VPC/VCN] ──> [Workload Identity] ──> [KMS Envelope Enc] ──> [Cloud Guard/WAF] │
└────────────────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        PARALLEL TRACK 2: OBSERVABILITY & RUNBOOKS                      │
│  [Flow Logs] ──> [ALB/LB Health] ──> [RDS Diagnostics] ──> [16 Production Runbooks]    │
└────────────────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        PARALLEL TRACK 3: HANDS-ON LABS & TERRAFORM                     │
│  [Lab 01: VPC] ──> [Lab 02: Compute] ──> [Lab 04: DB HA] ──> [Lab 10: Chaos DR]       │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📊 Module Prerequisite Matrix

| Module | Name | Direct Prerequisites | Unlocks Next |
| :---: | :--- | :--- | :--- |
| **01** | Cloud Foundations | None | 02, 03 |
| **02** | Regions, AZs & Infrastructure | 01 | 04, 22 |
| **03** | Networking Fundamentals | 01 | 04, 05, 07 |
| **04** | VPC & VCN Topology | 02, 03 | 05, 07, 18 |
| **05** | Subnets, Routing & Gateways | 04 | 06, 07, 08 |
| **06** | DNS & Service Discovery | 03, 05 | 07, 22 |
| **07** | Load Balancing | 05, 06 | 08, 09, 10 |
| **08** | Compute & VMs | 05, 07 | 09, 13, 21 |
| **09** | Containers & Registries | 08 | 10, 11 |
| **10** | Kubernetes (EKS & OKE) | 07, 09, 18 | 26, 30 |
| **11** | Serverless Functions | 07, 09 | 17, 30 |
| **12** | Object Storage | 04, 18 | 13, 25 |
| **13** | Block & File Storage | 08, 12 | 14, 22 |
| **14** | Managed Databases | 05, 13, 18 | 15, 16, 22 |
| **15** | NoSQL & Distributed Data | 14 | 16, 30 |
| **16** | In-Memory Caching | 14, 15 | 27, 30 |
| **17** | Messaging & Eventing | 08, 11 | 23, 30 |
| **18** | IAM & Cloud Security | 01, 04 | 19, 20, 25 |
| **19** | Secrets & Key Management | 18 | 14, 25, 26 |
| **20** | Observability & Telemetry | 07, 08, 14 | 21, 27 |
| **21** | Autoscaling & Capacity | 08, 17, 20 | 22, 27 |
| **22** | High Availability & DR | 02, 14, 21 | 23, 30 |
| **23** | Resilience & Chaos | 17, 21, 22 | 27, 30 |
| **24** | Cloud Migrations | 14, 18, 22 | 25, 30 |
| **25** | Infrastructure as Code | 04, 18, 19 | 26, 30 |
| **26** | CI/CD & Deployments | 10, 25 | 27, 30 |
| **27** | Performance Optimization | 16, 20, 21 | 28, 30 |
| **28** | Cost Optimization & FinOps | 08, 12, 14, 25 | 30 |
| **29** | 500 Interview Questions | 01–28 | 30 |
| **30** | System Design & Projects | 01–29 | Capstone |
