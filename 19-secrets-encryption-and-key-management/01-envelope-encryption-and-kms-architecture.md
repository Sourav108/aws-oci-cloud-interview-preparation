# 01. Envelope Encryption & KMS Architecture

## 1. Problem
Encrypting sensitive data at rest is a non-negotiable requirement across enterprise compliance frameworks (PCI-DSS, HIPAA, SOC 2, FedRAMP). However, naive encryption implementations suffer from catastrophic performance and architectural flaws: sending large files (e.g., a 10 GB database backup or a 500 MB video) directly over the network to a centralized Key Management Service (KMS) for encryption saturates network bandwidth, creates severe latency bottlenecks, and breaches cloud API payload limits (AWS KMS rejects payloads larger than **4 KB**). Furthermore, storing raw cryptographic keys in application memory or configuration files makes them vulnerable to memory-dump extraction attacks. Scalable cloud security demands **Hardware Security Modules (HSMs) and the Envelope Encryption protocol**.

## 2. Cloud Concept
### The Envelope Encryption Protocol
Envelope encryption is a hierarchical cryptographic pattern that combines the security of centralized hardware key management with the performance of local symmetric encryption:

```text
┌────────────────────────────────────────────────────────────────────────┐
│                   ENVELOPE ENCRYPTION HIERARCHY                        │
│                                                                        │
│   LEVEL 1: KEY ENCRYPTION KEY (KEK / Master Key / CMK)                 │
│   * Resides permanently inside the hardware security boundary (HSM)    │
│   * NEVER leaves the HSM in plaintext!                                │
│   * Managed by AWS KMS / OCI Vault                                     │
│                                │                                       │
│   Generates & Encrypts         │ Calls KMS: GenerateDataKey            │
│                                ▼                                       │
│   LEVEL 2: DATA ENCRYPTION KEY (DEK)                                   │
│   * Plaintext DEK: Used locally in application RAM to encrypt data     │
│     via AES-256-GCM. Wiped from memory immediately after encryption!   │
│   * Encrypted DEK: Stored alongside the ciphertext on disk!            │
│                                │                                       │
│   Encrypts                     │ Local CPU Hardware (AES-NI)           │
│                                ▼                                       │
│   LEVEL 3: THE ACTUAL DATA PAYLOAD (Gigabytes of raw files/records)    │
└────────────────────────────────────────────────────────────────────────┘
```

1. **The Core Protocol Sequence**:
   - **Step 1**: The application calls `KMS:GenerateDataKey` specifying the Master Key (KEK).
   - **Step 2**: KMS generates a high-entropy 256-bit symmetric key inside its FIPS 140-2 Level 3 HSM.
   - **Step 3**: KMS returns two versions of the key to the application:
     1. **Plaintext Data Key**: Used immediately by the local CPU (leveraging Intel AES-NI hardware instructions) to encrypt the multi-gigabyte data payload at memory bus speeds ($> 3\text{ GB/s}$).
     2. **Encrypted Data Key (Ciphertext)**: Encrypted under the Master Key.
   - **Step 4**: The application writes the encrypted data **and** the Encrypted Data Key to disk.
   - **Step 5**: The application **immediately wipes the Plaintext Data Key from RAM memory** (`Arrays.fill(key, 0)`).
2. **Decryption Sequence**:
   - When reading data, the application extracts the Encrypted Data Key from the file header and sends it to `KMS:Decrypt`.
   - KMS decrypts the key inside its HSM and returns the Plaintext Data Key.
   - The application decrypts the payload locally and zeroes the key in memory.

### Master Key Classifications: AWS KMS vs. OCI Vault
- **AWS Key Classifications**:
  1. *AWS Owned Keys*: Free, internal keys managed by AWS. Not visible in your account; no CloudTrail audit logging.
  2. *AWS Managed Keys*: Created automatically by AWS services (e.g., `aws/s3`, `aws/ebs`). Key policy cannot be modified; annual rotation is automatic and non-configurable.
  3. *Customer Managed Keys (CMKs)*: Full customer control. Can define custom Key Policies, IAM grants, enable/disable keys, configure annual auto-rotation, and track all cryptographic calls in CloudTrail.
