# System Design 02: Asynchronous Event-Driven Order Processing Platform

---

## 1. Requirements & Constraints (R)

### 1.1 Business Context & Problem Statement
A global retail enterprise requires an asynchronous, highly resilient order processing and fulfillment engine. The platform must absorb extreme burst traffic during flash sale events (e.g., Black Friday, holiday promotions) without dropping transactions, while orchestrating a multi-step saga: payment authorization, inventory allocation, fraud scoring, fiscal invoicing, and warehouse dispatch. Because synchronous processing introduces brittle cross-service dependencies and thread starvation, the enterprise mandates an asynchronous, event-driven decoupled architecture.

### 1.2 Functional Requirements
1. **Asynchronous Order Ingestion**: Ingest order submissions, return an immediate acknowledgment with a tracking `order_id`, and buffer messages for background execution.
2. **Distributed Saga Orchestration**: Coordinate order execution across multiple independent microservices (Payment, Inventory, Fraud, Fulfillment) using compensating transactions on failure.
3. **Idempotency & Exactly-Once Semantic Enforcement**: Ensure that network retransmissions, duplicate webhook callbacks, or message redeliveries do not cause double billing or duplicate inventory deductions.
4. **Order State Lifecycle Querying**: Provide real-time status visibility (`PENDING`, `AUTHORIZED`, `INVENTORY_RESERVED`, `FAILED`, `DISPATCHED`) to client applications.

### 1.3 Non-Functional Requirements & Quantitative SLAs
- **Throughput & Burst Absorbing**:
  - Baseline Order Volume: **10,000 orders/second**.
  - Peak Flash Sale Volume: **100,000 orders/second (10x burst)**.
  - Daily Transactional Volume: $\approx 864\text{ million orders/day}$.
- **Latency SLAs**:
  - Ingestion ACK Latency: $P95 < 20\text{ms}$, $P99 < 50\text{ms}$ (Client receives `HTTP 202 Accepted` with `order_id`).
  - End-to-End Processing Completion (Order Ingested $\to$ Fulfillment Dispatched): $P95 < 2.0\text{s}$, $P99 < 5.0\text{s}$.
- **Resilience & Durability Targets**:
  - **Recovery Point Objective (RPO)**: $\mathbf{0}$ (Zero loss of acknowledged orders; messages replicated across 3 AZs/ADs upon ingestion).
  - **Recovery Time Objective (RTO)**: $\mathbf{< 30\text{ seconds}}$ for worker fleet auto-recovery.
- **Data Retention**:
  - Event streaming log retention: 7 days.
  - Cold archival to Object Storage: 7 years for financial compliance.

---

## 2. High-Level Architecture (A)

### 2.1 Dual-Cloud Architectural Topology

```
========================================================================================================================
                          EVENT-DRIVEN ORDER PROCESSING ARCHITECTURE
========================================================================================================================

                                  [ Web / Mobile / Partner Clients ]
                                                   │
                                                   ▼
                         [ Edge Anycast DNS + CDN: Route 53 / OCI DNS Steering ]
                                                   │
                                                   ▼
                  [ API Gateway Layer: AWS HTTP API Gateway / OCI API Gateway ]
                     - JWT Authentication & Schema Validation
                     - Direct Service Proxy Buffer (Bypasses Compute)
                                                   │
                         ┌─────────────────────────┴─────────────────────────┐
                         │                                                   │
                         ▼ (AWS Ingress)                                     ▼ (OCI Ingress)
          [ Amazon SQS FIFO / Amazon MSK ]                     [ OCI Queue / OCI Streaming ]
          - Message Deduplication ID                           - Partition Key: customer_id
          - Partition Key: customer_id                         - Multi-Fault-Domain Persistence
                         │                                                   │
                         ▼                                                   ▼
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  SAGA ORCHESTRATION & WORKER FLEET (VPC / VCN Private Subnets across 3 AZs / 3 ADs)
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   AWS ORCHESTRATION: Step Functions / EventBridge     OCI ORCHESTRATION: OCI Events / OKE Worker Mesh
   ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
   │                                   DISTRIBUTED SAGA EXECUTION STEPS                                               │
   │                                                                                                                  │
   │  [ 1. Fraud Check ] ──► [ 2. Payment Charge ] ──► [ 3. Reserve Inventory ] ──► [ 4. Dispatch Fulfillment ]      │
   │           │                        │                           │                                                 │
   │           ▼ (Fail)                 ▼ (Fail: Refund)            ▼ (Fail: Release Inventory)                       │
   │       [ Reject Order ]         [ Mark Cancelled ]          [ Compensate Payment ]                                │
   └──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
   PERSISTENCE & STATE LEDGER (Isolated Private Data Tier)
   ┌───────────────────────────────────────────────┬──────────────────────────────────────────────────────────────────┐
   │ AWS: Amazon DynamoDB Global Tables            │ OCI: OCI NoSQL Database                                          │
   │  - Partition Key: customer_id                 │  - Sharded on customer_id                                        │
   │  - Sort Key: order_id                         │  - ACID Transactions, Sub-10ms P99                              │
   │  - DynamoDB Streams --> Kinesis Data Streams  │  - Table Streams --> OCI Streaming / OCI Data Flow               │
   └───────────────────────────────────────────────┴──────────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
   DEAD-LETTER QUEUE & RESILIENCE TIER
   [ AWS SQS DLQ + EventBridge Redrive / OCI Queue DLQ + Dead Letter Topic Alerts ]
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
```

