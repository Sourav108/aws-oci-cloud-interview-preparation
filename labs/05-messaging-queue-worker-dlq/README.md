# Lab 05: Asynchronous Queue & Dead-Letter Processing

---

## 1. Goal & Core Concept

The objective of this hands-on lab is to deploy, configure, and validate an asynchronous message-driven worker pipeline using **Amazon Simple Queue Service (SQS)** and **OCI Queue Service**, complete with visibility timeout tuning, poison pill message detection, and Dead-Letter Queue (DLQ) redrive automation.

### Core Architectural Concepts Tested
- **Visibility Timeout Mechanics**: Tuning consumer processing leases to prevent duplicate execution while handling worker crashes.
- **Dead-Letter Queue (DLQ) Redrive**: Quarantining malformed/poison messages after $N$ retry attempts (`maxReceiveCount = 3`).
- **At-Least-Once Delivery & Idempotency**: Protecting downstream systems from duplicate message deliveries.
- **Backpressure & Queue-Depth Telemetry**: Monitoring message backlog and age metrics.

---

## 2. Cost Warning & Resource Estimate

> [!NOTE]
> **ESTIMATED HOURLY RUN COST: `< $0.05 / hour`**
>
> | Cloud Provider | Resource Component | Quantity | Hourly Rate | Run Cost (2 Hours) |
> | :--- | :--- | :---: | :---: | :---: |
> | **AWS** | Amazon SQS (10,000 test messages) `[Doc: SQS, checked 2026]` | 2 Queues | Free Tier ($0.40/1M req) | $0.004 |
> | **AWS** | AWS Lambda (Worker execution) `[Doc: Lambda, checked 2026]` | 1 Function | Free Tier (1M free req/mo) | $0.000 |
> | **OCI** | OCI Queue (10,000 test messages) `[Doc: OCI Queue, checked 2026]` | 1 Queue | Free Tier ($0.20/1M req) | $0.002 |
>
> *Total Estimated Lab Cost for a 2-hour session: **$0.006 USD**.*
> *In No-Live-Account Fallback Mode, cost is **$0.00**.*

---

## 3. Architecture Diagram

```
========================================================================================================================
                          QUEUE & DEAD-LETTER PROCESSING TOPOLOGY
========================================================================================================================

  [ Producer Service ]
          │
          ▼ (1. SendMessage: Raw JSON Order Payload)
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  PRIMARY QUEUE (AWS SQS / OCI Queue)
  - Visibility Timeout: 30 seconds
  - Delivery Count Threshold (maxReceiveCount): 3
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
          │
          ├── (2. Poll & Process: ReceiveMessage)
          ▼
  [ Consumer Worker (Lambda / Container) ]
          │
          ├── Success: DeleteMessage ──► [ Process Complete ]
          │
          └── Unhandled Exception / Poison Pill: Fails 3 Times
                    │
                    ▼ (3. Broker Automatically Moves Message)
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  DEAD-LETTER QUEUE (DLQ)
  - Message Retention: 14 Days
  - Triggers CloudWatch / OCI Alarm
  - DLQ Redrive Automation (Replays messages after code bug fix)
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
```

---

## 4. Prerequisites

1. Terraform CLI v1.8+.
2. Cloud credentials allowing SQS/OCI Queue management.

---

## 5. Infrastructure Code (Terraform HCL)

### 5.1 AWS SQS & DLQ Implementation (`aws_queue.tf`)

```hcl
# AWS Reference Implementation: SQS Primary Queue with DLQ
resource "aws_sqs_queue" "order_dlq" {
  name                      = "lab05-order-dlq"
  message_retention_seconds = 1209600 # 14 Days
}

resource "aws_sqs_queue" "order_queue" {
  name                       = "lab05-order-primary-queue"
  visibility_timeout_seconds = 30
  message_retention_seconds  = 345600 # 4 Days

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.order_dlq.arn
    maxReceiveCount     = 3
  })
}
```

### 5.2 OCI Queue Implementation (`oci_queue.tf`)

```hcl
# OCI Reference Implementation: OCI Queue with Dead-Letter Handling
resource "oci_queue_queue" "order_queue" {
  compartment_id     = var.compartment_ocid
  display_name       = "lab05-order-queue"
  visibility_in_seconds = 30
  timeout_in_seconds    = 30
  dead_letter_queue_delivery_count = 3
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

Sending Healthy Message:
$ aws sqs send-message --queue-url ${QUEUE_URL} --message-body '{"order_id": "101", "amount": 50}'
{
    "MD5OfMessageBody": "3b2e5352fae498c11e5491176b334581",
    "MessageId": "6c367733-6cf0-4357-a36c-9400ef3da277"
}
```

---

## 8. Failure Injection Drill: Poison Pill Message Injection

### The Scenario
Send a malformed payload (e.g., recursive JSON or negative amount causing division by zero) that crashes the worker.

### The Injection
```bash
aws sqs send-message --queue-url ${QUEUE_URL} --message-body '{"order_id": "BAD_DATA", "corrupted": true}'
```

### Manifested Symptoms
- Worker process crashes 3 times.
- Message disappears from `lab05-order-primary-queue`.
- CloudWatch metric `ApproximateNumberOfMessagesVisible` on DLQ transitions from 0 to 1.

---

## 9. Debugging Walkthrough

```text
====================================================================================================
                        DLQ MESSAGE INSPECTION & REDRIVE
====================================================================================================

$ aws sqs receive-message --queue-url ${DLQ_URL} --attribute-names All
Output:
{
    "Messages": [
        {
            "Body": "{"order_id": "BAD_DATA", "corrupted": true}",
            "Attributes": {
                "ApproximateReceiveCount": "3",
                "SentTimestamp": "1725721200000"
            }
        }
    ]
}

Fix: Deploy patch to worker code handling invalid schema gracefully.
Redrive:
$ aws sqs start-message-move-task --source-arn ${DLQ_ARN} --destination-arn ${PRIMARY_ARN}
Result: Message re-queued and processed successfully.
====================================================================================================
```

---

## 10. Teardown & Verification

```bash
terraform destroy -auto-approve
aws sqs list-queues --queue-name-prefix lab05
```

---

## 11. Senior Interview Defense & Takeaways

> **Interviewer**: *"How do you handle head-of-line blocking in an SQS FIFO queue when a single poison pill message fails repeatedly?"*

**Candidate Defense**:
*"In standard SQS FIFO queues, if a message within a Message Group ID fails processing, the broker halts delivery of all subsequent messages in that same group to preserve strict chronological ordering. If a poison pill enters the queue, the entire customer queue blocks indefinitely!*

*To prevent head-of-line blocking while maintaining FIFO integrity, we configure a Dead-Letter Queue with `maxReceiveCount = 3`. Once the poison message fails 3 times, SQS moves it to the DLQ, unblocking subsequent messages in that group. Furthermore, we partition the `MessageGroupId` by customer or tenant ID so a single poisoned customer queue never blocks other customers."*
