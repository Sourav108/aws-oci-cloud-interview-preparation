# Hands-On Cloud Engineering Labs Catalog & Cost Governance

> **Implementation-First Pedagogy**: *Theory without hands-on infrastructure validation is shallow. Every lab in this catalog provides real, executable Terraform infrastructure code, failure injection scenarios, step-by-step debugging walkthroughs, and explicit teardown commands.*

---

## 🔒 Hard Cost Gates & Execution Safety Standard

To protect engineers from unexpected cloud billing surges, every lab in this repository enforces strict financial and execution guardrails:

1. **Sub-$1/Hour & Free-Tier Design Default**:
   - All lab Terraform templates default to free-tier eligible or micro/burstable instance types (e.g., AWS `t4g.nano` / `t3.micro`, OCI `VM.Standard.E5.Flex` with 1 OCPU / 4 GB RAM or Always-Free `VM.Standard.A1.Flex`).
2. **Boxed Cost Warning Gate**:
   - If an architecture genuinely requires non-free tier resources (e.g., managed Kubernetes control plane, multi-AZ database cluster, Aurora), the lab documentation **must open with a prominent boxed cost warning** detailing estimated hourly and cumulative costs before any CLI commands.
3. **Zero Automated `terraform apply` Rule**:
   - Automated scripts and agents are strictly forbidden from executing `terraform apply` against live cloud accounts without explicit real-time human approval.
4. **Mandatory Teardown Section**:
   - Every lab concludes with a clean, validated `terraform destroy` command block and verification checklist. No resources may be left running as a default path.

---

## 🛡️ No-Live-Account Fallback Mode

> If you do not possess live AWS or OCI cloud accounts, or prefer not to spend cloud credits, **all labs fully support No-Live-Account Fallback Mode**.

When operating in fallback mode:
- All Terraform configuration files (`main.tf`, `variables.tf`, `outputs.tf`) are written and statically verified using `terraform validate`.
- Architecture topology diagrams and packet flow sequences are fully rendered in ASCII/Mermaid.
- The lab documentation narrates the exact expected output of `terraform plan`, simulated curl responses, failure logs, and metric telemetry.
- Labs executed in this mode are explicitly labeled:
  ```text
  [Statically validated — not applied to a live account]
  ```

---

## 🧪 Hands-On Lab Index

| Lab ID & Directory | Architectural Domain | Dual-Cloud Technologies | Estimated Run Cost | Focus & Interview Skills Tested |
| :--- | :--- | :--- | :---: | :--- |
| **[Lab 01: Dual-Cloud VPC & VCN Topology](labs/01-vpc-vcn-networking/)** | Networking | AWS VPC vs. OCI VCN, Subnets, Gateways, Route Tables | `< $0.05/hr` `[Approximation]` | Build isolated 3-tier subnets, test cross-subnet packet flow, verify security group vs NSG packet filtering. |
| **[Lab 02: Autoscaling & Load Balanced Compute](labs/02-compute-autoscaling-lb/)** | Compute & Scaling | AWS ALB + EC2 ASG vs. OCI LB + Instance Pool | `< $0.15/hr` `[Approximation]` | L7 path routing, health check timeouts, synthetic CPU load injection, target tracking autoscaling. |
| **[Lab 03: Object Storage Lifecycle & Encryption](labs/03-object-storage-lifecycle/)** | Storage | AWS S3 vs. OCI Object Storage, KMS/Vault | `< $0.02/hr` `[Approximation]` | CMEK encryption at rest, pre-signed URL generation, automated lifecycle tiering transitions. |
| **[Lab 04: Managed PostgreSQL HA & Failover](labs/04-database-ha-failover/)** | Databases | AWS Aurora PG vs. OCI Base Database System | `< $0.45/hr` `[Approximation]` | Multi-AZ replication, connection pool configuration, simulating primary crash and measuring RTO. |
| **[Lab 05: Asynchronous Queue & Dead-Letter Processing](labs/05-messaging-queue-worker-dlq/)** | Messaging | AWS SQS + Worker vs. OCI Queue + Functions | `< $0.05/hr` `[Approximation]` | Poison pill message injection, visibility timeout calibration, DLQ redrive, consumer backpressure. |
| **[Lab 06: Zero-Trust Least-Privilege IAM & Workload Identity](labs/06-iam-workload-identity/)** | Security | AWS IAM Roles/IRSA vs. OCI Dynamic Groups | `$0.00 (Free)` | Credential-less compute authentication, compartment boundary containment, policy debugging. |
| **[Lab 07: Distributed Observability & Alarms](labs/07-observability-telemetry/)** | Observability | CloudWatch / X-Ray vs. OCI Monitoring / APM | `< $0.10/hr` `[Approximation]` | OpenTelemetry sidecar deployment, structured JSON logging, high-error burn-rate alert triggers. |
| **[Lab 08: Managed Kubernetes Ingress & Connectivity](labs/08-kubernetes-eks-oke/)** | Kubernetes | AWS EKS vs. OCI OKE, CNI, Ingress Controller | `< $0.35/hr` `[Approximation]` | Pod networking troubleshooting, NetworkPolicy enforcement, NodePort vs LoadBalancer ingress. |
| **[Lab 09: Infrastructure as Code & Drift Detection](labs/09-terraform-state-drift/)** | IaC / DevOps | Terraform, S3/OCI Remote State, State Locking | `< $0.05/hr` `[Approximation]` | Remote state locking with DynamoDB/Object Storage, injecting out-of-band drift, automated reconciliation. |
| **[Lab 10: Fault Injection, Chaos & Disaster Recovery](labs/10-chaos-disaster-recovery/)** | Reliability | Route 53 / OCI DNS Steering, Multi-Region DR | `< $0.30/hr` `[Approximation]` | Simulating complete AZ blackhole, automated DNS health check failover, measuring actual RPO and RTO. |

---

## 📋 Standard Lab Structure

Every lab in `labs/` adheres to a strict 11-section format:

1. **Goal & Core Concept**: The specific engineering outcome and mental model.
2. **Cost Warning & Resource Estimate**: Detailed pricing breakdown per resource.
3. **Architecture Diagram**: Visual ASCII / Mermaid representation of components.
4. **Prerequisites**: Required CLI tools and permissions.
5. **Infrastructure Code**: Complete, copy-pasteable Terraform configuration files.
6. **Step-by-Step Deployment Guide**: Commands to plan and deploy.
7. **Expected Validation Results**: Exact curl, CLI, or UI outputs confirming healthy state.
8. **Failure Injection Drill**: Controlled disruption (terminating nodes, corrupting configs, dropping routes).
9. **Debugging Walkthrough**: Identifying the root cause using metrics, logs, and flow logs.
10. **Teardown & Verification**: Explicit teardown commands to guarantee zero orphaned resources.
11. **Senior Interview Defense & Takeaways**: How this architecture is discussed and defended in interviews.