### 2.2 Dual-Cloud Component Mapping

| Architectural Function | AWS Cloud Native | OCI Cloud Native | Design Justification |
| :--- | :--- | :--- | :--- |
| **Ingress API Gateway** | Amazon HTTP API Gateway (Direct SQS Integration) `[Doc: API Gateway, checked 2026]` | OCI API Gateway with Direct OCI Streaming / OCI Queue Backend `[Doc: OCI API GW, checked 2026]` | Provides sub-15ms client ACK by buffering payloads straight into messaging brokers without compute execution cold starts. |
| **Ingress Messaging Buffer** | Amazon SQS FIFO / Amazon MSK (Managed Kafka) | OCI Queue / OCI Streaming (Kafka API Compatible) | Buffers 100,000 orders/sec burst traffic, decoupling ingress from downstream processing capacity. |
| **Saga Orchestrator** | AWS Step Functions (Distributed Express Workflows) | OCI Events + OKE Event-Driven Microservice Saga Mesh | Coordinates multi-step state machine with visual branching, retries, and automated compensation logic. |
| **Worker Processing Tier** | AWS Lambda + Amazon EKS Workers (KEDA Autoscaling) | OCI Functions + OCI Container Engine for Kubernetes (OKE) | Serverless execution for short tasks; containerized workers for complex batch and inventory calculations. |
| **State Ledger Database** | Amazon DynamoDB (Global Tables, On-Demand Mode) | OCI NoSQL Database (Table storage with ACID support) | Low-latency key-value state store with automatic partition scaling and sub-10ms P99 reads/writes. |
| **Dead-Letter Handling** | SQS Dead-Letter Queue (DLQ) + Lambda Redrive | OCI Queue Dead-Letter Queue + OCI Notification Service | Captures poison pills and exhausted retries, preventing head-of-line blocking while enabling automated replay. |
| **Audit & Event Archive** | Amazon S3 Standard + S3 Glacier Flexible Retrieval | OCI Object Storage + Archive Storage Tier | Stores raw immutable event payloads for 7-year regulatory compliance at lowest cost. |

---

## 3. Traffic Flow & Ingress Path (T)

### 3.1 End-to-End Ingress Sequence

```text
1. Client POST /api/v1/orders
2. CloudFront / OCI CDN terminates TLS 1.3 at Edge
3. WAF validates client IP token bucket rate limit (Max 500 req/min per IP)
4. HTTP API Gateway executes JWT validation against OAuth2 Identity Provider
5. API Gateway maps request body directly into Message Broker (No Lambda hop):
   - AWS: API Gateway AWS Service Integration --> sqs:SendMessage
   - OCI: OCI API Gateway HTTP Backend --> OCI Streaming PutMessages API
6. Message Broker persists message across 3 AZs/ADs and returns MessageId
7. API Gateway returns HTTP 202 Accepted: {"order_id": "...", "status": "PENDING"}
8. Client connection closes in < 18ms total latency
```

### 3.2 Direct Broker Integration (Eliminating Compute Cold Starts)
A classic anti-pattern is placing a Lambda or container between the API Gateway and the Queue merely to call `SendMessage`. Under a 100,000 QPS flash sale:
- 100,000 concurrent Lambda executions exhaust account concurrency quotas (default 1,000 per region).
- Compute execution adds $15\text{--}40\text{ms}$ of unnecessary latency and cost.
- **Solution**: Configure API Gateway **Service Integration** to authenticate via IAM Role / OCI Dynamic Group and pass the raw JSON payload straight into SQS or OCI Streaming.

---

## 4. Data Flow & Storage Engine (D)

### 4.1 The Distributed Saga Pattern & Compensating Transactions

