# 6-Week Cloud Engineering Interview Preparation Roadmap

> **Preparation Strategy**: *This 6-week intensive study roadmap prepares experienced backend and systems engineers for Senior and Staff Cloud Architecture, Platform Engineering, DevOps, and SRE interviews across AWS and OCI.*
>
> **Weekly Cadence**: `Theory & Deep Reading (Mon/Tue) → Architecture & Design Flow (Wed) → Hands-on Lab & Code (Thu) → Troubleshooting Drill (Fri) → Mock Interview & Revision (Sat/Sun)`

---

## 🗓️ 6-Week Master Timeline

```text
Week 1: Cloud Foundations & Core Networking (Modules 01–07)
   │
Week 2: Compute, Containers, Kubernetes & Storage (Modules 08–13)
   │
Week 3: Managed Databases, NoSQL, Caching & Messaging (Modules 14–17)
   │
Week 4: Security, KMS, Observability, Scaling & HA/DR (Modules 18–23)
   │
Week 5: Migrations, IaC, CI/CD, Performance & Cost Optimization (Modules 24–28)
   │
Week 6: 500 Interview Questions, System Design Projects & Mock Practice (Modules 29–30)
```

---

## 📅 Detailed Weekly Breakdown

### Week 1: Cloud Foundations, Virtual Networks & Traffic Routing
- **Modules Covered**: 01 (Foundations), 02 (Regions/AZs/ADs), 03 (Networking), 04 (VPC/VCN), 05 (Subnets/Gateways), 06 (DNS), 07 (Load Balancing).
- **Theory & Architecture**:
  - Compare AWS Regions/AZs with OCI Regions/ADs/Fault Domains.
  - Trace end-to-end packet flow: Internet $\to$ IGW $\to$ ALB/OCI LB $\to$ Private Subnet $\to$ NAT GW.
  - Master the differences: AWS Security Groups vs. OCI NSGs vs. OCI Security Lists.
- **Hands-On Lab**: [Lab 01: Dual-Cloud VPC & VCN Topology](labs/01-vpc-vcn-networking/).
- **Troubleshooting Drill**: Runbook RB-03 (Inter-Service Network Timeout) & RB-05 (Unhealthy LB Targets).
- **Interview Practice**: Explain how a VPC NAT Gateway works under the hood and why single-AZ NAT creates a single point of failure in AWS vs. regional resilience in OCI.

---

### Week 2: Compute, Containers, Kubernetes & Storage Architecture
- **Modules Covered**: 08 (Compute & VMs), 09 (Containers), 10 (Kubernetes: EKS vs. OKE), 11 (Serverless), 12 (Object Storage), 13 (Block & File Storage).
- **Theory & Architecture**:
  - EC2 Nitro architecture vs. OCI Flexible Shapes (`VM.Standard.E5.Flex`).
  - Cloud-managed Kubernetes: Control plane SLAs, CNI plugins (VPC CNI vs. VCN CNI), IRSA vs. OKE Workload Identity.
  - S3 strong consistency, multi-part uploads, and lifecycle tiering vs. OCI Object Storage auto-tiering.
  - EBS performance types (`gp3`, `io2`) vs. OCI Block Volume VPUs (0–120 VPUs/GB).
- **Hands-On Lab**: [Lab 02: Autoscaling & Load Balanced Compute](labs/02-compute-autoscaling-lb/) & [Lab 08: Managed Kubernetes](labs/08-kubernetes-eks-oke/).
- **Troubleshooting Drill**: Runbook RB-01 (Instance Unreachable) & RB-09 (Pod CrashLoopBackOff).
- **Interview Practice**: Defend choosing Kubernetes vs. Serverless Containers (ECS Fargate / OCI Container Instances) for a spiky microservices API.

---

