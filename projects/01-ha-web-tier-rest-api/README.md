# Reference Project 01: High-Availability Web Tier REST API

---

## 1. Executive Summary & Architecture Overview

This production reference architecture implements an enterprise-grade, highly available, low-latency REST API platform capable of serving **50,000 baseline queries per second (QPS)** with $P99 < 35\text{ms}$ response latency and **99.999% availability ("five nines")**.

The platform provides:
- **Dual-Cloud Bilingual Implementation**: Production blueprints for Amazon Web Services (AWS) and Oracle Cloud Infrastructure (OCI).
- **Three-Tier Network Segmentation**: Strict isolation across Public Ingress, Private Compute, and Isolated Data subnets across 3 Availability Zones (AWS) / 3 Fault Domains (OCI).
- **Containerized Microservices on Kubernetes**: Managed orchestration via Amazon EKS and OCI Kubernetes Engine (OKE) with native CNI IP targeting.
- **Polyglot Persistence & Caching Tier**: Amazon Aurora PostgreSQL (I/O-Optimized) / OCI Autonomous Transaction Processing (ATP) backed by distributed ElastiCache / OCI Cache Redis clusters.
- **Managed Connection Pooling**: AWS RDS Proxy and high-availability PgBouncer daemonsets preventing database socket exhaustion during horizontal scale-outs.

---

## 2. Dual-Cloud System Topology

```
========================================================================================================================
                                     DUAL-CLOUD REST API TOPOLOGY
========================================================================================================================

                                       [ Global Web / Mobile Clients ]
                                                      │
                                                      ▼
                              [ Global Anycast DNS: Route 53 / OCI DNS Steering ]
                                                      │
                                                      ▼
                                [ Edge CDN & WAF: CloudFront / OCI CDN + WAF ]
                                                      │ (TLS 1.3 / OWASP Top 10 Filtering)
                                                      ▼
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  VPC / VCN PERIMETER (10.100.0.0/16)
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   PUBLIC INGRESS SUBNETS (3 AZs / 3 ADs or 3 Fault Domains)
   [ AWS Application Load Balancer (ALB) / OCI Flexible Load Balancer (100-8000 Mbps) ]
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
                                                      │ (Direct Pod IP Forwarding via VPC CNI)
                                                      ▼
   PRIVATE COMPUTE SUBNETS (Non-Routable Application Tier)
   ┌───────────────────────────────────┬───────────────────────────────────┬───────────────────────────────────┐
   │ AZ-1 / AD-1 (Fault Domain 1)      │ AZ-2 / AD-2 (Fault Domain 2)      │ AZ-3 / AD-3 (Fault Domain 3)      │
   │ [ EKS Pod / OKE Container Pod ]   │ [ EKS Pod / OKE Container Pod ]   │ [ EKS Pod / OKE Container Pod ]   │
   │  - REST API Daemon (Go / Rust)    │  - REST API Daemon (Go / Rust)    │  - REST API Daemon (Go / Rust)    │
   │  - OpenTelemetry Daemonset Agent  │  - OpenTelemetry Daemonset Agent  │  - OpenTelemetry Daemonset Agent  │
   └─────────────────┬─────────────────┴─────────────────┬─────────────────┴─────────────────┬─────────────────┘
                     │                                   │                                   │
                     ▼                                   ▼                                   ▼
   PRIVATE ISOLATED CACHE & CONNECTION POOL TIER
   [ AWS ElastiCache Valkey/Redis Cluster (Multi-AZ) / OCI Cache with Redis ]
   [ AWS RDS Proxy / PgBouncer Pod Pool (Multiplexes 7,500 client connections down to 150 DB sockets) ]
                                                      │
                                                      ▼
   ISOLATED DATA SUBNETS (Zero Internet Access - DB Port 5432 / 1521 Only)
   ┌───────────────────────────────────┬───────────────────────────────────┬───────────────────────────────────┐
   │ AZ-1 / AD-1 (Primary Writer)      │ AZ-2 / AD-2 (Storage Quorum)      │ AZ-3 / AD-3 (Read Replica)        │
   │ [ Aurora PostgreSQL Primary /     │ [ Aurora 6-Way Quorum /           │ [ Aurora Read Replica /           │
   │   OCI Autonomous Transaction DB ] │   OCI Autonomous Local Standby ]  │   OCI Base DB Read Replica ]      │
   └───────────────────────────────────┴───────────────────────────────────┴───────────────────────────────────┘
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
```

---

## 3. Dual-Cloud Component Mapping Matrix

