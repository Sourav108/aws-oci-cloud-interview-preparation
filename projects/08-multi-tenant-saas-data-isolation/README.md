# Reference Project 08: Multi-Tenant SaaS Data Isolation Architecture

---

## 1. Executive Summary & Architecture Overview

This production reference architecture delivers a hardened, enterprise-grade multi-tenant software-as-a-service (SaaS) data platform implementing a **Hybrid Pool vs. Silo Isolation Model**. The platform provides ironclad tenant boundary enforcement across compute, networking, and relational persistence tiers.

Key Architectural Capabilities:
- **Hybrid Isolation Strategy**:
  - **Tier 1 (Enterprise VIP Customers - Silo Model)**: Dedicated, air-gapped database instances or schema silos with isolated customer-managed KMS encryption keys.
  - **Tier 2 (Standard SMB Customers - Pool Model)**: Shared database instances with database-enforced Row-Level Security (RLS) policies or Pluggable Databases (PDBs).
- **Dynamic Tenant Context Resolution**: API Gateway Lambda/Function authorizers extract tenant identity from signed JWT claims, dynamically vending scoped session credentials and database connection contexts.
- **Cross-Tenant Noisy Neighbor Protection**: Token-bucket rate limiting per tenant combined with priority-tiered worker queues prevents high-volume tenants from degrading shared platform performance.
- **Automated Tenant Lifecycle Provisioning**: Infrastructure-as-code orchestration scripts automate tenant onboarding, dedicated schema creation, and compliance audit trail isolation in $< 60\text{ seconds}$.

---

## 2. Dual-Cloud System Topology

```
========================================================================================================================
                          MULTI-TENANT SAAS DATA ISOLATION ARCHITECTURE
========================================================================================================================

                                [ Global SaaS End Users (Tenants 1..N) ]
                                                   │
                                                   ▼
                       [ Ingress API Gateway: AWS HTTP API GW / OCI API Gateway ]
                       - JWT Claim Extraction: tenant_id, tenant_tier
                       - Tenant-Level Token Bucket Rate Limiting
                                                   │
                                                   ▼
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  APPLICATION & TENANT CONTEXT ROUTER (EKS / OKE Pod Fleet)
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   [ Ingress Context Middleware (Go / Java) ]
    - Evaluates: tenant_tier == 'ENTERPRISE_SILO' vs 'STANDARD_POOLED'
    - Injects: SET LOCAL app.current_tenant = 'tenant-uuid'; into DB session
                                                   │
                       ┌───────────────────────────┴───────────────────────────┐
                       │                                                       │
                       ▼ (Standard Customers)                                  ▼ (Enterprise Customers)
  ───────────────────────────────────────────────────────────────────────────  ─────────────────────────────────────────
   POOLED DATA TIER (Shared RDS / Autonomous DB)                                SILO DATA TIER (Dedicated Instances)
   ┌───────────────────────────────────────────────────────────────────────┐    ┌──────────────────────────────────────┐
   │ AWS Aurora PostgreSQL / OCI Autonomous Database (Shared Cluster)       │    │ Dedicated Aurora / OCI Base DB       │
   │  - PostgreSQL Row-Level Security (RLS) Enforced                       │    │  - Air-gapped single-tenant DB       │
   │  - OCI Pluggable Databases (PDB per Tenant)                           │    │  - Tenant-Managed KMS Key (CMK)      │
   │  - Shared Connection Pool via RDS Proxy / PgBouncer                   │    │  - Zero data co-mingling             │
   └───────────────────────────────────────────────────────────────────────┘    └──────────────────────────────────────┘
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
```

---

## 3. Dual-Cloud Component Mapping Matrix

| Architectural Function | AWS Cloud Implementation | OCI Cloud Implementation | Implementation Notes |
| :--- | :--- | :--- | :--- |
| **Tenant Authentication** | Amazon Cognito User Pools / Custom Lambda Authorizer `[Doc: Cognito, checked 2026]` | OCI IAM Identity Domains + API Gateway JWT Validation `[Doc: OCI IAM, checked 2026]` | Validates tenant JWT claims and enforces per-tenant rate-limiting quotas at edge. |
| **Pooled Relational DB** | Amazon Aurora PostgreSQL with Row-Level Security (RLS) | OCI Autonomous Database / Base DB Pluggable Databases (PDBs) | RLS restricts queries at the DB engine level; PDBs provide physical catalog isolation within a shared container. |
| **Silo Relational DB** | Dedicated Amazon Aurora Cluster per Enterprise Tenant | Dedicated OCI Base DB / Autonomous Database per Enterprise Tenant | Complete physical hardware and storage isolation for high-compliance enterprise accounts. |
| **Tenant Key Management**| AWS KMS Multi-Tenant Key Hierarchy (Key per Tenant) | OCI Vault Dedicated Encryption Keys per Tenant | Allows enterprise tenants to retain control of cryptographic keys, enabling instantaneous crypto-shredding. |
| **Dynamic Workload IAM**| AWS STS Dynamic Session Policies (`AssumeRole`) | OCI Dynamic Groups with Tag-Based Condition Policies | Vends temporary credentials scoped strictly to tenant S3 prefixes (`s3://bucket/{tenant_id}/*`). |

