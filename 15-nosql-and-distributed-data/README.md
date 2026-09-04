# Module 15: NoSQL & Distributed Data (DynamoDB vs. OCI NoSQL)

> **Architectural Objective**: *Master hyperscale distributed NoSQL database architectures, partition-based horizontal scaling, single-table data modeling, capacity economics, and change data capture. Deconstruct Amazon DynamoDB against OCI NoSQL Database Service, master Paxos consensus replication, evaluate capacity modes (Provisioned vs. On-Demand), and engineer resilient multi-region active-active architectures.*

---

## 📑 Module Overview & Lessons

| Lesson | Focus Area | Architectural Depth |
| :--- | :--- | :--- |
| **[01. DynamoDB Architecture & Data Modeling](01-dynamodb-architecture-and-data-modeling.md)** | Storage Nodes & Paxos Consensus, Partition & Sort Keys, Single-Table Design Principles, GSIs vs. LSIs | Full 20-Section Deep Dive (~2,500 words) |
| **[02. Capacity Modes, Streams & OCI NoSQL](02-capacity-modes-streams-and-oci-nosql.md)** | Provisioned (RCU/WCU Math) vs. On-Demand, Change Data Capture (DynamoDB Streams), OCI NoSQL Table Engine & Units (RU/WU), Global Tables | Full 20-Section Deep Dive (~2,300 words) |
| **[03. Hot Partitions & Query Anti-Patterns](03-hot-partitions-and-anti-patterns.md)** | Partition Throughput Limits (1,000 WCU / 3,000 RCU), Key Salting & Write Sharding, Scan vs. Query Traps, Adaptive Capacity | Abbreviated NoSQL Guide (~900 words) |

---

## 🎯 Senior & Staff Interview Core Competencies

Upon completing this module, you will be prepared to defend:
1. **The Single-Table Design Paradigm**: Why hyperscale NoSQL systems reject multi-table joins in favor of pre-computed indexing and Generic Primary Key overloading (`PK`/`SK`), delivering single-digit millisecond latency regardless of dataset size.
2. **RCU and WCU Capacity Mathematics**: How to precisely calculate read and write capacity requirements across eventual consistency, strong consistency, and transactional operations to eliminate over-provisioning spend.
3. **Hot Partition Avoidance & Key Salting**: How to diagnose single-partition I/O bottlenecks and implement cryptographic salt prefixes to distribute write-heavy ingestion across dozens of storage partitions.