- **OCI Vault & Key Classifications**:
  - OCI provides centralized key management via **OCI Vault** `[Doc: OCI Vault Key Types, checked 2026-09-04]`:
    1. *Software-Protected Keys*: Cryptographic operations execute in hardened software. Highly cost-effective for dev/test.
    2. *HSM-Protected Keys*: Keys are generated, stored, and operated exclusively inside dedicated **FIPS 140-2 Level 3 Hardware Security Modules**.
    3. *External Key Management (EKM)*: Integrates OCI with third-party on-premises HSMs (Thales CipherTrust) for sovereign cloud compliance.

## 3. Mental Model
Think of envelope encryption as a high-security hotel safe:
- **The Master Key (KEK / KMS)** is the hotel manager's master physical brass key. It is chained to the hotel manager's wrist inside a bank vault and never leaves the room.
- **The Data Encryption Key (DEK)** is an inexpensive, plastic one-time digital keycard.
- **The Data** is your luggage.
- To lock your luggage, the manager presses a button that creates a plastic keycard (Plaintext DEK) and stamps a sealed wax envelope copy of it (Encrypted DEK). You lock your suitcase with the plastic card, throw the plastic card in the fireplace, and glue the wax envelope to the side of your suitcase. Even if a thief steals your suitcase, they cannot open the wax envelope without the hotel manager's master key!

## 4. Architecture Diagram
```text
ENVELOPE ENCRYPTION INGESTION & DECRYPTION FLOW:

ENCRYPTION INGEST PATH:
[Application Process]
       │
       ├──► 1. Calls: kms.GenerateDataKey(KeyId="alias/master-key", KeySpec="AES_256")
       │
       ▼
┌────────────────────────────────────────────────────────────────────────┐
│ CLOUD KMS / OCI VAULT HSM BOUNDARY (FIPS 140-2 Level 3)               │
│ * Master Key (KEK) never leaves this boundary!                         │
│ * Generates 256-bit random symmetric key                              │
│ * Encrypts key with Master Key                                         │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ Returns: (Plaintext DEK, Encrypted DEK)
                                   ▼
[Application Process]
       ├──► 2. Encrypts 10 GB file in RAM using Plaintext DEK via AES-256-GCM
       ├──► 3. Writes (Encrypted File + Encrypted DEK Header) to S3 / Object Storage
       └──► 4. PURGES Plaintext DEK from RAM memory!

DECRYPTION PATH:
[Application Process]
       ├──► 1. Reads file header: Extracts Encrypted DEK
       ├──► 2. Calls: kms.Decrypt(CiphertextBlob = Encrypted DEK)
       │         │
       │         ▼ KMS decrypts DEK inside HSM ──► Returns Plaintext DEK
       ├──► 3. Decrypts file payload in RAM
       └──► 4. PURGES Plaintext DEK from RAM memory!
```

## 5. AWS Implementation
In AWS Key Management Service (KMS):
- **API Request Quotas**:
  - Cryptographic operations (`GenerateDataKey`, `Decrypt`, `Encrypt`) share a regional account-level quota of **5,500 to 50,000 requests per second** depending on the region `[Doc: AWS KMS API Quotas, checked 2026-09-04]`.
  - Exceeding this quota throws HTTP 400 **`KMS.ThrottlingException`**, which can cascade and fail EC2 auto-scaling groups and Lambda invocations!
- **Key Policies vs. IAM Grants**:
  - *Key Policy*: **The primary authorization mechanism in KMS**. If an IAM user has `AdministratorAccess` in IAM, but the KMS Key Policy does not explicitly permit them or the account root, **they cannot use or administer the key!**
  - *KMS Grants*: Programmatic, temporary delegation of permissions. Used internally by AWS services (e.g., when an EC2 instance launches, Auto Scaling creates a temporary Grant for the EBS service to decrypt the root volume).
- **Multi-Region Keys**:
  - Related keys across multiple AWS regions sharing the exact same Key ID and key material.
  - Allows encrypting data in `us-east-1` and decrypting it locally in `eu-west-1` without cross-region network calls to KMS!

## 6. OCI Implementation
In Oracle Cloud Infrastructure Vault:
- **Vault Types & HSM Architecture**:
  - *Default Vault*: Multi-tenant FIPS 140-2 Level 3 HSM partition.
  - *Virtual Private Vault*: Dedicated physical HSM partition allocated exclusively to your organization, providing guaranteed transaction isolation and strict compliance partitioning `[Doc: OCI Dedicated Vault, checked 2026-09-04]`.
