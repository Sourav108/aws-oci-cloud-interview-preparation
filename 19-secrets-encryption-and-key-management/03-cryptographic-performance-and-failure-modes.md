# 03. Cryptographic Performance, KMS Throttling & Failure Modes

## 1. Problem
Key Management Services (AWS KMS and OCI Vault) enforce strict regional API rate limits to protect underlying Hardware Security Module (HSM) infrastructure from denial-of-service saturation. While standard steady-state traffic rarely breaches these limits, sudden scale-out events (e.g., an Auto Scaling Group launching 300 EC2 instances or 2,000 Lambda containers initializing simultaneously) cause thousands of simultaneous `kms:Decrypt` and `kms:GenerateDataKey` calls. The account exhausts its regional KMS quota, throwing HTTP 400 `KMS.ThrottlingException`. Instances fail to decrypt their EBS volumes, Lambdas crash on cold start, and the auto-scaling event collapses into a catastrophic production brownout.

## 2. Cloud Concept: KMS Throttling Dynamics
```text
AUTO-SCALING KMS THROTTLING CASCADE:
Traffic Surge ──► 1,000 Lambda containers boot simultaneously
                        │
                        ▼ 1,000 simultaneous calls: kms:Decrypt(DB_PASSWORD)
┌────────────────────────────────────────────────────────────────────────┐
│ AWS KMS / OCI VAULT REGIONAL API LIMIT (e.g., 5,500 req/s quota)       │
│ * Influx exceeds burst bucket rate limit!                              │
│ * KMS rejects remaining calls with HTTP 400 KMS.ThrottlingException!   │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ Throttling Exception Returned!
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│ COMPUTE WORKER FAILURE:                                                │
│ * Lambda containers fail to initialize database connection pools!      │
│ * Health checks fail ──► Platform returns HTTP 500 to customers!       │
└────────────────────────────────────────────────────────────────────────┘
```

- **KMS Regional Quotas**:
  - In AWS, cryptographic operations share a regional bucket with baseline limits between **5,500 and 50,000 requests per second** depending on the region `[Doc: AWS KMS Request Quotas, checked 2026-09-04]`.
  - Burst limits allow momentary spikes, but prolonged sustained bursts deplete the token bucket, resulting in hard throttling.

## 3. Client-Side Data Key Caching (The AWS Encryption SDK)
To eliminate KMS API bottlenecks, enterprise systems implement **Client-Side Data Key Caching** using the **AWS Encryption SDK** or local in-memory cryptographic caches:

```text
CLIENT-SIDE DATA KEY CACHING PATTERN:
1. Application needs to encrypt customer record #1.
2. Checks in-memory Cryptographic Materials Cache (CMC): MISS.
3. Calls KMS: GenerateDataKey ──► Receives Plaintext DEK + Encrypted DEK.
4. Stores DEK in memory cache with constraints:
   - Max Age: 300 seconds (5 minutes)
   - Max Messages Encrypted: 10,000 records
   - Max Bytes Encrypted: 1 GB
5. Records #2 through #10,000 use the cached DEK in local RAM!
   * ZERO calls to KMS! ZERO API latency! ZERO throttling risk!
6. When limits expire, DEK is securely purged and refreshed from KMS.
```

## 4. Key Policy Lockouts & Break-Glass Recovery
- **The "Nobody Owns the Key" Disaster**:
  - If a KMS Key Policy is updated to remove permissions from `arn:aws:iam::<account-id>:root` and only lists a specific IAM user who is subsequently deleted, **the key becomes completely orphaned**.
  - *Mitigation*: Key policies must **always** include a root administrative statement delegating key management authority to the AWS account root principal:
    ```json
    {
      "Sid": "EnableIAMUserPermissions",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::123456789012:root" },
      "Action": "kms:*",
      "Resource": "*"
    }
    ```

## 5. Production Failure Modes: Secret Desynchronization & Key Expiration
- **The Mid-Deployment Secret Desynchronization**: A developer manually updates the database password in AWS Secrets Manager without updating the running PostgreSQL database instance. The next auto-scaling event provisions new EC2 instances that pull the new password and fail authentication, creating a split-brain fleet where half the instances are connected and half are failing.
- **The Accidental Key Disabling Outage**: An administrator disables a KMS key to "temporarily test security." Every encrypted EBS volume, RDS database, and S3 bucket using that key immediately halts I/O operations, crashing production servers instantly.

## 6. Troubleshooting & Diagnostics
1. **Detect KMS Throttling in CloudWatch**:
   - Query the `AWS/KMS` namespace for the metric: `UserErrorCount` and dimension `Operation` = `ThrottlingException`.
2. **Decode KMS Access Denied**:
   ```bash
   aws sts decode-authorization-message --encoded-message <error-token>
   ```

## 7. Senior Interview Question & Defense
**Question**: *During a major flash sale, your serverless architecture scales from 50 to 3,000 concurrent Lambda functions in 60 seconds. Suddenly, hundreds of Lambda functions crash on startup with `KMS.ThrottlingException` while decrypting database credentials. How do you resolve this immediate production crisis, and what permanent architectural change do you implement?*

**Staff-Level Defense**:
> "This outage is caused by a **KMS API Quota Exhaustion Cascade** triggered by un-cached cold start decryption calls:
>
> 1. **Immediate Production Triage**:
>    - I immediately submit an emergency **AWS Service Quota Increase** request for KMS Cryptographic Operations in the region to elevate the limit from 5,500 to 20,000+ req/s.
>    - Concurrently, I configure **Provisioned Concurrency** on the Lambda functions (e.g., keep 500 execution environments permanently warm). This eliminates cold start initialization spikes and stops the flood of new decryption calls to KMS.
>
> 2. **The Permanent Architectural Fix**:
>    - **Implement Client-Side Data Key Caching via the AWS Encryption SDK**:
>      Instead of calling KMS to decrypt the database credential on every cold start or invocation, we use the **AWS Parameters and Secrets Lambda Extension**:
>      - The extension runs as an in-memory sidecar process inside the Lambda microVM.
>      - It queries Secrets Manager / KMS once, and caches the decrypted secret in local microVM memory with a configurable TTL (e.g., 300 seconds).
>      - Subsequent invocations read from local memory in $< 1\text{ms}$ with **zero KMS API calls**.
>    - **Result**: Even if the Lambda fleet scales to 10,000 concurrent instances, KMS API requests drop by 99.9%, completely insulating the platform from KMS throttling limits."
