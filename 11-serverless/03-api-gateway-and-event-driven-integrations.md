# 03. API Gateway & Event-Driven Serverless Integrations

## 1. Problem
When building serverless architectures, engineers frequently attempt to wire all application workflows through synchronous HTTP request-response APIs. When an API Gateway invokes a serverless function that performs heavy image processing or orchestrates three external third-party APIs, the client browser blocks waiting for a response. If any downstream dependency experiences latency, the invocation breaches API Gateway's 29-second integration timeout, dropping the connection and leaving backend operations in half-finished, unrecoverable states. Scalable serverless systems decouple user ingress from backend processing using asynchronous, event-driven integrations.

## 2. Cloud Concept: Synchronous vs. Asynchronous Ingress
```text
SYNCHRONOUS INGRESS (High Coupling, Fragile):
[Client] ──► [API Gateway] ──► [Lambda / OCI Function] ──► [Slow External API (Times out at 29s!)]
   ▲                                                             │
   └────────────────────── HTTP 504 Gateway Timeout ─────────────┘

ASYNCHRONOUS EVENT-DRIVEN INGRESS (Decoupled, Resilient):
[Client] ──► [API Gateway] ──► Returns HTTP 202 Accepted (< 50ms)
                    │
                    ▼
           [Event Broker: SQS / OCI Queue] ◄──► [Dead-Letter Queue (DLQ)]
                    │
                    ▼
           [Worker Function (Scales independently, retries automatically)]
```

- **Synchronous Ingress (HTTP Request-Response)**:
  - Invocation type: `RequestResponse`.
  - Client waits for the function execution to complete and return a payload.
  - Hard API Gateway timeout ceiling: **29 seconds** (AWS) / **30 seconds** (OCI).
- **Asynchronous Event-Driven Ingress**:
  - Invocation type: `Event`.
  - The caller drops the payload into an intermediary event queue or broker (Amazon SQS, Amazon EventBridge, OCI Queue, OCI Streaming) and immediately receives a confirmation.
  - Built-in automated retries (exponential backoff) and Dead-Letter Queue (DLQ) routing for poisoned payloads.

## 3. AWS Integration Patterns
In AWS:
- **API Gateway: REST APIs vs. HTTP APIs**:
  - *HTTP APIs*: Modern, lightweight, low-latency API gateway. Designed specifically for serverless proxies. Up to **70% cheaper** (\$1.00 per million requests vs. \$3.50 for REST APIs) and delivers sub-10ms latency.
  - *REST APIs*: Feature-rich legacy gateway supporting API keys, client request validation, request transformation (VTL templates), and WAF integration.
- **Dead-Letter Queues (DLQs) & Asynchronous Retries**:
  - When invoked asynchronously, AWS Lambda automatically retries failed executions **twice** with backoff delays.
  - If all retries fail, Lambda routes the failed event payload to a configured **Dead-Letter Queue (Amazon SQS or SNS)** or **EventBridge On-Failure Destination**, preserving failed event records for debugging.

## 4. OCI Integration Patterns
In OCI:
- **OCI API Gateway**:
  - Fully managed, high-performance API proxy deployed directly into your private VCN.
  - Supports rate-limiting, OAuth2/JWT token validation, CORS enforcement, and request transformation.
  - Routes requests directly to **OCI Functions**, HTTP backend servers, or Oracle Autonomous Database endpoints.
- **OCI Events Service & OCI Streaming**:
  - Built on open standards (CloudEvents 1.0).
  - OCI services automatically publish state change events (e.g., `object.create` in Object Storage).
  - The **OCI Events Service** evaluates event rules and triggers OCI Functions, OCI Notifications, or streams payloads into OCI Streaming (Kafka-compatible) for real-time processing.

## 5. Production Failure Modes: Poison Pills & Retry Storms
- **The Poison Pill Infinite Retry Loop**: An asynchronous Lambda function consumes messages from an SQS queue. A malformed message arrives that causes the application code to throw an uncaught `NullPointerException`. SQS returns the message to the queue after the visibility timeout. Lambda retries it immediately, crashes, and retries again forever, burning compute budget and blocking processing of all subsequent messages.
  - *Mitigation*: Configure an SQS **Redrive Policy** with `maxReceiveCount = 3` pointing to a Dead-Letter Queue (DLQ).

## 6. Troubleshooting & Diagnostics
1. **Inspect CloudWatch Lambda Error Metrics**:
   - `Errors`: Counts unhandled code exceptions.
   - `DeadLetterErrors`: Counts failures when Lambda cannot write failed payloads to the configured DLQ (typically due to missing IAM permissions).
2. **Inspect SQS Dead-Letter Queue**:
   ```bash
   aws sqs receive-message --queue-url <dlq-url> --attribute-names All
   ```
   Inspect the `MessageAttributes` to identify the stack trace and original error message.

## 7. Senior Interview Question & Defense
**Question**: *You are designing an e-commerce order processing pipeline where checkout takes 45 seconds due to legacy ERP integration. How do you design the API Gateway and Serverless architecture to guarantee zero lost orders, $< 100\text{ms}$ client response latency, and automatic failure recovery?*

**Staff-Level Defense**:
> "I implement an **Asynchronous Decoupled Event-Driven Pipeline using API Gateway, FIFO SQS Queues, and Dead-Letter Queues**:
>
> 1. **Ingress Tier (Sub-100ms Client Response)**:
>    - The client sends an HTTP `POST /orders` request to **AWS API Gateway (HTTP API)**.
>    - Instead of invoking Lambda synchronously, API Gateway uses **Native Service Integration** to write the order payload directly into an **Amazon SQS FIFO Queue**.
>    - API Gateway returns an immediate **HTTP 202 Accepted** with an `order_id` to the client in **under 40 milliseconds**, completely insulating the client from the 45-second ERP processing latency and bypassing API Gateway's 29-second timeout.
>
> 2. **Processing Tier (Elastic Serverless Worker)**:
>    - A dedicated worker Lambda consumes from the SQS FIFO queue with `batch_size = 1`.
>    - SQS triggers the worker. The worker interacts with the legacy ERP system across the private VPC network.
>    - SQS visibility timeout is configured to **6x the function timeout** (e.g., 300 seconds) to prevent duplicate processing while the ERP transaction executes.
>
> 3. **Failure Recovery & Idempotency**:
>    - The worker implements **Idempotency Keys** (using DynamoDB conditional writes) to guarantee that retried messages do not create duplicate ERP orders.
>    - If the legacy ERP is temporarily offline, Lambda fails and SQS retries the message using exponential backoff.
>    - If the message fails 3 consecutive times (`maxReceiveCount = 3`), SQS automatically isolates the poison pill payload in a **Dead-Letter Queue (DLQ)** and triggers a PagerDuty alert to the on-call engineer, guaranteeing **zero lost orders**."