| Architectural Function | AWS Cloud Implementation | OCI Cloud Implementation | Implementation Notes |
| :--- | :--- | :--- | :--- |
| **Global Edge & DNS** | Amazon Route 53 (Latency & Geoproximity) `[Doc: Route 53, checked 2026]` | OCI Traffic Management Steering Policies `[Doc: OCI DNS, checked 2026]` | Anycast resolution returns optimal edge IP in $< 15\text{ms}$. |
| **Edge CDN & WAF** | Amazon CloudFront + AWS WAF | OCI CDN + OCI Web Application Firewall (WAF) | Blocks volumetric DDoS and enforces token bucket IP rate limits. |
| **Ingress Load Balancing** | Application Load Balancer (ALB) | OCI Flexible Load Balancer (Public VIP) | Terminates TLS 1.3, executes path routing, probes pod health. |
| **Container Orchestration** | Amazon EKS (Managed Node Groups) | OCI Kubernetes Engine (OKE) with Virtual Nodes | Managed Kubernetes with native VPC/VCN CNI direct pod routing. |
| **In-Memory Caching Tier**| Amazon ElastiCache for Redis (Cluster Mode) | OCI Cache with Redis (3-node HA cluster) | Caches read queries and stores 24-hour idempotency tokens. |
| **Database Connection Pool**| AWS RDS Proxy | PgBouncer Deployment on OKE | Prevents connection pool starvation on horizontal pod scale-outs. |
| **Relational Storage** | Amazon Aurora PostgreSQL (I/O-Optimized) | OCI Autonomous Transaction Processing (ATP) | ACID compliance, synchronous multi-AZ persistence, sub-30s failover. |
| **Workload Identity** | EKS Pod Identity / AWS IRSA | OCI Workload Identity / Dynamic Groups | Credential-less token exchange; zero hardcoded cloud keys. |

---

## 4. Production Infrastructure as Code (Terraform HCL)

### 4.1 AWS Terraform Module (`aws_ha_rest_api.tf`)

```hcl
# AWS Reference Implementation: Ingress Load Balancer and Target Group
resource "aws_lb" "api_alb" {
  name               = "prod-ha-rest-api-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = [aws_subnet.public_az1.id, aws_subnet.public_az2.id, aws_subnet.public_az3.id]

  enable_deletion_protection = true
  drop_invalid_header_fields = true

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

resource "aws_lb_target_group" "api_tg" {
  name        = "prod-ha-rest-api-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip" # Direct CNI routing to EKS Pod IPs

  health_check {
    enabled             = true
    path                = "/healthz/ready"
    protocol            = "HTTP"
    port                = "8080"
    interval            = 10
    timeout             = 3
    healthy_threshold   = 2
    unhealthy_threshold = 2
    matcher             = "200"
  }
}

resource "aws_rds_cluster" "aurora_primary" {
  cluster_identifier      = "prod-aurora-pg-cluster"
  engine                  = "aurora-postgresql"
  engine_version          = "16.1"
  database_name           = "tier1api"
  master_username         = "dbadmin"
  manage_master_user_password = true
  storage_type            = "aurora-iopt1" # Aurora I/O-Optimized

  availability_zones      = ["us-east-1a", "us-east-1b", "us-east-1c"]
  db_subnet_group_name    = aws_db_subnet_group.isolated_db.name
  vpc_security_group_ids  = [aws_security_group.db_sg.id]
  backup_retention_period = 35
  preferred_backup_window = "02:00-03:00"
  deletion_protection     = true

  serverlessv2_scaling_configuration {
    max_capacity = 64.0
    min_capacity = 4.0
  }
}
```

### 4.2 OCI Terraform Module (`oci_ha_rest_api.tf`)

```hcl
# OCI Reference Implementation: Flexible Load Balancer and Autonomous Database
resource "oci_load_balancer_load_balancer" "api_lb" {
  compartment_id = var.compartment_ocid
  display_name   = "prod-ha-rest-api-lb"
  shape          = "flexible"
  is_private     = false
  subnet_ids     = [oci_core_subnet.public_ingress_subnet.id]

  shape_details {
    minimum_bandwidth_in_mbps = 100
    maximum_bandwidth_in_mbps = 8000
  }

  network_security_group_ids = [oci_core_network_security_group.lb_nsg.id]
}

resource "oci_load_balancer_backend_set" "api_backend_set" {
  name             = "prod-ha-api-backend-set"
  load_balancer_id = oci_load_balancer_load_balancer.api_lb.id
  policy           = "ROUND_ROBIN"

  health_checker {
    protocol          = "HTTP"
    port              = 8080
    url_path          = "/healthz/ready"
    interval_ms       = 10000
    timeout_in_millis = 3000
    retries           = 2
    return_code       = 200
  }
}

resource "oci_database_autonomous_database" "api_atp" {
  compartment_id           = var.compartment_ocid
  db_name                  = "tier1atp"
  display_name             = "prod-autonomous-transaction-db"
  db_workload              = "OLTP"
  is_dedicated             = false
  db_version               = "19c"
  compute_model            = "ECPU"
  compute_count            = 16
  data_storage_size_in_tbs = 5
  is_auto_scaling_enabled  = true
  is_free_tier             = false

  subnet_id                = oci_core_subnet.isolated_db_subnet.id
  nsg_ids                  = [oci_core_network_security_group.db_nsg.id]
  admin_password           = var.db_admin_password
}
```

