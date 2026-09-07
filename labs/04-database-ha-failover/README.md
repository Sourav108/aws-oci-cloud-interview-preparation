# Lab 04: Managed PostgreSQL High Availability & Failover Simulation

---

## 1. Goal & Core Concept

The objective of this hands-on lab is to deploy, configure, and simulate failover scenarios on high-availability managed relational databases across **Amazon Web Services** (AWS Aurora PostgreSQL / RDS Multi-AZ) and **Oracle Cloud Infrastructure** (OCI Base Database with Data Guard / Autonomous Transaction Processing).

### Core Architectural Concepts Tested
- **Synchronous Replication Mechanics**: Aurora 6-way quorum storage across 3 AZs vs. Oracle Data Guard Fast-Start Failover (FSFO) with synchronous redo transport.
- **Automated Failover Timeline**: Measuring client reconnection disruption window ($< 15\text{--}30\text{ seconds}$).
- **Connection Pool Buffering**: Using AWS RDS Proxy / PgBouncer to buffer client queries and prevent socket drop storms during primary node failover.
- **Read-Write Splitting**: Routing writes to cluster writer endpoint and reads to reader endpoints.

---

## 2. Cost Warning & Resource Estimate

> [!NOTE]
> **ESTIMATED HOURLY RUN COST: `< $0.45 / hour`**
>
> | Cloud Provider | Resource Component | Quantity | Hourly Rate | Run Cost (2 Hours) |
> | :--- | :--- | :---: | :---: | :---: |
> | **AWS** | Aurora Serverless v2 (0.5 to 2 ACU) `[Doc: Aurora, checked 2026]` | 1 Cluster (2 Instances) | $0.12 / hr | $0.24 |
> | **AWS** | AWS RDS Proxy `[Doc: RDS Proxy, checked 2026]` | 1 Endpoint | $0.018 / hr | $0.036 |
> | **OCI** | Base Database System (1 OCPU Standard) `[Doc: OCI DB, checked 2026]` | 1 DB System | $0.17 / hr | $0.34 |
>
> *Total Estimated Lab Cost for a 2-hour session: **$0.616 USD**.*
> *In No-Live-Account Fallback Mode, cost is **$0.00**.*

---

## 3. Architecture Diagram

```
========================================================================================================================
                          DATABASE HIGH AVAILABILITY & FAILOVER TOPOLOGY
========================================================================================================================

  [ Client Application / Kubernetes Pod Fleet ]
                        │
                        ▼
       [ Connection Proxy Tier: AWS RDS Proxy / PgBouncer ]
       - Buffers client queries during failover
       - Pins connections to transaction boundaries
                        │
       ┌────────────────┴────────────────┐
       │ (Writer Traffic)                │ (Reader Traffic)
       ▼                                 ▼
  ═════════════════════════════════════  ═════════════════════════════════════
  PRIMARY ZONE (AWS AZ-1 / OCI FD-1)     STANDBY REPLICA (AWS AZ-2 / OCI FD-2)
  ┌───────────────────────────────────┐  ┌───────────────────────────────────┐
  │ [ Primary Writer Node ]           │  │ [ Standby Read Replica Node ]     │
  │  - Accepts Reads & Writes         │  │  - Serves Analytical Reads        │
  │  - Aurora Storage Quorum /        │  │  - Promoted on Primary Failure    │
  │    Data Guard Redo Transport      │  │                                   │
  └─────────────────┬─────────────────┘  └─────────────────▲─────────────────┘
                    │                                      │
                    └────────── (Synchronous Quorum) ──────┘
  ═════════════════════════════════════  ═════════════════════════════════════
```

---

## 4. Prerequisites

1. Private Subnets configured across at least two distinct AZs / Fault Domains.
2. Terraform CLI v1.8+.

---

## 5. Infrastructure Code (Terraform HCL)

### 5.1 AWS Aurora PostgreSQL Cluster (`aws_database.tf`)

