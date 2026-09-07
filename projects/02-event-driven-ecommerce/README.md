# Reference Project 02: Asynchronous Event-Driven Order Processing Platform

---

## 1. Executive Summary & Architecture Overview

This production reference architecture provides a decoupled, asynchronous, highly scalable order processing and fulfillment engine designed to absorb sudden flash sale traffic surges of up to **100,000 orders/second** without dropping transactions or exhausting backend databases.

Key Architectural Capabilities:
- **Direct-to-Broker Ingestion**: Amazon API Gateway and OCI API Gateway map client order payloads directly into streaming queues (SQS FIFO / OCI Streaming) via IAM service proxies, eliminating compute cold starts and account concurrency limits.
- **Distributed Saga Orchestration**: Coordinates multi-service transactions (Fraud -> Payment -> Inventory -> Fulfillment) with automated compensating transactions on failure.
- **Idempotency by Design**: Guarantees exactly-once business execution using atomic conditional writes in Amazon DynamoDB and OCI NoSQL Database.
- **Queue-Lag Driven Autoscaling**: Employs Kubernetes Event-driven Autoscaling (KEDA) to scale consumer worker pods dynamically based on queue depth metrics.

---

## 2. Dual-Cloud System Topology

```
========================================================================================================================
                                 EVENT-DRIVEN E-COMMERCE TOPOLOGY
========================================================================================================================

                                  [ Web / Mobile / POS Checkout Clients ]
                                                     │
                                                     ▼
                             [ API Gateway: AWS HTTP API GW / OCI API Gateway ]
                             - JWT OAuth2 Token Validation
                             - Native Service Proxy Ingestion (Bypasses Compute)
                                                     │
                         ┌───────────────────────────┴───────────────────────────┐
                         │                                                       │
                         ▼ (AWS Ingress)                                         ▼ (OCI Ingress)
          [ Amazon SQS FIFO / Amazon MSK ]                         [ OCI Queue / OCI Streaming ]
          - Deduplication ID & Partition Group                     - Partition Key: customer_id
                         │                                                       │
                         ▼                                                       ▼
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  SAGA WORKER FLEET & ORCHESTRATION (Private Non-Routable Subnets across 3 AZs / 3 ADs)
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   AWS ORCHESTRATION: Step Functions Express / EventBridge   OCI ORCHESTRATION: OCI Events / OKE Microservices Mesh
   ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
   │                                     DISTRIBUTED SAGA TRANSACTION STEPS                                           │
   │                                                                                                                  │
   │  [ 1. Fraud Check ] ──► [ 2. Payment Charge ] ──► [ 3. Inventory Reserve ] ──► [ 4. Warehouse Dispatch ]         │
   │           │                        │                           │                                                 │
   │           ▼ (Fail)                 ▼ (Fail: Refund)            ▼ (Fail: Release Inventory)                       │
   │       [ Reject Order ]         [ Mark Cancelled ]          [ Compensate Payment ]                                │
   └──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                     │
                                                     ▼
   PERSISTENCE & STATE LEDGER (Zero Public Internet Access)
   ┌─────────────────────────────────────────────────┬────────────────────────────────────────────────────────────────┐
   │ AWS: Amazon DynamoDB (Global Tables)            │ OCI: OCI NoSQL Database (ACID Table Storage)                   │
   │  - Partition Key: customer_id                   │  - Sharded on customer_id                                      │
   │  - Sort Key: order_id                           │  - Sub-10ms P99 Latency                                        │
   │  - DynamoDB Streams -> Kinesis Archive          │  - Table Streams -> OCI Streaming Archive                      │
   └─────────────────────────────────────────────────┴────────────────────────────────────────────────────────────────┘
                                                     │
                                                     ▼
   DEAD-LETTER QUEUE & RESILIENCE TIER
   [ AWS SQS DLQ + EventBridge Redrive / OCI Queue Dead-Letter Queue + Alarm Topics ]
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
```

---

## 3. Dual-Cloud Component Mapping Matrix

| Architectural Function | AWS Cloud Implementation | OCI Cloud Implementation | Engineering Justification |
| :--- | :--- | :--- | :--- |
| **API Ingress Gateway** | Amazon HTTP API Gateway `[Doc: API Gateway, checked 2026]` | OCI API Gateway `[Doc: OCI API GW, checked 2026]` | Direct service proxy routes order payloads directly into messaging brokers with $< 15\text{ms}$ client ACK. |
| **Messaging Buffer** | Amazon SQS FIFO (Message Grouping) | OCI Queue / OCI Streaming | Buffers flash sale spikes, isolating backend workers from unbounded concurrency exhaustion. |
| **Saga Orchestrator** | AWS Step Functions (Express Workflows) | OCI Events + OKE Event-Driven Microservice Saga | Provides visual state machines, automatic retry policies, and audited compensating execution paths. |
| **Worker Processing Tier** | Amazon EKS with KEDA Queue Scaler | OKE with Native Cluster Autoscaler | Dynamically scales containerized worker pods based on queue message age and backlog depth. |
| **Order Ledger Database** | Amazon DynamoDB (On-Demand Mode) | OCI NoSQL Database (Table Storage) | High-throughput distributed key-value store with automatic horizontal partition scaling. |
| **Dead-Letter Handling** | SQS Dead-Letter Queue (DLQ) + Redrive | OCI Queue Dead-Letter Queue | Quarantines poison pills without blocking subsequent valid orders. |

