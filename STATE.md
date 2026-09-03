# Repository State Tracking

This document is the **single source of truth** for multi-session continuity and execution tracking across the entire `aws-oci-cloud-interview-preparation` repository build.

Before initiating any work in a session, read this file and `COVERAGE_MATRIX.md` first. Never redo a phase marked complete without explicit instruction; never skip ahead of what is listed in `Next action`.

---

## Current Status

- **Last completed phase**: Phase 1 — Modules 1–3 (`feat: add cloud foundations`)
- **Last commit hash/message**: Pending commit (`feat: add cloud foundations`)
- **Modules fully complete**:
  - Module 01: Cloud Foundations & Shared Responsibility (3 lessons + README)
  - Module 02: Regions, AZs, ADs & Global Infrastructure (3 lessons + README)
  - Module 03: Networking Fundamentals & Packet Flow (3 lessons + README)
- **Modules in progress**: none
- **Module 29 sub-phase reached**: none
- **Known gaps / deferred items**: none
- **Next action**: Phase 2 — Modules 4–7 (`feat: add cloud networking`)

---

## Progress Dashboard

| Dimension | Completed | Target | Status |
| :--- | :---: | :---: | :--- |
| **Numbered Modules (01–30)** | 3 | 30 | Modules 01–03 Complete |
| **AWS & OCI Bilingual Coverage** | 3 | 30 Modules | Modules 01–03 Bilingual Complete |
| **Module 29 Interview Questions** | 0 | 500 Questions | Pending Phase 9 (Sub-phases 29.1–29.5) |
| **Production Runbooks** | 0 | 16 Scenarios | Initialized in `PRODUCTION_RUNBOOKS.md` |
| **Hands-On Labs** | 0 | 10 Labs | Cataloged in `LABS.md` |
| **Cloud System Design Projects** | 0 | 8 Projects | Pending Phase 10 |

---

## Phase Execution Plan & Checklist

- [x] **Phase 0**: Repository Initialization (§30) (`chore: initialize AWS OCI cloud interview curriculum`)
- [x] **Phase 1**: Modules 01–03 (`feat: add cloud foundations`)
  - [x] 01: Cloud Foundations
  - [x] 02: Regions, AZs and Global Infrastructure
  - [x] 03: Networking Fundamentals
- [ ] **Phase 2**: Modules 04–07 (`feat: add cloud networking`)
  - 04: VPC and Virtual Cloud Network
  - 05: Subnets, Routing and Gateways
  - 06: DNS and Service Discovery
  - 07: Load Balancing
- [ ] **Phase 3**: Modules 08–13 (`feat: add cloud compute and storage`)
  - 08: Compute and Virtual Machines
  - 09: Containers
  - 10: Kubernetes (Cloud-Managed Lens: EKS vs OKE)
  - 11: Serverless (Lambda vs Functions)
  - 12: Object Storage (S3 vs OCI Object Storage)
  - 13: Block and File Storage (EBS/EFS vs Block Volume/File Storage)
- [ ] **Phase 4**: Modules 14–17 (`feat: add cloud data and messaging`)
  - 14: Managed Databases (RDS/Aurora vs Base DB/Autonomous DB)
  - 15: NoSQL and Distributed Data (DynamoDB vs OCI NoSQL)
  - 16: Caching (ElastiCache vs OCI Cache)
  - 17: Messaging and Eventing (SQS/SNS/EventBridge vs Queue/Notifications/Streaming)
- [ ] **Phase 5**: Modules 18–20 (`feat: add cloud security and observability`)
  - 18: IAM and Cloud Security
  - 19: Secrets, Encryption and Key Management
  - 20: Observability, Monitoring and Logging
- [ ] **Phase 6**: Modules 21–23 (`feat: add scaling and reliability`)
  - 21: Autoscaling and Capacity
  - 22: High Availability and Disaster Recovery
  - 23: Resilience and Fault Tolerance
- [ ] **Phase 7**: Modules 24–26 (`feat: add migration, iac, cicd`)
  - 24: Cloud Migrations
  - 25: Infrastructure as Code (Terraform)
  - 26: CI/CD and Cloud Deployments
- [ ] **Phase 8**: Modules 27–28 (`feat: add cloud optimization`)
  - 27: Cloud Performance Optimization
  - 28: Cloud Cost Optimization
- [ ] **Phase 9**: Module 29 Sub-Phases (500 Questions)
  - [ ] **Sub-phase 29.1**: Fundamentals, AWS Core, OCI Core, Networking, Compute (125 Qs) (`feat: add interview questions foundations networking compute`)
  - [ ] **Sub-phase 29.2**: Storage, Databases, Caching, Messaging, Kubernetes (125 Qs / Total: 250) (`feat: add interview questions storage data messaging k8s`)
  - [ ] **Sub-phase 29.3**: Serverless, IAM, Security, Observability, Autoscaling (125 Qs / Total: 375) (`feat: add interview questions serverless security observability`)
  - [ ] **Sub-phase 29.4**: HA, DR, Resilience, Terraform, CI/CD (100 Qs / Total: 475) (`feat: add interview questions ha dr iac cicd`)
  - [ ] **Sub-phase 29.5**: Performance, Cost, Troubleshooting, Architecture (25 Qs / Total: 500) (`feat: add interview questions performance cost architecture`)
- [ ] **Phase 10**: Module 30 + Projects (`feat: add cloud system design and projects`)
- [ ] **Phase 11**: Hands-on Labs Implementation (`feat: add cloud labs`)

---

## Session Continuity Rules

1. **Verify State Before Execution**: Always inspect `STATE.md` and `COVERAGE_MATRIX.md` before executing any phase.
2. **Strict Phase Adherence**: Execute only the phase indicated by `Next action`. Do not skip ahead or bundle multiple phases without explicit user instruction.
3. **Commit Gate**: Every phase ends with:
   - Updating `STATE.md` and `COVERAGE_MATRIX.md`.
   - Running `git status` and `git diff --check`.
   - Static syntax validation of all written `.tf` files (`terraform validate` if CLI present).
   - Clean git commit matching the phase commit message pattern.
4. **Zero Billable Apply Without Approval**: Never execute `terraform apply` or live provisioning without explicit user confirmation and pre-stated cost estimate.