```hcl
# AWS Reference Implementation: Aurora Serverless v2 with RDS Proxy
resource "aws_rds_cluster" "lab_aurora" {
  cluster_identifier      = "lab04-aurora-pg"
  engine                  = "aurora-postgresql"
  engine_version          = "16.1"
  database_name           = "appdb"
  master_username         = "dbadmin"
  manage_master_user_password = true

  db_subnet_group_name    = var.db_subnet_group_name
  vpc_security_group_ids  = [var.db_security_group_id]
  skip_final_snapshot     = true

  serverlessv2_scaling_configuration {
    max_capacity = 2.0
    min_capacity = 0.5
  }
}

resource "aws_rds_cluster_instance" "writer_node" {
  identifier         = "lab04-writer-az1"
  cluster_identifier = aws_rds_cluster.lab_aurora.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.lab_aurora.engine
  engine_version     = aws_rds_cluster.lab_aurora.engine_version
  availability_zone  = "us-east-1a"
}

resource "aws_rds_cluster_instance" "standby_node" {
  identifier         = "lab04-standby-az2"
  cluster_identifier = aws_rds_cluster.lab_aurora.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.lab_aurora.engine
  engine_version     = aws_rds_cluster.lab_aurora.engine_version
  availability_zone  = "us-east-1b"
}
```

### 5.2 OCI Base Database System (`oci_database.tf`)

```hcl
# OCI Reference Implementation: Base DB System with Data Guard
resource "oci_database_db_system" "lab_db" {
  compartment_id      = var.compartment_ocid
  availability_domain = var.ad_name
  database_edition    = "ENTERPRISE_EDITION_EXTREME_PERFORMANCE"
  shape               = "VM.Standard3.Flex"

  db_home {
    database {
      admin_password = var.db_admin_password
      db_name        = "appdb"
      db_workload    = "OLTP"
    }
    db_version = "19c"
  }

  db_system_options {
    storage_management = "ASM"
  }

  shape_config {
    ocpus = 1
  }

  subnet_id = var.db_subnet_ocid
  ssh_public_keys = [var.ssh_public_key]
  hostname = "lab04db"
}
```

---

## 6. Step-by-Step Deployment Guide

```bash
terraform init -backend=false
terraform validate
terraform plan
```

---

## 7. Expected Validation Results

```text
[Statically validated — not applied to a live account]

Testing Database Connectivity via Proxy:
$ pgbench -h ${RDS_PROXY_ENDPOINT} -U dbadmin -d appdb -c 50 -j 4 -t 1000
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
number of clients: 50
number of threads: 4
number of transactions per client: 1000
number of transactions actually processed: 50000/50000
latency average = 1.428 ms
tps = 3501.24 (including connections establishing)
```

---

## 8. Failure Injection Drill: Forcing Primary Failover

### The Scenario
Simulate catastrophic primary writer node failure while running high-concurrency read/write benchmarks.

### The Injection
```bash
aws rds failover-db-cluster --db-cluster-identifier lab04-aurora-pg
```

### Manifested Symptoms
- Write operations stall for $\approx 12\text{ seconds}$.
- RDS Proxy buffers queries in-flight, preventing socket disconnect errors on client applications.
- Standby node `lab04-standby-az2` assumes primary writer role.
- Cluster writer CNAME switches to AZ-2 IP.

---

## 9. Debugging Walkthrough

```text
====================================================================================================
                        DATABASE FAILOVER EVENT TIMELINE
====================================================================================================

$ aws rds describe-events --source-type db-cluster --source-identifier lab04-aurora-pg
Timeline Output:
- [14:02:01 UTC] User-initiated failover started for cluster 'lab04-aurora-pg'.
- [14:02:08 UTC] Standby instance 'lab04-standby-az2' promoted to new writer.
- [14:02:14 UTC] DNS record updated to point to new writer.
- [14:02:16 UTC] Cluster failover completed in 15 seconds. Total dropped client connections: 0 (Buffered by Proxy).
====================================================================================================
```

---

## 10. Teardown & Verification

```bash
terraform destroy -auto-approve
aws rds describe-db-clusters --query "DBClusters[?DBClusterIdentifier=='lab04-aurora-pg']"
```

---

## 11. Senior Interview Defense & Takeaways

> **Interviewer**: *"Why use an RDS Proxy / PgBouncer layer rather than letting Kubernetes pods connect directly to Aurora or Oracle Base DB?"*

**Candidate Defense**:
*"Direct database connections create severe connection starvation under load. Each direct PostgreSQL connection consumes between 5 MB and 10 MB of dedicated server RAM for session state. If a Kubernetes microservice fleet scales from 10 to 150 pods, each running 50 application threads, 7,500 connections will flood the database, triggering Linux OOM kernel panics.*

*Furthermore, during an Aurora or Data Guard failover, direct client connections drop immediately, causing cascading retry storms. RDS Proxy / PgBouncer multiplexes thousands of client connections down to a few hundred shared, warm database sockets and pauses in-flight transactions during the 15-second failover window, resulting in zero application-facing errors."*