---

## 4. Production Infrastructure as Code (Terraform HCL)

### 4.1 AWS Terraform Module (`aws_event_order.tf`)

```hcl
# AWS Reference Implementation: SQS FIFO Queue with DLQ and Redrive Policy
resource "aws_sqs_queue" "order_dlq" {
  name                        = "prod-order-ingest-dlq.fifo"
  fifo_queue                  = true
  content_based_deduplication = true
  message_retention_seconds   = 1209600 # 14 Days

  kms_master_key_id = "alias/aws/sqs"
}

resource "aws_sqs_queue" "order_ingest_queue" {
  name                        = "prod-order-ingest-queue.fifo"
  fifo_queue                  = true
  content_based_deduplication = true
  visibility_timeout_seconds  = 60
  message_retention_seconds   = 345600  # 4 Days

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.order_dlq.arn
    maxReceiveCount     = 3
  })

  kms_master_key_id = "alias/aws/sqs"
}

resource "aws_dynamodb_table" "order_ledger" {
  name         = "prod-order-ledger"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "customer_id"
  range_key    = "order_id"

  attribute {
    name = "customer_id"
    type = "S"
  }

  attribute {
    name = "order_id"
    type = "S"
  }

  point_in_time_recovery {
    enabled = true
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = aws_kms_key.dynamo_key.arn
  }
}
```

### 4.2 OCI Terraform Module (`oci_event_order.tf`)

```hcl
# OCI Reference Implementation: OCI Queue and OCI NoSQL Database Table
resource "oci_queue_queue" "order_queue" {
  compartment_id     = var.compartment_ocid
  display_name       = "prod-order-ingest-queue"
  visibility_in_seconds = 60
  timeout_in_seconds    = 30
  dead_letter_queue_delivery_count = 3

  custom_encryption_key_id = oci_kms_key.vault_key.id
}

resource "oci_nosql_table" "order_ledger" {
  compartment_id = var.compartment_ocid
  name           = "prod_order_ledger"

  ddl_statement = "CREATE TABLE IF NOT EXISTS prod_order_ledger (customer_id STRING, order_id STRING, status STRING, total_amount NUMBER, saga_history JSON, created_at TIMESTAMP, PRIMARY KEY (SHARD(customer_id), order_id))"

  table_limits {
    max_read_units     = 10000
    max_write_units    = 5000
    max_storage_in_gbs = 200
  }
}
```

---

## 5. Security, Workload Identity & Encryption

### 5.1 Credential-less Service-to-Service Ingress
- The API Gateway executes a direct integration to SQS / OCI Streaming using an **AWS IAM Execution Role** or **OCI Resource Principal**.
- Backend workers running on EKS / OKE acquire short-lived STS / workload identity tokens to interact with DynamoDB and decrypt KMS data keys.

---

## 6. Observability, SLIs/SLOs & Alerting Runbook

### 6.1 Critical Event Telemetry Metrics
1. **ApproximateAgeOfOldestMessage**: Measures consumer lag. Alert if $> 15\text{ seconds}$.
2. **DeadLetterQueueSize**: Absolute message count redirected to DLQ. Alert if $> 0$.
3. **End-to-End Processing Latency**: Duration from order submission to warehouse dispatch. P99 target: $< 5.0\text{s}$.

---

## 7. Deployment & Verification Runbook

```bash
# Verify End-to-End Ingestion via API Gateway
ORDER_ID=$(uuidgen)
curl -X POST https://${API_GATEWAY_URL}/v1/orders   -H "Content-Type: application/json"   -H "Idempotency-Key: ${ORDER_ID}"   -d '{
    "customer_id": "c18f3a92-4b21-4d9e",
    "order_id": "'"${ORDER_ID}"'",
    "amount_cents": 9999,
    "items": [{"sku": "PROD-101", "qty": 1}]
  }'

# Expected Output: HTTP 202 Accepted {"order_id": "...", "status": "PENDING"}
```

---

## 8. FinOps Cost Breakdown & Sizing Economics

```text
====================================================================================================
                        FINOPS COST BREAKDOWN (100M ORDERS / MONTH)
====================================================================================================

TIER                             AWS MONTHLY COST        OCI MONTHLY COST
----------------------------------------------------------------------------------------------------
Ingress API Gateway              $100                    $70
Streaming & Queue Buffer         $400                    $250
Saga Orchestration               $1,200                  $450
Worker Fleet Compute             $3,200                  $2,100
State Ledger Database            $1,850                  $1,100
----------------------------------------------------------------------------------------------------
TOTAL RUN-RATE (100M ORDERS)     $6,750 / month          $3,970 / month
UNIT COST PER 1,000 ORDERS       ~$0.0675                ~$0.0397
====================================================================================================
```

---

## 9. Failure Mode Drills & Chaos Engineering Runbook

### 9.1 Game Day Drill: Worker Fleet Brownout & Queue Drain
1. **Action**: Scale worker deployment to 0 replicas during sustained 5,000 orders/sec ingress.
2. **Verification**:
   - Ingress API Gateway continues returning `HTTP 202 Accepted` with $< 20\text{ms}$ latency.
   - SQS/OCI Queue buffers 300,000 messages safely across 3 AZs/ADs without data loss.
   - Scale worker fleet to 100 replicas; verify KEDA autoscales to 400 replicas and clears backlog in $< 90\text{ seconds}$.
