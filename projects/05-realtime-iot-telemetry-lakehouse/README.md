# Reference Project 05: Real-Time IoT Telemetry & Streaming Lakehouse

---

## 1. Executive Summary & Architecture Overview

This production reference architecture delivers a massive-scale IoT streaming telemetry ingestion and Open Table Format (Apache Iceberg) Lakehouse platform capable of ingesting **1,000,000 telemetry events per second** from connected industrial sensors, electric vehicle fleets, and smart grid meters.

Key Architectural Capabilities:
- **Massive Ingestion Ingress**: Amazon Kinesis Data Streams / Amazon MSK and OCI Streaming ingest multi-gigabit sensor telemetry across partitioned shards with sub-second buffer latency.
- **Continuous Stream Processing**: Apache Flink (via Amazon Managed Service for Apache Flink / EMR) and OCI Data Flow execute rolling-window aggregations, anomaly detection, and schema validation.
- **Open Table Format Lakehouse**: Telemetry records are written in Apache Iceberg / Parquet format directly into Amazon S3 / OCI Object Storage, eliminating proprietary storage lock-in.
- **Serverless Interactive Analytics**: Query engines (Amazon Athena / OCI Data Lakehouse + Autonomous Data Warehouse) provide ad-hoc SQL analytics and Grafana dashboards over petabytes of telemetry with sub-second query pruning.

---

## 2. Dual-Cloud System Topology

```
========================================================================================================================
                                REAL-TIME IOT TELEMETRY LAKEHOUSE TOPOLOGY
========================================================================================================================

                                [ Millions of IoT Devices / Connected Fleets ]
                                                       │
                                                       ▼ (MQTT / HTTPS Telemetry Ingress)
                            [ Edge Broker: AWS IoT Core / OCI Streaming Ingress ]
                                                       │
                                                       ▼
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  HIGH-THROUGHPUT STREAMING BUFFER (Partition Key: device_id_hash)
  ┌───────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────┐
  │ AWS: Amazon Kinesis Data Streams (On-Demand Mode) │ OCI: OCI Streaming Service (Dedicated Stream Pools)           │
  │  - 1,000,000 events/sec capacity                  │  - Kafka-compatible REST and binary producer protocol         │
  └─────────────────────────┬─────────────────────────┴───────────────────────────────┬───────────────────────────────┘
                            │                                                         │
                            ▼                                                         ▼
  CONTINUOUS STREAM PROCESSING ENGINE (Stateful Complex Event Processing)
  ┌───────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────┐
  │ AWS: Managed Service for Apache Flink (Flink 1.19)│ OCI: OCI Data Flow (Managed Apache Spark / Flink Streaming)   │
  │  - 1-minute tumbling window anomaly detection     │  - Real-time device vibration & temperature alerting          │
  │  - Compacts micro-batches into Parquet Iceberg    │  - Writes ACID Iceberg metadata manifests to Object Storage   │
  └─────────────────────────┬─────────────────────────┴───────────────────────────────┬───────────────────────────────┘
                            │                                                         │
                            ▼                                                         ▼
  OPEN TABLE FORMAT (APACHE ICEBERG) LAKEHOUSE TIER
  ┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │ Storage: Amazon S3 Standard (Tiered to S3 Glacier) / OCI Object Storage Standard (Tiered to Archive)              │
  │ Metadata Catalog: AWS Glue Data Catalog / OCI Data Catalog (Iceberg REST Catalog Compliant)                       │
  │ Format: Snappy-compressed Apache Parquet with column-level min/max dictionary pruning                             │
  └───────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────┘
                                                      │
                                                      ▼
  INTERACTIVE SERVERLESS QUERY & ANALYTICS
  ┌───────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────┐
  │ AWS: Amazon Athena (Engine v3) + Amazon QuickSight│ OCI: Autonomous Data Warehouse (ADW) + OCI Data Science       │
  │ - Sub-second SQL partition pruning over petabytes │ - Direct external tables querying Object Storage Lakehouse    │
  └───────────────────────────────────────────────────┴───────────────────────────────────────────────────────────────┘
  ══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
```

---

## 3. Dual-Cloud Component Mapping Matrix

| Architectural Function | AWS Cloud Implementation | OCI Cloud Implementation | Implementation Notes |
| :--- | :--- | :--- | :--- |
| **IoT Ingestion Gateway** | AWS IoT Core (MQTT / HTTP) `[Doc: AWS IoT, checked 2026]` | OCI Streaming Service (Kafka Ingress) `[Doc: OCI Streaming, checked 2026]` | High-scale device connectivity and protocol translation. |
| **Streaming Buffer** | Amazon Kinesis Data Streams (On-Demand) | OCI Streaming (Dedicated Stream Pool) | Buffers 1M events/sec with multi-AZ/AD durability and 7-day retention. |
| **Stream Processing Engine**| Managed Apache Flink (Kinesis Data Analytics) | OCI Data Flow (Streaming Spark / Flink) | Executes sliding-window anomaly detection and micro-batch Parquet file compaction. |
| **Lakehouse Table Format** | Apache Iceberg via AWS Glue Data Catalog | Apache Iceberg via OCI Data Catalog | ACID table format with schema evolution, hidden partitioning, and time travel. |
| **Object Lake Storage** | Amazon S3 Standard with Intelligent-Tiering | OCI Object Storage with Auto-Tiering | Highly durable storage backing for petabytes of historical IoT telemetry. |
| **Interactive Query Tier**| Amazon Athena (Presto/Trino-based) | OCI Autonomous Data Warehouse (External Tables) | Serverless SQL engine executing ad-hoc queries over Iceberg tables without data movement. |