---

## 4. Production Database Isolation (SQL & Terraform)

### 4.1 PostgreSQL Row-Level Security Policy (`rls_schema.sql`)

```sql
-- Enable Row Level Security on Core SaaS Tables
ALTER TABLE customer_orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE customer_orders FORCE ROW LEVEL SECURITY;

-- Create Tenant Context Isolation Policy
CREATE POLICY tenant_isolation_policy ON customer_orders
    FOR ALL
    USING (tenant_id = NULLIF(current_setting('app.current_tenant', true), '')::uuid)
    WITH CHECK (tenant_id = NULLIF(current_setting('app.current_tenant', true), '')::uuid);

-- Application Connection Setup Before Executing Queries:
-- The connection pool checks out a connection and sets session variable:
-- SET LOCAL app.current_tenant = 'd4bb47e9-340b-44e7-b507-f43271747461';
-- Even if an attacker injects "SELECT * FROM customer_orders;", the engine returns ZERO rows for other tenants!
```

### 4.2 AWS IAM Dynamic Tenant Session Policy (S3 Prefix Scoping)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowTenantScopedS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::saas-customer-data-bucket/${aws:PrincipalTag/TenantId}/*"
    }
  ]
}
```

---

## 5. Security, Workload Identity & Encryption

### 5.1 Crypto-Shredding for Instantaneous Tenant Deprovisioning
- When an enterprise customer terminates their contract or exercises GDPR "Right to be Forgotten":
  - Traditional database row deletion across petabytes of backups takes days and risks residual data retention.
  - **Crypto-Shredding Architecture**:
    - Each enterprise tenant possesses a dedicated AWS KMS CMK / OCI Vault key.
    - All data stored in tables and S3/Object Storage is encrypted with this key.
    - Calling `kms:ScheduleKeyDeletion` or `DisableKey` instantly renders all tenant data mathematically unrecoverable across all hot stores, replicas, and historical backups in $< 1\text{ second}$.

---

## 6. Observability, SLIs/SLOs & Alerting Runbook

1. **Cross-Tenant Access Anomaly (Security P0)**: If a query executes without `app.current_tenant` set, trigger an immediate security alert.
2. **Tenant Quota Saturation**: Measures QPS consumption against tenant tier limit. Alert when tenant reaches $90\%$ of provisioned rate limit.
3. **Noisy Neighbor Latency Drift**: Tracks P99 latency variance across tenants sharing pooled database clusters.

---

## 7. Deployment & Verification Runbook

```bash
# Verify Row-Level Security Isolation Under Simulation
psql -h ${DB_HOST} -U appuser -d saasdb -c "
SET LOCAL app.current_tenant = '11111111-1111-1111-1111-111111111111';
SELECT COUNT(*) FROM customer_orders;
-- Returns: 42 (Tenant 1's orders)

SET LOCAL app.current_tenant = '22222222-2222-2222-2222-222222222222';
SELECT COUNT(*) FROM customer_orders;
-- Returns: 18 (Tenant 2's orders; zero visibility into Tenant 1)
"
```

---

## 8. FinOps Cost Breakdown & Sizing Economics

```text
====================================================================================================
                        FINOPS COST BREAKDOWN: MULTI-TENANT SAAS
====================================================================================================

TIER                             POOLED MODEL (1,000 TENANTS)     SILO MODEL (PER VIP TENANT)
----------------------------------------------------------------------------------------------------
Relational Database              $6,200 (Aurora 3-AZ Cluster)     $1,450 (Dedicated Aurora Instance)
Compute Tier                     $3,800 (Shared EKS/OKE Fleet)    $480 (Dedicated Worker Pods)
Object Storage & KMS             $1,200                           $250 (Dedicated KMS Key)
Ingress & API Gateway            $450                             $80
----------------------------------------------------------------------------------------------------
TOTAL MONTHLY RUN-RATE           $11,650 / month                  $2,260 / month per enterprise silo
AVERAGE COST PER POOLED TENANT   ~$11.65 / tenant / month         Premium enterprise pricing model
====================================================================================================
```

---

## 9. Failure Mode Drills & Chaos Engineering Runbook

### 9.1 Game Day Drill: Noisy Neighbor Flash Sale Simulation
1. **Action**: Simulate a single pooled tenant executing 20,000 QPS while other tenants average 100 QPS.
2. **Verification**:
   - API Gateway token bucket rate limiter clamps the rogue tenant to their 1,000 QPS SLA tier, returning `HTTP 429 Too Many Requests`.
   - Other pooled tenants experience zero degradation, maintaining $P99 < 35\text{ms}$.
   - RDS Proxy / PgBouncer ensures the noisy tenant cannot monopolize shared database sockets.
