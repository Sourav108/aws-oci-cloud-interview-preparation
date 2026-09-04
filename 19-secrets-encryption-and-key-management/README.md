# Module 19: Security, KMS & Key Management

> **Architectural Objective**: *Master cloud cryptography, envelope encryption, hardware security modules (HSMs), automated secrets lifecycle management, and enterprise compliance architectures. Deconstruct AWS KMS and Secrets Manager against OCI Vault and Certificates Service, evaluate cryptographic key hierarchy models (KEK vs. DEK), and architect resilient zero-trust encryption at rest and in transit.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. Envelope Encryption & KMS Architecture](01-envelope-encryption-and-kms-architecture.md)** | Cryptographic Mechanics (Envelope Encryption, DEK vs. KEK/CMK), AWS KMS (Customer Managed, AWS Managed, Grants) vs. OCI Vault (Software vs. HSM FIPS 140-2 Level 3), CloudHSM | Full 20-Section Deep Dive (~2,400 words) |
| **[02. Secrets Management & Certificate Authority](02-secrets-management-and-certificate-authority.md)** | AWS Secrets Manager (Auto-Rotation, Replication) vs. SSM Parameter Store vs. OCI Vault Secrets, AWS Private CA & ACM vs. OCI Certificates (mTLS), SOC 2 / HIPAA / PCI-DSS Frameworks | Full 20-Section Deep Dive (~2,300 words) |
| **[03. Cryptographic Performance & Failure Modes](03-cryptographic-performance-and-failure-modes.md)** | KMS API Throttling Cascades, Client-Side Data Key Caching (AWS Encryption SDK), Key Policy Lockouts, Secret Rotation Desynchronization | Abbreviated Cryptography Guide (~900 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **The Envelope Encryption Mathematical Protocol**: Why transmitting raw multi-gigabyte payloads across the network to KMS is an anti-pattern, and how Data Encryption Keys (DEKs) provide line-rate encryption without KMS throughput bottlenecks.
2. **Secrets Manager vs. Parameter Store Decision Matrix**: When to choose AWS Secrets Manager (dynamic rotation, cross-region replication) versus Systems Manager Parameter Store (free standard tier, configuration flags).
3. **KMS Throttling & Client-Side Caching**: How to prevent massive auto-scaling spikes from exhausting regional KMS API quotas ($5,500\text{ req/sec}$) by implementing cryptographic data key caching.
