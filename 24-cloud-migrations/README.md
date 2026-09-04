# Module 24: Cloud Migrations & The 6 R's Framework (AWS vs. OCI)

---

## 1. Module Overview & Learning Objectives

Enterprise cloud migration is not merely a technical exercise of copying virtual machine disk images and database rows across a wide-area network. It is an organizational, financial, and architectural transformation. Migrating hundreds or thousands of legacy on-premises applications to the cloud requires a structured methodology to inventory assets, analyze dependencies, evaluate total cost of ownership (TCO), and select the optimal migration strategy for each workload.

This module provides comprehensive technical and architectural mastery over enterprise workload migration across Amazon Web Services (AWS) and Oracle Cloud Infrastructure (OCI).

### What You Will Master
1. **The 6 R's Migration Framework**: Categorizing applications into Rehost, Replatform, Repurchase, Refactor, Retain, and Retire.
2. **Discovery & Portfolio Assessment**: Operating AWS Application Discovery Service & Migration Hub vs. OCI Cloud Migration Service.
3. **Block-Level Server Migration**: Configuring continuous block replication with AWS Application Migration Service (AWS MGN) and OCI Cloud Migration.
4. **Zero-Downtime Database Migration**: Running continuous Change Data Capture (CDC) with AWS Database Migration Service (DMS) + Schema Conversion Tool (SCT) vs. OCI GoldenGate and Zero Downtime Migration (ZDM).
5. **Petabyte-Scale Physical & Network Transport**: Mobilizing AWS Snowball Edge and DataSync vs. OCI Data Transfer Service (DTS Appliance & Disk).
6. **Production Cutover Windows**: Orchestrating maintenance windows, DNS TTL dissipation, and emergency fallback reversibility.

---

## 2. Directory Roadmap & Lesson Catalog

```
24-cloud-migrations/
├── README.md                                                  # Module guide & architectural index
├── 01-migration-strategies-the-6-rs-framework.md             # [Major] The 6 R's, AWS MGN vs OCI Cloud Migration, wave planning
├── 02-database-migration-and-continuous-replication.md       # [Major] AWS DMS/SCT vs OCI GoldenGate/ZDM, CDC, schema conversion
└── 03-large-scale-data-transfer-and-cutover.md               # [Supporting] Snowball vs OCI DTS, DataSync, cutover runbook
```

---

## 3. The 6 R's Migration Decision Tree

```
                                  [ Existing Workload / Application ]
                                                  |
                     +----------------------------+----------------------------+
                     | Is business value retained?                             | NO
                     v YES                                                     v
          Is modern SaaS available?                                    [ RETIRE / DECOMMISSION ]
          +----------+----------+
          | YES                 | NO
          v                     v
   [ REPURCHASE (SaaS) ]   Does it run on modern supported OS/DB?
                           +----------+----------+
                           | YES                 | NO
                           v                     v
                 Can we rewrite now?       Does it need cloud-native features?
                 +----+----+               +----------+----------+
                 | YES     | NO            | YES                 | NO
                 v         v               v                     v
            [ REFACTOR ]  [ REHOST ]  [ REPLATFORM ]       [ RETAIN (Keep on-prem) ]
            (Cloud-Native) (Lift & Shift) (Managed Services)
```

---

## 4. Side-by-Side Dual-Cloud Migration Primitives

| Migration Domain | AWS Native Primitives | OCI Native Primitives |
| :--- | :--- | :--- |
| **Portfolio Discovery** | AWS Application Discovery Service / Migration Hub | OCI Cloud Migration Service Discovery Agent |
| **Server / VM Migration** | AWS Application Migration Service (AWS MGN) | OCI Cloud Migration Service (Replication Appliance) |
| **Heterogeneous DB Migration**| AWS Database Migration Service (DMS) + SCT | OCI GoldenGate Managed Service |
| **Homogeneous Oracle DB** | AWS DMS / Native Oracle Data Pump | OCI Zero Downtime Migration (ZDM) Physical/Logical |
| **Physical Data Appliance** | AWS Snowcone (8TB), Snowball Edge (80TB), Snowmobile | OCI Data Transfer Appliance (150TB) & Transfer Disk |
| **Network Data Transfer** | AWS DataSync / AWS Transfer Family | OCI Data Transfer over Network / FastConnect |
| **Cutover Traffic Steering** | Route 53 DNS / AWS Global Accelerator | OCI Traffic Management Steering Policies |

---

## 5. Staff-Level Engineering Scenarios Covered

* **Zero-Downtime Cutover Orchestration**: Synchronizing database CDC pipelines with server replication to execute enterprise cutovers in under 15 minutes of scheduled downtime.
* **The "Two-Way Door" Rollback Strategy**: Implementing reverse Change Data Capture (CDC) pointing from cloud target back to on-premises source during the first 72 hours of post-cutover operation.
* **Complex Heterogeneous Schema Conversion**: Refactoring proprietary Oracle PL/SQL packages, stored procedures, and triggers to open-source PostgreSQL (Aurora / OCI Base DB).
