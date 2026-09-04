# Module 22: High Availability & Disaster Recovery (AWS vs. OCI)

---

## 1. Module Overview & Learning Objectives

Designing for failure is the foundational tenet of modern cloud architecture. Hardware fails, subsea fiber cables are severed, natural disasters strike data centers, and human operator errors trigger widespread control-plane outages. High Availability (HA) minimizes downtime within a region, while Disaster Recovery (DR) guarantees business continuity across regional boundaries or cloud providers when catastrophic events occur.

This module provides deep architectural, operational, and implementation mastery of HA and DR strategies across Amazon Web Services (AWS) and Oracle Cloud Infrastructure (OCI).

### What You Will Master
1. **The DR Spectrum & RPO/RTO Calibration**: Architecting across Backup & Restore, Pilot Light, Warm Standby, and Multi-Region Active-Active topologies.
2. **Database & Storage Cross-Region Replication**: Synchronous vs. Asynchronous replication, AWS Aurora Global Database vs. OCI Active Data Guard (ADG) and Autonomous Data Guard.
3. **Automated DR Orchestration**: Operating AWS Route 53 Application Recovery Controller (ARC) and OCI Full Stack Disaster Recovery (FSDR).
4. **Immutable Backup Governance & Ransomware Defense**: Enforcing WORM (Write Once Read Many) storage with AWS Backup Vault Lock and OCI Immutable Retention Rules.
5. **Game Days & Failover Drills**: Running production failover simulations, testing split-brain fencing, and managing DNS TTL cache dissipation.

---

## 2. Directory Roadmap & Lesson Catalog

```
22-high-availability-and-disaster-recovery/
├── README.md                                             # Module guide & architectural index
├── 01-ha-and-dr-architectures-and-rpo-rto.md             # [Major] DR tiers, RPO/RTO curves, Aurora Global vs Active Data Guard
├── 02-backup-governance-and-ransomware-vault-locks.md   # [Major] 3-2-1-1-0 rule, AWS Backup Vault Lock vs OCI Immutable Rules
└── 03-dr-orchestration-and-failover-testing.md           # [Supporting] Route 53 ARC, OCI FSDR, Game Days & split-brain fencing
```

---

## 3. The DR Spectrum: Trade-Off Matrix

```
RPO / RTO
   ^
   |  [ Tier 1: Backup & Restore ]  (RPO: Hours/Days, RTO: 24h+, Cost: $)
   |          \
   |           [ Tier 2: Pilot Light ]  (RPO: Minutes, RTO: 1-4h, Cost: $$)
   |                     \
   |                      [ Tier 3: Warm Standby ]  (RPO: Seconds, RTO: Minutes, Cost: $$$)
   |                                \
   |                                 [ Tier 4: Active-Active Multi-Region ] (RPO: ~0, RTO: ~0, Cost: $$$$)
   +----------------------------------------------------------------------------------------------------> Cost / Complexity
```

---

## 4. Side-by-Side Dual-Cloud HA/DR Primitives

| Reliability Domain | AWS Native Primitives | OCI Native Primitives |
| :--- | :--- | :--- |
| **Zonal HA** | Multi-AZ Deployments (3+ AZs/Region) | Multi-AD Regions + 3 Fault Domains (FDs) per AD |
| **DR Orchestration** | Route 53 Application Recovery Controller (ARC) | OCI Full Stack Disaster Recovery (FSDR) |
| **Relational DB Replication** | Aurora Global Database / RDS Cross-Region Read Replicas | OCI Active Data Guard (ADG) / Autonomous Data Guard |
| **Block Volume Replication** | AWS EBS Snapshot Copy (Asynchronous scheduled) | OCI Cross-Region Asynchronous Block Volume Replication |
| **Object Storage Replication**| S3 Cross-Region Replication (CRR) / S3 RTC (15 min) | OCI Object Storage Cross-Region Replication |
| **Immutable WORM Backups** | AWS Backup Vault Lock (Compliance & Governance modes) | OCI Object Storage Retention Rules / Locked Buckets |
| **Global DNS Steering** | Route 53 Traffic Flow (Latency, Failover, Geoproximity) | OCI Traffic Management Steering Policies (Failover, Load) |

---

## 5. Staff-Level Engineering Scenarios Covered

* **Zero-RPO Cross-Region Consistency**: Evaluating physical latency speed-of-light limits ($c \approx 5 \text{ ms per 1,000 km}$) against synchronous commit latency penalties.
* **Split-Brain Prevention**: Deploying fencing tokens, distributed consensus quorums, and health checks to prevent two regions from acting as simultaneous write primaries.
* **Ransomware Blast Radius Containment**: Using cross-account immutable vaults and strict KMS deletion holds to survive root-account credential compromise.