```text
[OrderSubmitted] ──► (State: PENDING)
        │
        ▼
   [Step 1: Fraud Evaluation]
        ├── Pass ──► (State: FRAUD_CLEARED)
        └── Fail ──► (State: REJECTED_FRAUD) ──► Publish OrderFailed Event
        │
        ▼
   [Step 2: Payment Authorization]
        ├── Success ──► (State: PAYMENT_AUTHORIZED)
        └── Decline ──► (State: PAYMENT_FAILED) ──► Publish OrderFailed Event
        │
        ▼
   [Step 3: Inventory Reservation]
        ├── Allocated ──► (State: INVENTORY_RESERVED)
        └── Out of Stock ──► [COMPENSATING TRANSACTION]:
                                1. Issue Payment Refund to Payment Gateway
                                2. Update Order State to CANCELLED_OUT_OF_STOCK
                                3. Notify Customer
        │
        ▼
   [Step 4: Warehouse Dispatch] ──► (State: DISPATCHED) ──► Publish OrderCompleted Event
```

### 4.2 State Ledger Schema (DynamoDB / OCI NoSQL)

```json
{
  "PK": "CUSTOMER#c18f3a92-4b21-4d9e",
  "SK": "ORDER#o882149b-712a-431e",
  "order_id": "o882149b-712a-431e",
  "customer_id": "c18f3a92-4b21-4d9e",
  "idempotency_key": "idem-9921-bc7412e0",
  "status": "INVENTORY_RESERVED",
  "total_amount_cents": 14999,
  "currency": "USD",
  "items": [
    {"sku": "SKU-992", "quantity": 2, "unit_price": 4999},
    {"sku": "SKU-104", "quantity": 1, "unit_price": 5001}
  ],
  "saga_history": [
    {"step": "FRAUD_CHECK", "status": "PASSED", "timestamp": "2026-09-07T12:00:01.102Z"},
    {"step": "PAYMENT_AUTH", "status": "APPROVED", "timestamp": "2026-09-07T12:00:01.520Z"},
    {"step": "INVENTORY_RESERVE", "status": "ALLOCATED", "timestamp": "2026-09-07T12:00:02.015Z"}
  ],
  "created_at": 1788782401,
  "ttl": 1820318401
}
```

### 4.3 Idempotency Enforcement Algorithm
To guarantee exactly-once business semantics despite at-least-once message delivery:
1. When a consumer worker receives an event, it executes a conditional write to the Idempotency table:
   - **AWS DynamoDB**:
     ```python
     dynamodb.put_item(
         TableName="OrderProcessingIdempotency",
         Item={
             "IdempotencyKey": {"S": idempotency_key},
             "Status": {"S": "IN_PROGRESS"},
             "ExpiresAt": {"N": str(int(time.time()) + 3600)}
         },
         ConditionExpression="attribute_not_exists(IdempotencyKey)"
     )
     ```
   - **OCI NoSQL**:
     ```sql
     INSERT INTO OrderProcessingIdempotency (idempotency_key, status, expires_at)
     VALUES (:idem_key, 'IN_PROGRESS', :exp_time)
     IF NOT EXISTS;
     ```
2. If the conditional write fails (`ConditionalCheckFailedException`):
   - The message is a duplicate in-flight or already completed execution.
   - The consumer drops the message and returns `ACK` to the broker, preventing duplicate processing.

---

## 5. Security Architecture (S)

### 5.1 Credential-less Service-to-Service Authorization
- **AWS Configuration**:
  - API Gateway assumes an IAM Service Role with strict `sqs:SendMessage` permissions scoped exclusively to the ARN of the target FIFO queue.
  - Kubernetes Worker Pods leverage **EKS Pod Identity**; pods mount temporary STS tokens to query DynamoDB and decrypt KMS data keys.
- **OCI Configuration**:
  - OCI API Gateway uses an **Instance Principal / Resource Principal** to publish records directly to OCI Streaming.
  - OKE Worker Pods utilize **OCI Workload Identity**; dynamic groups map the pod service account to policies permitting `use streams` and `manage nosql-tables` within the production compartment.

### 5.2 Envelope Encryption Architecture (KMS / OCI Vault)
- Every order contains Personally Identifiable Information (PII) and payment tokens.
- Envelope encryption is enforced at every tier:
  1. Messages in SQS / OCI Queue are encrypted at rest using an AWS KMS Customer Managed Key (CMK) / OCI Vault HSM key.
  2. DynamoDB / OCI NoSQL tables enforce CMK encryption at rest.
  3. Sensitive attributes (customer billing address, payment token) undergo field-level client-side encryption before serialization, ensuring DB administrators cannot view raw PII.