### Week 3: Data Systems, Caching & Distributed Messaging
- **Modules Covered**: 14 (Managed Databases), 15 (NoSQL), 16 (Caching), 17 (Messaging & Eventing).
- **Theory & Architecture**:
  - Amazon Aurora distributed 6-way storage quorum vs. OCI Base DB / Autonomous DB Active Data Guard.
  - DynamoDB partition key design, write sharding, and hot partition mitigation vs. OCI NoSQL.
  - Cache-aside, cache stampede prevention (XFetch / mutex locks), and Redis multi-AZ failover.
  - SQS visibility timeouts, FIFO message groups, and DLQ redrive vs. OCI Queue.
- **Hands-On Lab**: [Lab 04: Managed PostgreSQL HA & Failover](labs/04-database-ha-failover/) & [Lab 05: Asynchronous Queue & DLQ](labs/05-messaging-queue-worker-dlq/).
- **Troubleshooting Drill**: Runbook RB-06 (Database Connection Pool Exhaustion) & RB-07 (Queue Backlog Surge).
- **Interview Practice**: Walk through the exact failure sequence when an Aurora primary node crashes: how long does failover take, how does DNS update, and how do client connection pools react?

---

### Week 4: Security, Secrets, Observability, Scaling & Reliability
- **Modules Covered**: 18 (IAM & Security), 19 (Secrets & KMS), 20 (Observability), 21 (Autoscaling), 22 (HA/DR), 23 (Resilience & Chaos).
- **Theory & Architecture**:
  - AWS IAM policy evaluation logic vs. OCI Compartment policy inheritance.
  - Workload identity: IAM Roles for EC2/EKS vs. OCI Dynamic Groups.
  - Envelope encryption with AWS KMS CMKs vs. OCI Vault Master Keys.
  - SLO error budget multi-burn rate alerting vs. static CPU threshold alerts.
  - Designing for RPO=0 and RTO < 60s in a multi-region deployment.
- **Hands-On Lab**: [Lab 06: Least-Privilege IAM & Workload Identity](labs/06-iam-workload-identity/) & [Lab 10: Fault Injection & Chaos](labs/10-chaos-disaster-recovery/).
- **Troubleshooting Drill**: Runbook RB-11 (IAM Authorization Denial) & RB-14 (Region Impairment).
- **Interview Practice**: Defend the trade-off between Synchronous Multi-Region Replication (zero data loss, high write latency) vs. Asynchronous Replication (sub-second latency, non-zero RPO).

---

### Week 5: Migrations, IaC, CI/CD, Performance & FinOps
- **Modules Covered**: 24 (Migrations), 25 (Terraform), 26 (CI/CD Deployments), 27 (Performance), 28 (Cost Optimization).
- **Theory & Architecture**:
  - The 6 R's of cloud migration; database cutover strategies using AWS DMS / OCI Database Migration.
  - Terraform remote state locking (S3 + DynamoDB vs. OCI Object Storage), state drift detection, and modular blast-radius containment.
  - Zero-downtime deployment pipelines: Blue/Green vs. Canary routing with automated health alarms.
  - FinOps unit economics: Compute right-sizing, Savings Plans vs. OCI Universal Credits, and eliminating NAT data processing fees.
- **Hands-On Lab**: [Lab 09: Terraform State Drift & Reconciliation](labs/09-terraform-state-drift/).
- **Troubleshooting Drill**: Runbook RB-13 (Deployment Emergency Rollback) & RB-16 (Cost Spike).
- **Interview Practice**: You discover a monthly $10,000 cloud bill surge attributed to NAT Gateway data transfer. How do you diagnose and permanently eliminate this cost?

---

### Week 6: Comprehensive Interview Drills & Reference System Design
- **Modules Covered**: 29 (500 Interview Questions: Sub-phases 29.1–29.5) & 30 (System Design Projects).
- **Focus Areas**:
  - Drill through the 500-question interview bank across all 5 sub-phases.
  - Execute full end-to-end whiteboarding for the 5 reference system design problems using the **R-A-T-D-S-R-S-O-C-F-T** framework.
  - Practice dual-cloud translation: Immediately produce both AWS and OCI architectural implementations for any design prompt.
  - Conduct full timed mock interviews with senior peers or mentors.
