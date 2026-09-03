# Cloud Security & Zero-Trust Architecture Checklist

> **Security by Design**: *Security is never an afterthought bolted onto an existing deployment. In Senior and Staff interviews, every architecture must inherently embody least-privilege identity, perimeter isolation, ubiquitous encryption, immutable audit trails, and automated threat detection.*

This checklist provides a senior-level audit framework spanning 10 critical security domains across **AWS** and **OCI**.

---

## 🛡️ The 10 Security Pillars

### 1. Identity & Access Management (IAM)
- [ ] **MFA Enforcement**: Enforce mandatory phishing-resistant Multi-Factor Authentication (FIDO2 / WebAuthn) for all human console logins.
- [ ] **Zero Root / Tenancy Admin Usage**: Lock away AWS Root account credentials and OCI Default Tenancy Administrator. Create dedicated administrative roles with temporary elevation.
- [ ] **Workload Identity (Credential-less)**:
  - AWS: Use IAM Roles attached to EC2 Instance Profiles or EKS Pod Identity (IRSA). Ban hardcoded access keys.
  - OCI: Use **Dynamic Groups** matching instance OCIDs or tags, paired with IAM policies granting permissions to the dynamic group.
- [ ] **Least Privilege Scoping**:
  - AWS: Eliminate wildcard actions (`Action: "*"`) and wildcards on resources (`Resource: "*"`). Use IAM Access Analyzer.
  - OCI: Use restrictive policy verbs (`inspect`, `read`, `use`, `manage`) scoped strictly to designated child compartments.
- [ ] **Permission Boundaries & Guardrails**:
  - AWS: Service Control Policies (SCPs) at the Organization level to prevent disabling security services (e.g., CloudTrail, GuardDuty).
  - OCI: **Security Zones** to enforce mandatory encryption, private subnet isolation, and strict compartment policies.

### 2. Network Perimeter & Traffic Isolation
- [ ] **Private Compute Default**: 100% of backend application servers, databases, and microservice containers reside in private subnets with no public IPs assigned.
- [ ] **Defense-in-Depth Firewalls**:
  - AWS: Enforce **Security Groups** at the ENI level (stateful) and **NACLs** at the subnet boundary (stateless).
  - OCI: Enforce **Network Security Groups (NSGs)** at the VNIC level. Use **Security Lists** only for baseline subnet-level packet filtering.
- [ ] **Private Cloud Service Ingress/Egress**:
  - AWS: Deploy VPC Endpoints (Gateway for S3/DynamoDB; Interface/PrivateLink for other AWS services) to eliminate NAT gateway traversal for internal APIs.
  - OCI: Deploy an **OCI Service Gateway** (`all-services-in-region`) to route internal traffic directly to Oracle services over the internal backbone.
- [ ] **WAF & DDoS Shielding**:
  - Deploy AWS WAF / OCI WAF at the edge load balancer to filter SQL injection, cross-site scripting (XSS), and rate-limit HTTP abuse.
  - Enable AWS Shield / OCI DDoS Protection for layer 3/4 volumetric mitigation.

### 3. Encryption in Transit
- [ ] **Mandatory TLS 1.3 / 1.2**: Terminate external public traffic with modern TLS ciphers only on ALB / OCI Load Balancer. Reject TLS 1.0/1.1 and insecure ciphers.
- [ ] **Internal Service mTLS**: Enforce mutual TLS (mTLS) between microservice pods in Kubernetes using service mesh (Istio, Linkerd) or Envoy sidecars.
- [ ] **Database Connection Encryption**: Force SSL/TLS on all client database connections (`rds.force_ssl=1` on AWS RDS PostgreSQL; TLS mandatory on OCI Autonomous DB).
- [ ] **Private Inter-Region Backbone**: Verify cross-region traffic flows over cloud provider private fiber networks rather than the public internet.

### 4. Encryption at Rest & Key Management
- [ ] **Customer-Managed Encryption Keys (CMEK)**:
  - AWS: Use AWS KMS with Customer Managed Keys (CMKs) featuring annual automated key rotation.
  - OCI: Use **OCI Vault** with Master Encryption Keys (MEKs) stored in dedicated Hardware Security Modules (HSM).