---

## 4. Production Infrastructure as Code (Terraform HCL)

### 4.1 AWS Terraform Module (`aws_iot_lakehouse.tf`)

```hcl
# AWS Reference Implementation: Kinesis Data Stream and Glue Iceberg Catalog
resource "aws_kinesis_stream" "iot_stream" {
  name             = "prod-iot-telemetry-stream"
  stream_mode_details {
    stream_mode = "ON_DEMAND" # Automatically scales up to 1M events/sec
  }

  retention_period = 168 # 7 Days retention
  encryption_type  = "KMS"
  kms_key_id       = aws_kms_key.iot_kms_key.id

  tags = {
    Environment = "production"
    Application = "telemetry-lakehouse"
  }
}

resource "aws_glue_catalog_database" "lakehouse_db" {
  name = "prod_iot_lakehouse"
}

resource "aws_glue_catalog_table" "iceberg_telemetry" {
  name          = "vehicle_telemetry"
  database_name = aws_glue_catalog_database.lakehouse_db.name
  table_type    = "EXTERNAL_TABLE"

  parameters = {
    "table_type" = "ICEBERG"
    "format"     = "PARQUET"
  }

  storage_descriptor {
    location      = "s3://${aws_s3_bucket.lakehouse_bucket.id}/tables/vehicle_telemetry/"
    input_format  = "org.apache.iceberg.mr.hive.HiveIcebergInputFormat"
    output_format = "org.apache.iceberg.mr.hive.HiveIcebergOutputFormat"
  }
}
```

### 4.2 OCI Terraform Module (`oci_iot_lakehouse.tf`)

```hcl
# OCI Reference Implementation: OCI Stream Pool and Object Storage Bucket
resource "oci_streaming_stream_pool" "iot_pool" {
  compartment_id = var.compartment_ocid
  name           = "prod-iot-stream-pool"

  kafka_settings {
    auto_create_topics_enable = false
    log_retention_in_hours    = 168
    num_partitions            = 64
  }

  custom_encryption_key_id = oci_kms_key.vault_iot_key.id
}

resource "oci_objectstorage_bucket" "iot_lakehouse" {
  compartment_id = var.compartment_ocid
  name           = "prod_iot_lakehouse_bucket"
  namespace      = var.object_storage_namespace
  storage_tier   = "Standard"
  auto_tiering   = "InfrequentAccess"

  versioning     = "Enabled"
}
```

---

## 5. Security, Workload Identity & Encryption

### 5.1 Device Attestation & Lakehouse Encryption
- Devices authenticate to the edge broker using **X.509 Device Certificates** registered in AWS IoT Core / OCI Identity Domains.
- Stream processors assume IAM roles via EKS Pod Identity / OCI Workload Identity to write Parquet files to Object Storage.
- KMS Customer Managed Keys (CMK) / OCI Vault keys encrypt all stream partitions and Parquet files at rest.

---

## 6. Observability, SLIs/SLOs & Alerting Runbook

1. **Ingestion MillisBehindLatest**: Measures stream processing lag. Alert if $> 10,000\text{ms}$.
2. **Kinesis / OCI WriteThroughputExceeded**: Alert if partition writes trigger throttling.
3. **Parquet File Compaction Health**: Alert if small-file count in Iceberg table exceeds 5,000 without compaction.

---

## 7. Deployment & Verification Runbook

```bash
# Query Iceberg Lakehouse via Amazon Athena
aws athena start-query-execution   --query-string "SELECT device_id, AVG(battery_temp), MAX(speed_kph) FROM prod_iot_lakehouse.vehicle_telemetry WHERE event_date = CURRENT_DATE GROUP BY device_id HAVING AVG(battery_temp) > 65.0;"   --work-group "primary"   --result-configuration "OutputLocation=s3://${ATHENA_RESULTS_BUCKET}/"
```

---

## 8. FinOps Cost Breakdown & Sizing Economics

```text
====================================================================================================
                   FINOPS COST BREAKDOWN (1M EVENTS/SEC = 86.4B EVENTS/DAY)
====================================================================================================

TIER                             AWS MONTHLY COST        OCI MONTHLY COST
----------------------------------------------------------------------------------------------------
Streaming Buffer (Kinesis/Stream)$4,800                  $3,100
Flink / Spark Compute (EMR/Flow) $8,400                  $5,200
Iceberg Lakehouse Storage (S3)   $12,500 (Tiered)        $10,200 (Tiered)
Query Analytics (Athena / ADW)   $2,200                  $2,800 (ADW ECPUs)
----------------------------------------------------------------------------------------------------
TOTAL RUN-RATE                   $27,900 / month         $21,300 / month
UNIT COST PER 1M EVENTS          ~$0.0107                ~$0.0082
====================================================================================================
```

---

## 9. Failure Mode Drills & Chaos Engineering Runbook

### 9.1 Game Day Drill: Stream Processing Job Checkpoint Failure
1. **Action**: Kill active Flink / Data Flow worker tasks mid-stream during peak ingestion.
2. **Verification**:
   - The streaming broker buffers incoming sensor events across 3 AZs/ADs without data loss.
   - Stream engine recovers from the latest Apache Iceberg metadata snapshot checkpoint.
   - Zero duplicate records written to the Lakehouse due to Iceberg ACID upsert semantics.