- **Master Encryption Keys (MEK)**:
  - Supports AES (128, 192, 256-bit), RSA (2048, 3072, 4096-bit), and ECDSA (Elliptic Curve).
  - Can be rotated manually or configured for **automated scheduled rotation** (e.g., every 90 days).
  - When a MEK is rotated, OCI generates a new key version. Existing data encrypted under older key versions remains readable because Vault retains all historical key versions!
- **Cross-Region Key Replication**:
  - OCI supports native replication of Vaults and Master Encryption Keys across OCI regions.
  - Ensures disaster recovery databases in a secondary region can decrypt backups instantly without inter-region network dependencies.

## 7. Configuration
Comparing Customer Managed Key (CMK) provisioning in Terraform across AWS and OCI:

### AWS KMS Customer Managed Key with Key Policy (Terraform)
```hcl
# AWS KMS Customer Managed Key (CMK) with Annual Rotation
resource "aws_kms_key" "app_encryption_key" {
  description             = "Production Application Master Encryption Key"
  deletion_window_in_days = 30 # Prevents accidental deletion!
  enable_key_rotation     = true # Automatic 365-day rotation!

  # Strict Key Policy
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "EnableRootAdministration"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "AllowApplicationUsage"
        Effect = "Allow"
        Principal = {
          AWS = var.app_role_arn
        }
        Action = [
          "kms:GenerateDataKey",
          "kms:Decrypt",
          "kms:DescribeKey"
        ]
        Resource = "*"
      }
    ]
  })
}

# KMS Alias for abstraction
resource "aws_kms_alias" "app_key_alias" {
  name          = "alias/production-app-key"
  target_key_id = aws_kms_key.app_encryption_key.key_id
}
```

### OCI Vault & Master Encryption Key (Terraform)
```hcl
# 1. OCI KMS Vault (FIPS 140-2 Level 3 HSM)
resource "oci_kms_vault" "prod_vault" {
  compartment_id = var.compartment_id
  display_name   = "production-core-vault"
  vault_type     = "DEFAULT" # HSM-backed!
}

# 2. Master Encryption Key (MEK) with Auto-Rotation
resource "oci_kms_key" "master_key" {
  compartment_id      = var.compartment_id
  display_name        = "app-master-key"
  management_endpoint = oci_kms_vault.prod_vault.management_endpoint

  key_shape {
    algorithm = "AES"
    length    = 32 # 256-bit AES symmetric key
  }

  protection_mode = "HSM" # Hardware Security Module!

  # Automated 90-day key rotation
  auto_key_rotation_details {
    rotation_interval_in_days = 90
  }
}
```

## 8. Data Flow
```text
The Complete Cryptographic Verification Path:
1. Application receives sensitive credit card payload.
2. App dispatches request to KMS: GenerateDataKey(KeyId="alias/app-key").
3. KMS Hardware Security Module (HSM):
   - Authenticates calling IAM identity / Instance Principal.
   - Evaluates Key Policy: Effect = Allow.
   - Hardware True Random Number Generator (TRNG) generates 256-bit DEK.
   - HSM encrypts DEK under Master Key.
4. Response arrives: PlaintextDEK (32 bytes), EncryptedDEK (512 bytes).
5. App initializes AES-256-GCM cipher with PlaintextDEK:
   - Computes Ciphertext and 128-bit Authentication Tag.
6. App stores record in DynamoDB / Autonomous DB:
   {
     "card_id": "c_4921",
     "encrypted_payload": "<binary-ciphertext>",
     "encrypted_dek": "<binary-encrypted-dek>",
     "auth_tag": "<binary-tag>"
   }
7. App overwrites PlaintextDEK byte array with zeros in RAM.
```

## 9. Security
- **Defense Against Ransomware Key Deletion**:
  - Cloud KMS engines prevent immediate key deletion.
  - AWS KMS enforces a mandatory **7 to 30 day waiting period** (`ScheduleKeyDeletion`).
  - OCI Vault enforces a mandatory **7 to 30 day cancellation window**.
  - During this window, security alarms notify administrators, allowing cancelled deletion before data becomes unrecoverable.