- [ ] **Storage Volume & Bucket Encryption**:
  - Enforce default encryption on all Amazon EBS volumes / S3 buckets and OCI Block Volumes / Object Storage buckets.
  - Deny bucket uploads that do not specify encryption headers.
- [ ] **Envelope Encryption Pattern**: Use KMS/Vault to generate short-lived Data Encryption Keys (DEKs). Encrypt data locally using the DEK, then encrypt the DEK with the Master Key and store alongside ciphertext.

### 5. Secrets Management & Credential Lifecycle
- [ ] **No Secrets in Code or Config**: Zero plain-text credentials in Git, Dockerfiles, Terraform scripts, or environment variable dumps.
- [ ] **Automated Secret Rotation**:
  - AWS: Store database credentials in AWS Secrets Manager with automated Lambda rotation functions.
  - OCI: Store secrets in OCI Vault Secrets with configured rotation rules and notifications.
- [ ] **Ephemeral Database Credentials**: Use IAM database authentication (AWS RDS IAM auth or OCI IAM token auth) where supported to eliminate static passwords entirely.

### 6. Centralized Logging & Audit Trails
- [ ] **Immutable Cloud Audit Logging**:
  - AWS: Enable multi-region **AWS CloudTrail** with log file integrity validation enabled, writing to an isolated, write-once (S3 Object Lock) security account.
  - OCI: Enable **OCI Audit** across all regions and compartments, retaining audit events for at least 365 days.
- [ ] **Network Flow Logs**: Enable VPC Flow Logs (AWS) / VCN Flow Logs (OCI) on all production subnets and route to centralized analytics (OpenSearch / SIEM).
- [ ] **Kubernetes Control Plane Audit**: Enable API server, controller-manager, and authenticator audit logs in EKS and OKE.

### 7. Continuous Threat Detection & Vulnerability Scanning
- [ ] **AI/Behavioral Threat Detection**:
  - AWS: Enable **Amazon GuardDuty** across all accounts for DNS exfiltration, compromised EC2 credentials, and anomalous IAM calls.
  - OCI: Enable **OCI Cloud Guard** and **Vulnerability Scanning Service (VSS)** with automated problem remediation responders.
- [ ] **Container Image Vulnerability Scanning**:
  - AWS: Enable Amazon ECR enhanced scanning powered by Amazon Inspector.
  - OCI: Enable OCI Container Registry vulnerability scanning on image push.
  - Enforce CI/CD pipeline gates blocking images with Critical/High CVSS scores.

### 8. Backup Integrity & Ransomware Resilience
- [ ] **Cross-Account / Cross-Region Backups**:
  - Store database and storage snapshots in a separate, isolated backup account/tenancy with restricted access.
- [ ] **WORM Storage (Write Once, Read Many)**:
  - AWS: S3 Object Lock in Compliance Mode prevents even root/admin users from deleting backups before the retention period expires.
  - OCI: Object Storage Retention Rules with locked duration.
- [ ] **Continuous Restoration Drills**: Automated scripts regularly restore snapshots to isolated sandbox environments to prove recoverability.

### 9. Governance, Posture & Compliance
- [ ] **Automated Configuration Drift Auditing**:
  - AWS: Deploy **AWS Config** conformance packs (CIS AWS Foundations Benchmark).
  - OCI: Deploy OCI Cloud Guard security recipes aligned with CIS Oracle Cloud Infrastructure Foundations.
- [ ] **Infrastructure as Code Security Linting**:
  - Integrate static analysis (tfsec, checkov, trivy) in CI pipelines to block non-compliant Terraform PRs before deployment.

### 10. Incident Response & Containment Readiness
- [ ] **Automated Quarantine Workflows**: Pre-built Lambda / OCI Function responders capable of instantly swapping an instance's Security Group to an empty quarantine group and capturing a memory snapshot upon threat detection.
- [ ] **Break-Glass Procedures**: Documented, audited, and time-limited emergency elevation access with real-time alerting to security operations (SecOps).
- [ ] **Blast Radius Containment**: Compartmentalize workloads so that compromise of one service or account does not compromise the broader tenancy.
