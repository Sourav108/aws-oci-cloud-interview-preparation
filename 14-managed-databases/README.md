# Module 14: Managed Databases (RDS/Aurora vs. Base DB/Autonomous DB)

> **Architectural Objective**: *Master enterprise relational database architectures in the cloud, storage engine disaggregation, consensus replication, high-availability failover mechanics, and automated operations. Deconstruct Amazon RDS and Amazon Aurora against OCI Base Database Service and Oracle Autonomous Database (ATP/ADW), evaluate distributed quorum storage models (Aurora 6-way storage vs. Exadata Smart Scans), and master zero-data-loss disaster recovery.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. RDS & Aurora Architecture](01-rds-and-aurora-architecture.md)** | Amazon RDS Multi-AZ vs. Aurora Log-Structured Distributed Storage (6 copies, 4/6 write quorum), Global Database, RDS Proxy | Full 20-Section Deep Dive (~2,400 words) |
| **[02. OCI Base Database & Autonomous Database](02-oci-base-database-and-autonomous-database.md)** | Base DB on ASM, Autonomous Database (ATP vs. ADW) Self-Driving Engine, Exadata Smart Scans & RoCE, Active Data Guard (FSFO) | Full 20-Section Deep Dive (~2,400 words) |
| **[03. Database Failover & Backup Recovery](03-database-failover-and-backup-recovery.md)** | Failover Mechanics (DNS CNAME vs. VIP Swapping vs. Data Guard), Point-in-Time Recovery (PITR) WAL Archiving, Split-Brain Mitigation | Abbreviated Database Guide (~950 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **The Log-Structured Storage Revolution**: How Amazon Aurora decoupled database compute from storage to eliminate write amplification and crash recovery redo logging, achieving 5x PostgreSQL/MySQL throughput.
2. **Exadata Smart Scan Offloading**: How Oracle Autonomous Database and Exadata push SQL `WHERE` clause predicates and column projections down to the physical storage cells over 100 Gbps RoCE networks, slashing network data transfer by 90%.
3. **Database Failover & Connection Management**: How to architect connection pooling layers (Amazon RDS Proxy / Oracle CMAN) to prevent connection storms and reduce failover reconnection times from minutes to seconds.