- **Cryptographic Erasure (Crypto-Shredding)**:
  - When compliance mandates require permanently deleting customer data (GDPR "Right to be Forgotten"):
  - If millions of records are encrypted with an individual customer's Master Key, **deleting that customer's key instantly renders all customer data mathematically impossible to decrypt**, achieving instant crypto-shredding without expensive storage scrubs.

## 10. Reliability
- **Multi-Region Availability**:
  - Using AWS Multi-Region Keys or OCI Cross-Region Vault Replication ensures that if Region 1 suffers a network partition, Region 2 can decrypt data independently with zero cross-region dependencies.

## 11. Scaling
- **The KMS 4 KB Direct Limit**:
  - The `kms:Encrypt` API has a hard, non-adjustable limit of **4,096 bytes (4 KB)** `[Doc: AWS KMS Encrypt Limits, checked 2026-09-04]`.
  - Attempting to encrypt any file larger than 4 KB directly with KMS throws `ValidationException`. You **must** use Envelope Encryption.

## 12. Observability
- **Auditing Cryptographic Operations**:
  - CloudTrail / OCI Audit logs every individual call to `GenerateDataKey` and `Decrypt`.
  - Audits record the exact IAM identity, source IP, timestamp, and **Encryption Context** (key-value metadata pair that must match during decryption).

## 13. Cost
- **KMS Billing Math**:
  - AWS KMS Customer Managed Key: **\$1.00 per key-month**.
  - Cryptographic requests: **\$0.03 per 10,000 requests** `[Doc: AWS KMS Pricing, checked 2026-09-04]`.
  - An application performing 50,000,000 requests per month directly against KMS incurs **\$150.00/month** in API fees. Using the **AWS Encryption SDK with Data Key Caching** slashes API calls by 99%, reducing the bill to **\$1.50/month**!
- **OCI Vault Pricing**:
  - First 20 Master Encryption Keys per tenancy are **100% Free**; \$0.50/key-month thereafter.

## 14. Failure Modes
- **The Accidental Root Key Policy Lockout**: An administrator edits a KMS Key Policy in Terraform and accidentally removes the statement granting permissions to `arn:aws:iam::<account-id>:root`. Because AWS IAM cannot override a KMS Key Policy, **no one in the entire organization (including the account administrator) can manage the key**! Only AWS Support can intervene.
- **Auto-Scaling KMS Throttling Collapse**: A sudden traffic surge triggers 500 EC2 instances or 2,000 Lambda functions to launch simultaneously. Every instance makes multiple KMS calls to decrypt environment variables or EBS volumes. The account breaches the 5,500 req/s KMS quota; KMS returns HTTP 400 `ThrottlingException`. Compute nodes fail to boot, turning a traffic surge into a total platform outage!

## 15. Troubleshooting
When KMS throws `AccessDeniedException`:
1. **Differentiate Identity vs. Key Policy Failure**:
   - Remember: In AWS KMS, an IAM policy granting `kms:Decrypt` is completely useless if the **KMS Key Policy** does not also delegate authority to the account or identity!
2. **Inspect Encryption Context**:
   - If data was encrypted with an Encryption Context (`{ "tenant_id": "100" }`), the decryption call **must pass the exact same key-value pair**, or KMS returns `InvalidCiphertextException`.

## 16. Common Mistakes
- **Sending Large Payloads to KMS Directly**: Trying to call `kms:Encrypt` on a 50 KB JSON file. Always generate a data key and encrypt locally.
- **Forgetting Automated Key Rotation**: Leaving custom keys on manual rotation for years, violating PCI-DSS and SOC 2 compliance mandates.

## 17. Trade-offs
| Feature | AWS Managed Keys | Customer Managed Keys (CMK) | CloudHSM / Dedicated Vault |
| :--- | :--- | :--- | :--- |
| **Cost** | Free | \$1.00 / key-month | High (\$1.45–\$2.00/hour) |
| **Key Policy Control** | None (AWS Managed) | **Full granular control** | Full physical partition control |
| **Audit Logging** | Limited CloudTrail | Complete CloudTrail logs | Tamper-evident hardware logs |
| **Compliance** | Standard FIPS 140-2 L2/L3 | FIPS 140-2 Level 3 | Strict FIPS 140-2 Level 3 dedicated |