---

## 6. Reliability & High Availability (R)

### 6.1 Blast Radius Containment & Partitioning
- **Message Sharding**:
  - SQS FIFO Message Group ID / OCI Streaming Partition Key is set to `customer_id`.
  - Orders from the same customer are processed in strict chronological sequence, while orders across distinct customers are distributed across hundreds of parallel partitions across 3 AZs/ADs.
- **Poison Pill Isolation**:
  - If an order payload contains a corrupted schema or triggers an unhandled memory exception in the worker runtime, standard retries would crash the worker repeatedly (head-of-line blocking).
  - **Dead-Letter Queue (DLQ) Policy**:
    - `maxReceiveCount = 3`: If a message fails processing 3 times, the broker automatically transfers it to the DLQ.
    - SRE alert fires; a diagnostic Lambda inspects the DLQ, logs the failure signature to CloudWatch/OCI Logging, and alerts the engineering on-call channel.

### 6.2 Circuit Breaking & Backpressure
- When downstream third-party systems (e.g., legacy payment gateway or credit card processor) experience degradation:
  - Worker pods utilize an **Envoy Circuit Breaker** or **Resilience4j** instance.
  - If payment gateway error rate exceeds 30%, the circuit opens.
  - Workers stop calling the payment gateway and re-queue messages with an exponential visibility timeout (`visibilityTimeout = 30 * (2 ** retry_count) + jitter`).
  - SQS/OCI Queue buffers the traffic; the upstream API continues accepting orders without degradation.

---

## 7. Scaling & Capacity Planning (S)

### 7.1 Event-Driven Autoscaling with KEDA

```text
[ SQS / OCI Streaming Queue ]
       │
       ▼ (Exposes Queue Backlog Metric: ApproximateNumberOfMessagesVisible)
[ KEDA Autoscaler (Kubernetes Event-driven Autoscaling) ]
       │
       ▼ (Calculates: Target 50 messages per active worker pod)
[ ScaledObject Controller ]
       │
       ▼ (Scales Pod Replica Fleet)
[ Order Consumer Pod Fleet (10 pods ────► 500 pods in < 90 seconds) ]
```

- **KEDA ScaledObject Manifest (Production AWS/OCI)**:
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-consumer-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-processing-worker
  minReplicaCount: 10
  maxReplicaCount: 600
  cooldownPeriod: 300
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/111122223333/order-ingest-fifo.fifo
      queueLength: "50"
      awsRegion: us-east-1
      identityOwner: pod
```

---

## 8. Observability & Production Telemetry (O)

### 8.1 Golden Signals for Event-Driven Architectures
1. **Consumer Lag (Queue Age)**:
   - Metric: `ApproximateAgeOfOldestMessage` (AWS SQS) / `MessageLag` (OCI Streaming).
   - Target: $< 1,000\text{ms}$ during normal operations; alert if $> 15,000\text{ms}$ for $> 3\text{ minutes}$.
2. **End-to-End Processing Duration**:
   - Time elapsed from `order_created` timestamp to `order_completed` event publication.
   - P99 Target: $< 5.0\text{s}$.
3. **Dead-Letter Ingestion Rate**:
   - Absolute count of messages redirected to DLQ.
   - Target: $0$ messages. Alert if $> 5$ messages in 5 minutes.
4. **Broker Ingress vs. Worker Processing Rate**:
   - Comparison ratio: `IngressRate / ProcessingRate`. If ratio $> 1.5$ for 10 minutes, workers are falling behind.

### 8.2 Asynchronous Distributed Tracing
- In HTTP APIs, tracing headers are passed in HTTP headers. In event-driven systems, the tracer injects W3C TraceContext into the **message metadata attributes**:
```json
{
  "order_id": "o882149b-712a-431e",
  "MessageAttributes": {
    "traceparent": {
      "DataType": "String",
      "StringValue": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
    }
  }
}
```
- Each downstream microservice extracts this traceparent, ensuring AWS X-Ray and OCI APM reconstruct the entire cross-service asynchronous saga graph.

---

## 9. Cloud Cost & Unit Economics (C)

### 9.1 Monthly FinOps BOM (100 Million Orders/Month)

```text
====================================================================================================
                      FINOPS COST ESTIMATE: 100M ORDERS/MONTH
====================================================================================================