---

## 5. Security, Workload Identity & Encryption

### 5.1 Credential-less Workload Identity
- **AWS Configuration**: Applications authenticate via **EKS Pod Identity**. Kubernetes pods associate with an IAM Service Account bound to an IAM Role granting access exclusively to KMS decrypt and Secrets Manager read operations.
- **OCI Configuration**: Pods utilize **OCI Workload Identity**. The OKE cluster issues Service Account tokens mapped to an IAM Dynamic Group via tenancy matching rules. Zero passwords or API keys are placed on filesystem volumes.

### 5.2 Microsegmentation Security Rules

```text
[ Internet Client ]
       │ TCP 443
       ▼
[ Ingress Load Balancer Security Group / NSG ]
  - Ingress: TCP 443 from CDN IP prefix lists.
  - Egress: TCP 8080 to Application Pod Security Group.
       │ TCP 8080
       ▼
[ Application Pod Security Group / NSG ]
  - Ingress: TCP 8080 from Load Balancer Security Group.
  - Egress: TCP 6379 to Redis Cache SG, TCP 5432/1521 to DB Proxy SG.
       │ TCP 5432 / 1521
       ▼
[ Database Security Group / NSG ]
  - Ingress: TCP 5432/1521 strictly from Compute/Proxy Security Group.
  - Egress: ZERO public internet outbound.
```

---

## 6. Observability, SLIs/SLOs & Alerting Runbook

### 6.1 The 4 Golden Signals Instrumentation
1. **Latency**: CloudWatch Metric `TargetResponseTime` / OCI Monitoring `Latency`. Alert if P95 $> 25\text{ms}$ for 3 minutes.
2. **Traffic**: Request count per target. Alert if QPS drops $> 50\%$ abruptly (indicates upstream edge failure).
3. **Errors**: HTTP 5xx error rate. Alert on multi-burn-rate condition (14.4x burn rate over 1 hour consumes 2% error budget).
4. **Saturation**: Connection pool utilization on RDS Proxy / PgBouncer. Alert if active connections $> 80\%$ of limit.

---

## 7. Deployment, Validation & Verification Runbook

### 7.1 Dry-Run Linting & Validation
```bash
# 1. Initialize without cloud backend
terraform init -backend=false

# 2. Enforce code formatting and syntax checks
terraform fmt -check
terraform validate

# 3. Security posture analysis
checkov -d . --framework terraform
```

### 7.2 Smoke Test Validation Scripts
```bash
# Verify End-to-End API Health
curl -s -o /dev/null -w "%{http_code} %{time_total}s
"   -H "Host: api.enterprise.com"   https://${ALB_DNS_NAME}/healthz/ready

# Expected Output: 200 0.012s

# Verify Idempotent Resource Creation
curl -X POST https://${ALB_DNS_NAME}/api/v1/resources   -H "Host: api.enterprise.com"   -H "Content-Type: application/json"   -H "Idempotency-Key: test-idem-001"   -d '{"resource_name": "production-cluster-alpha"}'

# Expected Output: HTTP 201 Created on first invocation; HTTP 200/409 on re-transmission.
```

---

## 8. FinOps Cost Breakdown & Sizing Economics

```text
====================================================================================================
                        FINOPS COST BREAKDOWN: DEV vs STAGING vs PROD
====================================================================================================

ENVIRONMENT       AWS MONTHLY COST        OCI MONTHLY COST        SIZING & TOPOLOGY
----------------------------------------------------------------------------------------------------
Development       $450 / month            $290 / month            Single-AZ/AD, 2 Pods, Serverless DB
Staging           $2,800 / month          $1,850 / month          2-AZ, 10 Pods, Aurora Small 2-Node
Production        $16,880 / month         $10,350 / month         3-AZ, 150 Pods, Aurora Multi-AZ 3-Node
----------------------------------------------------------------------------------------------------
UNIT COST (PROD)  ~$0.130 per 1M requests ~$0.079 per 1M requests Scaled at 50,000 baseline QPS
====================================================================================================
```

---

## 9. Failure Mode Drills & Chaos Engineering Runbook

### 9.1 Game Day Drill: Single Availability Zone Drop
1. **Action**: Terminate all compute worker nodes in `us-east-1a` (AWS) / `Fault-Domain-1` (OCI).
2. **Expected System Behavior**:
   - ALB/OCI LB detects target unresponsiveness in $< 3\text{ seconds}$; stops routing to AZ-1 pods.
   - Kubernetes scheduler re-provisions pods in AZ-2 and AZ-3.
   - Aurora storage layer continues operating without failover (4-of-6 quorum maintained).
   - Client impact: P99 latency briefly elevates to $65\text{ms}$ during pod rescheduling; zero 5xx errors returned.