## 18. Interview Questions
1. *What is Envelope Encryption? Why is sending a 10 MB payload directly to AWS KMS or OCI Vault an anti-pattern, and how does the DEK/KEK hierarchy resolve both performance and security constraints?*
2. *Explain the architectural difference between an AWS KMS Key Policy and an IAM Identity Policy. What happens if a full AWS Administrator is excluded from a KMS Key Policy?*
3. *How do you protect high-velocity serverless applications from exhausting regional AWS KMS API quotas during auto-scaling events?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "Envelope Encryption is a foundational cryptographic design pattern that combines the security of centralized hardware key management with the line-rate performance of local symmetric encryption:
>
> 1. **The Architectural Bottleneck of Direct KMS Encryption**:
>    - Sending a 10 MB payload directly over the network to AWS KMS or OCI Vault fails for two critical reasons:
>      1. **Hard API Payload Ceilings**: The AWS `kms:Encrypt` API enforces a **hard physical ceiling of 4,096 bytes (4 KB)**. It physically cannot encrypt a 10 MB file!
>      2. **Network & I/O Saturation**: Transmitting gigabytes of raw data to a centralized cryptographic endpoint introduces severe network latency, burns API request quotas, and exposes raw data to network serialization overhead.
>
> 2. **The Envelope Encryption Hierarchy**:
>    - We decouple the key into a **two-tier hierarchy**:
>      - **Key Encryption Key (KEK / Master Key)**: Lives permanently inside a FIPS 140-2 Level 3 Hardware Security Module (HSM). It never leaves the HSM in plaintext under any circumstances.
>      - **Data Encryption Key (DEK)**: A high-entropy symmetric 256-bit key generated on-demand by KMS.
>
> 3. **The Protocol Mechanics**:
>    - When the application needs to encrypt a 10 MB file, it calls `kms:GenerateDataKey`.
>    - KMS returns two items: (1) the **Plaintext DEK**, and (2) the **Encrypted DEK** (encrypted under the Master Key).
>    - The application uses the Plaintext DEK to encrypt the 10 MB file locally in RAM using **AES-256-GCM** via native CPU hardware acceleration (Intel AES-NI), moving data at memory bus speeds ($> 3\text{ GB/s}$) in single-digit milliseconds.
>    - The application prepends the Encrypted DEK to the ciphertext on disk, and **immediately zeroes out the Plaintext DEK from RAM**.
>
> 4. **The Security Guarantee**:
>    - The data is secured with AES-256-GCM.
>    - The DEK is secured by the Master Key inside the HSM.
>    - Even if an attacker steals the storage disk, they cannot decrypt the data without calling KMS to decrypt the DEK, and every decryption call is logged in CloudTrail, delivering maximum security with zero throughput bottlenecks."

## 20. Hands-on Exercise
**Objective**: Execute an end-to-end Envelope Encryption workflow using the AWS CLI.

### Verification Steps
1. Create a Customer Managed Key alias: `alias/test-envelope-key`.
2. Generate a Data Encryption Key via CLI:
   ```bash
   aws kms generate-data-key --key-id alias/test-envelope-key --key-spec AES_256 \
     --output json > dek.json
   ```
3. Extract Plaintext DEK and Encrypted DEK:
   ```bash
   jq -r .Plaintext dek.json | base64 --decode > plaintext.key
   jq -r .CiphertextBlob dek.json > encrypted_dek.b64
   ```
4. Encrypt a local text file locally using OpenSSL with the Plaintext DEK:
   ```bash
   openssl enc -aes-256-cbc -in sample.txt -out sample.enc -pass file:./plaintext.key
   ```
5. **Securely shred the plaintext key from disk**:
   ```bash
   rm -f plaintext.key
   ```
6. Decrypt the Encrypted DEK via KMS:
   ```bash
   aws kms decrypt --ciphertext-blob fileb://<(base64 --decode encrypted_dek.b64) \
     --output text --query "Plaintext" | base64 --decode > recovered.key
   ```
7. Decrypt the file using the recovered key:
   ```bash
   openssl enc -d -aes-256-cbc -in sample.enc -out recovered.txt -pass file:./recovered.key
   ```
8. Confirm `sample.txt` and `recovered.txt` are identical, proving the end-to-end envelope encryption protocol.