COMPONENT                        AWS NATIVE COST                  OCI NATIVE COST
----------------------------------------------------------------------------------------------------
Ingress API Gateway              $100 (HTTP API Gateway)          $70 (OCI API Gateway)
Queue / Event Broker             $400 (SQS FIFO 100M requests)    $250 (OCI Queue / Streaming)
Saga Orchestration               $1,200 (Step Functions Express)  $450 (OKE Event Mesh)
Worker Compute Fleet             $3,200 (EKS Graviton Nodes)      $2,100 (OKE Ampere A1 Compute)
State Ledger DB (DynamoDB/NoSQL) $1,850 (On-demand R/W units)     $1,100 (OCI NoSQL provisioned)
Object Storage (Raw Archive)     $230 (S3 Glacier Instant)        $140 (OCI Infrequent Access)
Telemetry (CloudWatch / OCI Log) $650 (Logs + Traces)             $320 (OCI APM + Logging)
----------------------------------------------------------------------------------------------------
TOTAL MONTHLY COST               $7,630                           $4,430
COST PER 1,000 PROCESSED ORDERS  ~$0.076 per 1,000 orders         ~$0.044 per 1,000 orders
====================================================================================================
```

---

## 10. Failure Modes & Cascades (F)

### 10.1 Scenario A: Downstream Payment Gateway Brownout
- **Failure Condition**: The third-party credit card gateway response latency spikes from $200\text{ms}$ to $25\text{ seconds}$ before throwing `HTTP 500`.
- **Mitigation Flow**:
  1. Circuit breaker trips after 20 consecutive failures.
  2. Workers stop sending synchronous calls to the gateway.
  3. Saga state for affected orders shifts to `PAYMENT_PENDING_RETRY`.
  4. Messages are pushed to an exponential delay retry queue with jitter.
  5. API Gateway continues accepting new orders without dropping customer requests.

### 10.2 Scenario B: Poison Pill Message Crashes Worker Pod
- **Failure Condition**: A corrupted JSON payload with an unexpected recursive schema triggers an out-of-memory (OOM) panic in the worker container runtime.
- **Mitigation Flow**:
  1. The container crashes and kubelet restarts the pod.
  2. The message becomes visible again in SQS / OCI Queue.
  3. Worker re-reads the message and crashes a second and third time (`ReceiveCount = 3`).
  4. Message broker detects `ReceiveCount >= maxReceiveCount` and moves the message to the **Dead-Letter Queue (DLQ)**.
  5. The poison pill is isolated; the worker pod resumes processing healthy orders from the queue.

---

## 11. Trade-offs & Defense (T)

### 11.1 Key Architectural Compromises

```text
====================================================================================================
                                  ARCHITECTURAL TRADE-OFF MATRIX
====================================================================================================

DESIGN CHOICE                   CHOSEN OPTION              REJECTED ALTERNATIVE       TECHNICAL JUSTIFICATION
----------------------------------------------------------------------------------------------------
Asynchronous vs Synchronous     Event-Driven Queue Buffer  Synchronous REST Chaining  Synchronous chaining causes
                                                                                      cascading failures and thread
                                                                                      starvation during flash sales.
----------------------------------------------------------------------------------------------------
Saga Coordination Pattern       Orchestration              Choreography               Orchestration provides a single
                                (Step Functions / State)   (Pure Event Pub/Sub)       authoritative state machine,
                                                                                      eliminating circular event storms.
----------------------------------------------------------------------------------------------------
Data Persistence Model          DynamoDB / OCI NoSQL       RDBMS (Aurora / Base DB)   NoSQL eliminates connection pool
                                                                                      exhaustion when 500 worker pods
                                                                                      scale out concurrently.
====================================================================================================
```

### 11.2 Bar-Raiser Defense Script

> **Interviewer**: *"Why use an asynchronous architecture with a Saga pattern instead of a standard synchronous REST API with two-phase commit (2PC) to guarantee immediate financial consistency?"*

**Candidate Defense**:
*"Two-Phase Commit (2PC) is an anti-pattern in high-throughput cloud environments. 2PC requires distributed locking across all participating databases (Payment, Inventory, Order, Warehouse) for the entire duration of the transaction. If any single service, network link, or database node experiences latency or downtime, all locked resources remain frozen, causing immediate connection starvation and cascading brownouts across the entire platform.*

*By adopting an asynchronous event-driven architecture with the Saga pattern, we achieve eventual consistency and non-blocking scalability. Each microservice commits its local ACID transaction immediately and publishes an event. If a downstream step like inventory reservation fails, the orchestrator triggers automated compensating transactions (such as issuing a payment refund). This guarantees that our ingress tier can absorb 100,000 orders/sec without bottlenecking on distributed locks."*
