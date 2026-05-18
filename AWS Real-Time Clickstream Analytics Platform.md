# AWS Real-Time Clickstream Analytics Platform

[![Architecture](https://img.shields.io/badge/Architecture-Serverless%20Event--Driven-blueviolet?style=for-the-badge&logo=amazonaws)](https://aws.amazon.com)
[![Storage](https://img.shields.io/badge/Storage-Amazon%20S3%2011%20Nines-orange?style=for-the-badge&logo=amazons3)](https://aws.amazon.com/s3/)
[![Ingestion](https://img.shields.io/badge/Ingestion-Kinesis%20Data%20Firehose-red?style=for-the-badge&logo=amazonaws)](https://aws.amazon.com/kinesis/data-firehose/)
[![Query Engine](https://img.shields.io/badge/Query-Amazon%20Athena%20%7C%20Serverless%20SQL-232F3E?style=for-the-badge&logo=amazonaws)](https://aws.amazon.com/athena/)
[![Visualization](https://img.shields.io/badge/BI-Amazon%20QuickSight-FF9900?style=for-the-badge&logo=amazonaws)](https://aws.amazon.com/quicksight/)
[![IaC](https://img.shields.io/badge/IaC-AWS%20CloudFormation-red?style=for-the-badge&logo=amazonaws)](https://aws.amazon.com/cloudformation/)
[![Security](https://img.shields.io/badge/Security-IAM%20Least%20Privilege-green?style=for-the-badge&logo=amazonaws)](https://aws.amazon.com/iam/)
[![Compliance](https://img.shields.io/badge/Compliance-Encryption%20At%20Rest%20%26%20Transit-blue?style=for-the-badge)](https://aws.amazon.com/compliance/)

---

> **⚠️ PRODUCTION NOTICE:** This platform is architected for high-throughput, compliance-regulated behavioral analytics workloads. All components operate under a **least-privilege IAM model**, with encryption enforced both in transit (TLS) and at rest (SSE-S3/SSE-KMS) at no additional cost. Do **not** modify bucket policies, IAM trust relationships, or Firehose delivery stream configurations without a formal change-management review.

---

## Table of Contents

- [Platform Overview](#platform-overview)
- [Architecture Deep Dive](#architecture-deep-dive)
- [Technology Stack & Justification Matrix](#technology-stack--justification-matrix)
- [Security & Compliance Posture](#security--compliance-posture)
- [Prerequisites](#prerequisites)
- [Deployment Guide](#deployment-guide)
  - [Task 1 — IAM Policies & Roles](#task-1--iam-policies--roles)
  - [Task 2 — S3 Data Lake Bucket](#task-2--s3-data-lake-bucket)
  - [Task 3 — Lambda Transformation Function](#task-3--lambda-transformation-function)
  - [Task 4 — Kinesis Data Firehose Delivery Stream](#task-4--kinesis-data-firehose-delivery-stream)
  - [Task 5 — S3 Bucket Policy for Firehose Ingress](#task-5--s3-bucket-policy-for-firehose-ingress)
  - [Task 6 — API Gateway REST Ingestion Endpoint](#task-6--api-gateway-rest-ingestion-endpoint)
  - [Task 7 — Amazon Athena Query Layer](#task-7--amazon-athena-query-layer)
  - [Task 8 — QuickSight Business Intelligence Dashboard](#task-8--quicksight-business-intelligence-dashboard)
- [Event Schema Specification](#event-schema-specification)
- [Environment & Configuration Reference](#environment--configuration-reference)
- [Storage Architecture & Cost Optimization](#storage-architecture--cost-optimization)
- [Kinesis Service Selection Matrix](#kinesis-service-selection-matrix)
- [Athena Query Optimization Guide](#athena-query-optimization-guide)
- [Architecture Enhancement Roadmap](#architecture-enhancement-roadmap)
- [Operational Runbook](#operational-runbook)
- [Resource Teardown Procedure](#resource-teardown-procedure)

---

## Platform Overview

This repository defines a **production-grade, fully serverless, real-time behavioral analytics pipeline** built entirely on AWS-managed services. The platform ingests high-velocity clickstream events emitted by client-side JavaScript from multi-tenant digital menus (QR-code-driven Progressive Web Applications), processes and durably stores them in an encrypted S3-backed data lake, and exposes the aggregated dataset through an interactive QuickSight BI layer for operational decision-making.

**Primary Business Objective:** Enable multi-tenant restaurant operators to derive behavioral insights — dwell time per menu item, navigation flow analysis, category conversion rates — without requiring dedicated data engineering staff or on-premise infrastructure.

**Core Design Principles:**

| Principle | Implementation |
|---|---|
| **Serverless-First** | No EC2 instances, no cluster management, zero idle compute cost |
| **Separation of Storage & Compute** | S3 as immutable data lake; Athena as ephemeral query engine |
| **Pay-Per-Use Billing** | S3 (per GB stored), Athena (per TB scanned), Firehose (per GB ingested) |
| **Operational Simplicity** | Designed for reduced-staff environments; SQL-only query interface |
| **Durability by Default** | S3 provides 99.999999999% (11 nines) object durability via multi-AZ replication |
| **Encryption Everywhere** | TLS in transit; SSE at rest — enforced by default, zero additional cost |

---

## Architecture Deep Dive
```

┌─────────────────────────────────────────────────────────────────────────────────┐ │ CLIENT TIER (Out of Scope) │ │ │ │ [Mobile / Browser] ──scan──▶ [QR Code] ──▶ [S3 Static Site / CloudFront] │ │ │ │ │ │ │ JavaScript SDK emits clickstream events │ HTML/JS served │ │ │ via HTTPS POST │ │ └─────────┼────────────────────────────────────────────────────────────────────────┘ │ HTTPS POST /poc (application/json) ▼ ┌─────────────────────────────────────────────────────────────────────────────────┐ │ INGESTION TIER (In Scope) │ │ │ │ ┌──────────────────────┐ VTL Mapping ┌─────────────────────────────┐ │ │ │ API Gateway REST │ ──────────────────▶ │ Kinesis Data Firehose │ │ │ │ (Regional, POST) │ PutRecord API │ (Direct PUT delivery │ │ │ │ IAM Role: API- │ Base64-encoded │ stream, 60s buffer) │ │ │ │ Firehose │ JSON payload │ │ │ │ └──────────────────────┘ └──────────┬──────────────────┘ │ │ │ │ │ ┌──────────▼──────────────────┐ │ │ │ AWS Lambda (Python 3.8) │ │ │ │ transform-data │ │ │ │ • Base64 decode │ │ │ │ • Append newline delimiter │ │ │ │ • Base64 re-encode │ │ │ │ • Timeout: 10s │ │ │ └──────────┬──────────────────┘ │ └──────────────────────────────────────────────────────────┼──────────────────────┘ │ Transformed records ▼ ┌─────────────────────────────────────────────────────────────────────────────────┐ │ STORAGE TIER (In Scope) │ │ │ │ ┌──────────────────────────────────────────────────────────────────────────┐ │ │ │ Amazon S3 (Private Bucket — Encryption: SSE-S3) │ │ │ │ Partitioned: s3://<bucket>/yyyy/MM/dd/HH/<object> │ │ │ │ Durability: 99.999999999% │ Cross-AZ replication │ Lifecycle: IT │ │ │ └──────────────────────────────────────────────────────────────────────────┘ │ └─────────────────────────────────────────────────────────────────────────────────┘ │ ▼ ┌─────────────────────────────────────────────────────────────────────────────────┐ │ QUERY TIER (In Scope) │ │ │ │ ┌──────────────────────────────────────────────────────────────────────────┐ │ │ │ Amazon Athena (Serverless SQL — JsonSerDe — Partition Projection) │ │ │ │ Table: my_ingested_data │ Format: JSON │ Pay-per-scan │ │ │ └──────────────────────────────────────────────────────────────────────────┘ │ └─────────────────────────────────────────────────────────────────────────────────┘ │ ▼ ┌─────────────────────────────────────────────────────────────────────────────────┐ │ VISUALIZATION TIER (In Scope) │ │ │ │ ┌──────────────────────────────────────────────────────────────────────────┐ │ │ │ Amazon QuickSight (Enterprise BI — SPICE Engine — Session-based Billing│ │ │ │ Data Source: Athena │ Dataset: poc-clickstream │ Visuals: Pie, Bar, KPI │ │ │ └──────────────────────────────────────────────────────────────────────────┘ │ └─────────────────────────────────────────────────────────────────────────────────┘

```

---

## Technology Stack & Justification Matrix

The following matrix documents the evaluated service candidates for each architectural layer and the rationale for inclusion or elimination.

### Storage Layer

| Service | Evaluated | Decision | Justification |
|---|---|---|---|
| **Amazon S3** | ✅ | ✅ **SELECTED** | API-driven, storage-compute decoupled, 11-nines durability, cross-Region replication, pay-per-byte, Intelligent-Tiering, no instance attachment required |
| Amazon EBS | ✅ | ❌ Eliminated | Requires EC2 attachment, billed on allocated capacity (not actual usage), durability limited to AZ-level replication |
| Amazon EFS | ✅ | ❌ Eliminated | Requires attachment to compute resources (EC2/containers/servers); does not fulfill storage-compute separation requirement |

### Ingestion Layer

| Service | Evaluated | Decision | Justification |
|---|---|---|---|
| **Kinesis Data Firehose** | ✅ | ✅ **SELECTED** | Fully managed, no consumer code required, native S3 delivery, Lambda transformation support, 60s latency acceptable per customer SLA |
| Kinesis Data Streams | ✅ | ❌ Eliminated | Requires custom producer/consumer code; sub-second latency not required by this workload |
| Kinesis Data Analytics | ✅ | ❌ Eliminated | Real-time transformation not required; data arrives pre-structured in JSON |
| Amazon EMR | ✅ | ❌ Eliminated | Cluster-based billing model; requires big-data framework expertise; conflicts with reduced-staff operational constraint |
| AWS DMS | ✅ | ❌ Eliminated | Purpose-built for database migrations; no database source exists in this architecture |
| AWS Data Exchange | ✅ | ❌ Eliminated | Designed for third-party data catalog integration; not applicable to internal event stream ingestion |

### Query Layer

| Service | Evaluated | Decision | Justification |
|---|---|---|---|
| **Amazon Athena** | ✅ | ✅ **SELECTED** | Serverless SQL on S3, no ETL required, pay-per-scan, JSON SerDe support matches payload format, immediately accessible to SQL-skilled analysts |
| Amazon S3 Select | ✅ | ❌ Eliminated | Single-object query scope; incompatible with multi-file partitioned data lake |
| AWS Glue | ✅ | ❌ Eliminated | Powerful for ETL/unstructured data but introduces learning curve and operational overhead incompatible with reduced-staff constraint |
| Amazon EMR | ✅ | ❌ Eliminated | Cluster management overhead; cost model misaligned with pay-per-query requirement |

### Visualization Layer

| Service | Evaluated | Decision | Justification |
|---|---|---|---|
| **Amazon QuickSight** | ✅ | ✅ **SELECTED** | Native Athena integration, SPICE in-memory engine, session-based billing, no servers to manage, pre-existing team familiarity confirmed by customer |
| Amazon CloudWatch | ✅ | ❌ Eliminated | Operational metrics platform; not designed for business intelligence or behavioral analytics |
| Amazon OpenSearch | ✅ | ❌ Eliminated | Embedded storage+processing violates storage-compute separation; higher cost profile than required |
| Amazon Managed Grafana | ✅ | ⚠️ Runner-up | Viable for time-series visualization; deprioritized due to customer team's existing QuickSight proficiency |

---

## Security & Compliance Posture

> **🔒 SECURITY MANDATE:** All IAM roles in this architecture are constructed on the principle of **least privilege**. No wildcard resource ARNs (`"Resource": "*"`) should exist in production deployments beyond the initial `firehose:PutRecord` bootstrap policy. Replace with explicit delivery stream ARNs post-deployment.

### IAM Role Architecture

| Role Name | Trusted Principal | Attached Policies | Purpose |
|---|---|---|---|
| `APIGateway-Firehose` | `apigateway.amazonaws.com` | `API-Firehose` (custom), `AmazonAPIGatewayPushToCloudWatchLogs` (AWS-managed) | Authorizes API Gateway to invoke `firehose:PutRecord` and emit execution logs to CloudWatch |
| `KinesisFirehoseServiceRole-*` | `firehose.amazonaws.com` | Auto-generated by Firehose service | Authorizes Firehose to invoke the Lambda transformer and write objects to the target S3 bucket |

### Encryption Controls

| Data State | Mechanism | Key Management |
|---|---|---|
| **In Transit** | TLS 1.2+ enforced on all API Gateway and S3 endpoints | AWS-managed certificates |
| **At Rest (S3)** | SSE-S3 (AES-256) by default | AWS-managed S3 keys; upgrade to SSE-KMS for BYOK compliance |
| **In Transit (Firehose → Lambda)** | Internal AWS network encryption | Managed by AWS service fabric |

### S3 Bucket Security Baseline

- ✅ Public access blocked by default (`BlockPublicAcls`, `BlockPublicPolicy`, `IgnorePublicAcls`, `RestrictPublicBuckets`)
- ✅ Bucket policy scoped exclusively to the Firehose IAM role ARN
- ✅ No cross-account access configured
- ✅ Versioning recommended for production deployments (enable separately)

---

## Prerequisites

Before initiating deployment, confirm the following:

- [ ] AWS account with administrator-equivalent permissions (for initial bootstrap only)
- [ ] AWS Management Console access or AWS CLI v2 configured with appropriate credentials
- [ ] Target Region set to `us-east-1` (US East — N. Virginia) throughout all tasks
- [ ] QuickSight subscription confirmed (Standard or Enterprise Edition)
- [ ] All placeholder values (`<account ID>`, bucket ARNs, role ARNs) collected in a secure scratchpad before proceeding
- [ ] Familiarity with JSON IAM policy syntax and VTL (Velocity Template Language) for API Gateway mapping templates

---

## Deployment Guide

> **ℹ️ REGION LOCK:** All resources **must** be deployed in `us-east-1`. Cross-region deployments will require corresponding updates to IAM trust policies, Firehose stream configurations, and Athena query result buckets.

---

### Task 1 — IAM Policies & Roles

#### Step 1.1 — Create the `API-Firehose` Custom IAM Policy

Navigate to **IAM → Policies → Create policy** and apply the following JSON document:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "FirehosePutRecordAccess",
            "Effect": "Allow",
            "Action": "firehose:PutRecord",
            "Resource": "*"
        }
    ]
}
```

> **⚠️ HARDENING NOTE:** The `"Resource": "*"` scope is acceptable for initial deployment. Once the Kinesis Data Firehose delivery stream ARN is known, replace `*` with the explicit stream ARN (e.g., `arn:aws:firehose:us-east-1:<account-id>:deliverystream/<stream-name>`) to enforce least-privilege.

- **Policy Name:** `API-Firehose`

#### Step 1.2 — Create the `APIGateway-Firehose` IAM Execution Role

Navigate to **IAM → Roles → Create role** and configure:

| Setting | Value |
|---|---|
| Trusted entity type | AWS service |
| Use case | API Gateway |
| Role name | `APIGateway-Firehose` |

After role creation, attach the following policies:

1. **Custom Policy:** `API-Firehose` (created above)
2. **AWS Managed Policy:** `AmazonAPIGatewayPushToCloudWatchLogs`

> **📋 RECORD:** Copy and securely store the full ARN of the `APIGateway-Firehose` role. It will be required in Task 6.
> Format: `arn:aws:iam::<account-id>:role/APIGateway-Firehose`

---

### Task 2 — S3 Data Lake Bucket

Navigate to **S3 → Create bucket** and apply the following configuration:

| Setting | Value |
|---|---|
| Bucket name | `<org>-clickstream-analytics-<env>` (must be globally unique) |
| AWS Region | `us-east-1` |
| Object Ownership | ACLs disabled (recommended) |
| Block Public Access | **All options enabled (default — do not modify)** |
| Versioning | Optional for POC; **mandatory for production** |
| Encryption | SSE-S3 (AES-256) — enabled by default |

> **📋 RECORD:** After bucket creation, navigate to **Properties** and copy the bucket ARN.
> Format: `arn:aws:s3:::<bucket-name>`

---

### Task 3 — Lambda Transformation Function

The Lambda function serves as the Firehose data transformation hook. Its sole responsibility is to append a newline character (`\n`) to each decoded record before re-encoding, ensuring Athena can correctly parse line-delimited JSON objects from the S3 objects.

Navigate to **Lambda → Create function → Use a blueprint**.

| Setting | Value |
|---|---|
| Blueprint filter | `Kinesis` |
| Blueprint | `Process records sent to a Kinesis Firehose stream` (Python 3.8) |
| Function name | `transform-data` |
| Runtime | Python 3.8 |

After creation, replace the default blueprint code with the following production-ready handler:

```python
import json
import boto3
import base64

output = []

def lambda_handler(event, context):
    """
    Kinesis Data Firehose transformation handler.

    Decodes each Base64-encoded record from the Firehose stream,
    appends a newline delimiter required by Athena's line-delimited
    JSON parsing, and re-encodes before returning to Firehose.

    Args:
        event (dict): Firehose event object containing 'records' list.
        context (LambdaContext): Lambda execution context.

    Returns:
        dict: Transformed records with 'Ok' result status.
    """
    for record in event['records']:
        # Decode the Base64-encoded payload from Firehose
        payload = base64.b64decode(record['data']).decode('utf-8')

        # Append newline — required for Athena line-delimited JSON parsing
        row_w_newline = payload + "\n"

        # Re-encode to Base64 for Firehose downstream delivery
        row_w_newline = base64.b64encode(row_w_newline.encode('utf-8'))

        output_record = {
            'recordId': record['recordId'],
            'result': 'Ok',
            'data': row_w_newline
        }
        output.append(output_record)

    return {'records': output}
```

After deploying the code, update the function configuration:

| Configuration | Setting | Value |
|---|---|---|
| General | Timeout | `10 seconds` |
| General | Memory | `128 MB` (default, sufficient for POC) |

> **📋 RECORD:** Copy the Lambda function ARN from the Function overview panel.
> Format: `arn:aws:lambda:us-east-1:<account-id>:function:transform-data`

---

### Task 4 — Kinesis Data Firehose Delivery Stream

#### Step 4.1 — Create the Delivery Stream

Navigate to **Kinesis → Kinesis Data Firehose → Create delivery stream**.

| Setting | Value |
|---|---|
| Source | Direct PUT |
| Destination | Amazon S3 |
| Enable data transformation | ✅ Enabled |
| AWS Lambda function ARN | `<Lambda ARN from Task 3>` |
| Lambda version/alias | `$LATEST` |
| S3 bucket | Select the bucket created in Task 2 |
| Buffer interval | `60 seconds` (minimum) |
| Compression | Disabled (for Athena JSON compatibility in POC) |
| Encryption | SSE with AWS-owned CMK |

> **⏱️ NOTE:** Delivery stream provisioning takes approximately 2–5 minutes. Monitor the status until it transitions to **Active** before proceeding.

#### Step 4.2 — Capture the Firehose IAM Role ARN

1. Open the delivery stream details page.
2. Navigate to **Configuration → Service access → IAM role**.
3. The IAM console will open scoped to the auto-generated Firehose service role.

> **📋 RECORD:** Copy the Firehose service role ARN.
> Format: `arn:aws:iam::<account-id>:role/service-role/KinesisFirehoseServiceRole-PUT-S3-<suffix>`

---

### Task 5 — S3 Bucket Policy for Firehose Ingress

Navigate to **S3 → `<your-bucket>` → Permissions → Bucket policy → Edit**.

Apply the following policy, substituting all placeholder values before saving:

```json
{
    "Version": "2012-10-17",
    "Id": "FirehoseDeliveryStreamAccess",
    "Statement": [
        {
            "Sid": "AllowFirehoseDelivery",
            "Effect": "Allow",
            "Principal": {
                "AWS": "<PASTE: Firehose IAM Role ARN from Task 4.2>"
            },
            "Action": [
                "s3:AbortMultipartUpload",
                "s3:GetBucketLocation",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:ListBucketMultipartUploads",
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": [
                "<PASTE: S3 Bucket ARN from Task 2>",
                "<PASTE: S3 Bucket ARN from Task 2>/*"
            ]
        }
    ]
}
```

**Example (populated):**

```json
{
    "Version": "2012-10-17",
    "Id": "FirehoseDeliveryStreamAccess",
    "Statement": [
        {
            "Sid": "AllowFirehoseDelivery",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::123456789012:role/service-role/KinesisFirehoseServiceRole-PUT-S3-AbCdE-us-east-1-1234567890"
            },
            "Action": [
                "s3:AbortMultipartUpload",
                "s3:GetBucketLocation",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:ListBucketMultipartUploads",
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": [
                "arn:aws:s3:::my-org-clickstream-analytics-poc",
                "arn:aws:s3:::my-org-clickstream-analytics-poc/*"
            ]
        }
    ]
}
```

---

### Task 6 — API Gateway REST Ingestion Endpoint

Navigate to **API Gateway → Create API → REST API → Build**.

#### Step 6.1 — Create the API

| Setting | Value |
|---|---|
| Protocol | REST |
| Create new API | New API |
| API name | `clickstream-ingest-poc` |
| Endpoint type | Regional |

#### Step 6.2 — Create Resource & Method

1. In the Resources pane: **Actions → Create Resource** → Resource Name: `poc`
2. Select `/poc`: **Actions → Create Method → POST** (confirm with checkmark)

#### Step 6.3 — Configure AWS Service Integration

| Setting | Value |
|---|---|
| Integration type | AWS Service |
| AWS Region | `us-east-1` |
| AWS Service | `Firehose` |
| HTTP method | `POST` |
| Action type | Use action name |
| Action | `PutRecord` |
| Execution role | `<APIGateway-Firehose role ARN from Task 1.2>` |
| Content handling | Passthrough |

#### Step 6.4 — Configure VTL Mapping Template

Navigate to **Integration Request → Mapping Templates**:

- Request body passthrough: `When there are no templates defined (recommended)`
- **Add mapping template** → Content-Type: `application/json`

Paste the following VTL transformation script, replacing the `DeliveryStreamName` value:

```velocity
{
    "DeliveryStreamName": "<PASTE: Your Firehose Delivery Stream Name>",
    "Record": {
        "Data": "$util.base64Encode($util.escapeJavaScript($input.json('$')).replace('\', ''))"
    }
}
```

> **🔍 VTL EXPLANATION:** This template performs three operations: (1) serializes the entire incoming JSON body via `$input.json('$')`, (2) escapes JavaScript special characters, (3) Base64-encodes the payload to satisfy the `firehose:PutRecord` `Data` field binary encoding requirement.

#### Step 6.5 — Validate Integration via Test Console

Navigate to **/poc → POST → Test** and submit the following representative clickstream payloads sequentially. Confirm `HTTP 200` and `"Successfully completed execution"` in the response logs for each:

```json
{
    "element_clicked": "entree_1",
    "time_spent": 67,
    "source_menu": "restaurant_name",
    "created_at": "2024-01-15 14:30:00"
}
```

```json
{
    "element_clicked": "entree_4",
    "time_spent": 32,
    "source_menu": "restaurant_name",
    "created_at": "2024-01-15 14:31:00"
}
```

```json
{
    "element_clicked": "drink_1",
    "time_spent": 15,
    "source_menu": "restaurant_name",
    "created_at": "2024-01-15 14:32:00"
}
```

```json
{
    "element_clicked": "drink_3",
    "time_spent": 14,
    "source_menu": "restaurant_name",
    "created_at": "2024-01-15 14:33:00"
}
```

> **⏱️ BUFFER NOTICE:** Kinesis Data Firehose buffers records for the configured interval (60 seconds minimum) before flushing to S3. Wait at least 90 seconds after the final test invocation before verifying S3 object delivery. Verify the object exists in S3 under the datetime-partitioned prefix (e.g., `s3://<bucket>/2024/01/15/14/`) before proceeding to Task 7.

---

### Task 7 — Amazon Athena Query Layer

#### Step 7.1 — Configure Query Result Location

Navigate to **Athena → Query editor → Settings → Manage**:

- Set query result location to: `s3://<your-bucket>/athena-results/`

> **⚠️ PREREQUISITE:** This setting **must** be configured before executing any DDL or DML queries in Athena. Failure to set this will result in query execution errors.

#### Step 7.2 — Create the External Table

In the Athena query editor, execute the following DDL statement after substituting your bucket name in both `LOCATION` and `storage.location.template`:

```sql
CREATE EXTERNAL TABLE my_ingested_data (
    element_clicked STRING,
    time_spent      INT,
    source_menu     STRING,
    created_at      STRING
)
PARTITIONED BY (
    datehour STRING
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES (
    'paths' = 'element_clicked, time_spent, source_menu, created_at'
)
LOCATION "s3://<your-bucket-name>/"
TBLPROPERTIES (
    "projection.enabled"                  = "true",
    "projection.datehour.type"            = "date",
    "projection.datehour.format"          = "yyyy/MM/dd/HH",
    "projection.datehour.range"           = "2021/01/01/00,NOW",
    "projection.datehour.interval"        = "1",
    "projection.datehour.interval.unit"   = "HOURS",
    "storage.location.template"           = "s3://<your-bucket-name>/${datehour}/"
);
```

> **🔍 ARCHITECTURE NOTE:** The `CREATE EXTERNAL TABLE` keyword is critical. Athena does **not** copy or own the data — it reads directly from S3. The `EXTERNAL` designation ensures that a `DROP TABLE` operation removes only the Athena metadata definition, leaving the underlying S3 objects intact.

#### Step 7.3 — Validate Data Ingestion

Open a new query tab and execute:

```sql
SELECT
    element_clicked,
    time_spent,
    source_menu,
    created_at
FROM my_ingested_data
LIMIT 100;
```

Confirm that all submitted test payloads are present in the result set.

---

### Task 8 — QuickSight Business Intelligence Dashboard

#### Step 8.1 — Configure QuickSight S3 & Athena Permissions

Navigate to **QuickSight → user menu (top-right) → Manage QuickSight → Security & permissions → Manage** under "QuickSight access to AWS services":

- [ ] Enable **Amazon S3** → Select your clickstream bucket
- [ ] Enable **Write permission for Athena Workgroup**
- [ ] Save changes

#### Step 8.2 — Create Athena Data Source

Navigate to **QuickSight → Datasets → New dataset → Athena**:

| Setting | Value |
|---|---|
| Data source name | `poc-clickstream` |
| Athena workgroup | `[primary]` |

Choose **Create data source** → Select table `my_ingested_data` → **Select**.

#### Step 8.3 — Configure SPICE Import & Build Analysis

- Import mode: **Import to SPICE for quicker analytics** → **Visualize**
- In the analysis canvas, explore the following visualizations:
  - **Pie chart** → Field: `element_clicked` — Menu item click distribution
  - **Bar chart** → X-axis: `element_clicked`, Value: `time_spent` (Average) — Dwell time per item
  - **KPI** → Value: `time_spent` (Sum) — Total engagement seconds

---

## Event Schema Specification

All clickstream events submitted to the API Gateway endpoint must conform to the following JSON schema:

```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "title": "ClickstreamEvent",
    "description": "Behavioral event emitted by client-side JS on menu interaction",
    "type": "object",
    "required": ["element_clicked", "time_spent", "source_menu", "created_at"],
    "properties": {
        "element_clicked": {
            "type": "string",
            "description": "Identifier of the menu element that triggered the event",
            "example": "entree_1"
        },
        "time_spent": {
            "type": "integer",
            "description": "Duration in seconds the user dwelled on the element",
            "minimum": 0,
            "example": 67
        },
        "source_menu": {
            "type": "string",
            "description": "Tenant identifier — the restaurant menu that served this event",
            "example": "restaurant_name"
        },
        "created_at": {
            "type": "string",
            "format": "date-time",
            "description": "ISO 8601 timestamp of the event occurrence",
            "example": "2024-01-15 14:30:00"
        }
    },
    "additionalProperties": false
}
```

---

## Environment & Configuration Reference

| Parameter | Description | Example Value | Source |
|---|---|---|---|
| `AWS_REGION` | Deployment region for all resources | `us-east-1` | Hardcoded — do not modify |
| `S3_BUCKET_NAME` | Target data lake bucket name | `my-org-clickstream-analytics-poc` | Task 2 output |
| `S3_BUCKET_ARN` | Full ARN of the S3 bucket | `arn:aws:s3:::my-org-clickstream-analytics-poc` | Task 2 — Properties tab |
| `LAMBDA_FUNCTION_ARN` | ARN of the transform-data Lambda | `arn:aws:lambda:us-east-1:123456789012:function:transform-data` | Task 3 — Function overview |
| `FIREHOSE_STREAM_NAME` | Kinesis Firehose delivery stream name | `PUT-S3-AbCdE` | Task 4 — Delivery streams list |
| `FIREHOSE_ROLE_ARN` | IAM role ARN auto-created by Firehose | `arn:aws:iam::123456789012:role/service-role/KinesisFirehoseServiceRole-...` | Task 4.2 |
| `API_GATEWAY_ROLE_ARN` | IAM execution role ARN for API Gateway | `arn:aws:iam::123456789012:role/APIGateway-Firehose` | Task 1.2 |
| `ATHENA_TABLE_NAME` | Athena external table name | `my_ingested_data` | Task 7 |
| `QUICKSIGHT_DATA_SOURCE` | QuickSight Athena data source name | `poc-clickstream` | Task 8 |

---

## Storage Architecture & Cost Optimization

### S3 Intelligent-Tiering

For production deployments with unpredictable access patterns, enable **S3 Intelligent-Tiering** on the clickstream bucket. The service automatically migrates objects between access tiers based on observed access patterns, with no retrieval fees.

| Access Tier | Activation Condition | Storage Cost |
|---|---|---|
| Frequent Access | Active access | Standard rate |
| Infrequent Access | No access for 30 consecutive days | ~40% reduction |
| Archive Instant Access | No access for 90 consecutive days | ~68% reduction |
| Archive Access | No access for 90+ days (opt-in) | ~71% reduction |
| Deep Archive Access | No access for 180+ days (opt-in) | ~95% reduction |

> Objects smaller than 128 KB are stored in Intelligent-Tiering but always billed at Frequent Access rates and are exempt from monitoring charges.

### S3 Cross-Region Replication (CRR)

For disaster recovery or compliance requirements mandating geo-distributed data residency, enable CRR to automatically replicate all new objects to a secondary Region bucket. CRR is configurable at bucket, prefix, or object-tag level.

**Key CRR Use Cases:**

| Use Case | Configuration |
|---|---|
| Regulatory data residency (e.g., GDPR) | Replicate to `eu-central-1` |
| Low-latency reads for regional analytics teams | Replicate to geographically proximate Region |
| Active-active DR with independent compute clusters | Replicate to all active compute Regions |

---

## Kinesis Service Selection Matrix

| Dimension | Kinesis Data Streams | Kinesis Data Firehose | Kinesis Data Analytics |
|---|---|---|---|
| **Management Overhead** | High (custom producer/consumer code required) | Low (fully managed delivery) | Medium (Apache Flink-based) |
| **Latency** | Sub-second (< 1s) | 60 seconds (minimum buffer interval) | Real-time stream processing |
| **Use Case** | Real-time dashboards, anomaly detection, dynamic pricing | Durable delivery to S3/Redshift/OpenSearch | Stream transformation, aggregation, filtering |
| **Code Requirements** | Producer SDK + Consumer SDK | None (direct PUT or stream chaining) | Flink application or SQL |
| **Billing Model** | Per shard-hour + PUT payload unit | Per GB ingested | Per KPU (Kinesis Processing Unit) hour |
| **Selected for This Architecture** | ❌ | ✅ | ❌ |
| **Selection Rationale** | Sub-second latency not required | Convenience-first, no consumer code, operational simplicity | No real-time transformation required |

---

## Athena Query Optimization Guide

> **💡 COST AWARENESS:** Athena charges **$5.00 per TB scanned**. Every optimization below directly reduces operational cost and improves query performance. Apply all three strategies before promoting to production.

### 1. Data Compression

Compress S3 objects before delivery. Compression reduces object size, directly reducing the volume of data Athena must scan per query.

| Format | Compression | Athena Support | Cost Reduction |
|---|---|---|---|
| JSON (raw) | None | ✅ (JsonSerDe) | Baseline |
| JSON (GZIP) | GZIP | ✅ | 30–60% |
| Apache Parquet | Snappy/GZIP | ✅ (native columnar) | 60–90% |
| Apache ORC | Snappy/ZLIB | ✅ (native columnar) | 60–90% |

### 2. Partition Projection

This architecture already implements **Athena Partition Projection** via `TBLPROPERTIES`. Partition projection eliminates the need for `MSCK REPAIR TABLE` by dynamically generating partition metadata from the S3 prefix pattern at query time.

```sql
-- Efficient: Athena scans only the specified hourly partition
SELECT element_clicked, COUNT(*) as click_count
FROM my_ingested_data
WHERE datehour = '2024/01/15/14'
GROUP BY element_clicked
ORDER BY click_count DESC;

-- Inefficient: Full table scan across all partitions
SELECT element_clicked, COUNT(*) as click_count
FROM my_ingested_data
GROUP BY element_clicked;
```

### 3. Columnar Format Conversion

For production scale (>1M events/day), convert JSON files to **Apache Parquet** using AWS Glue ETL. Parquet's columnar storage allows Athena to read only the columns referenced in a query, bypassing unrelated data blocks entirely.

**Benefits of Parquet over JSON for Athena:**
- **Predicate pushdown:** Block-level min/max statistics allow Athena to skip entire row groups
- **Column pruning:** `SELECT element_clicked` reads only the `element_clicked` column bytes
- **Parallelism:** Parquet row groups enable Athena to split reads across parallel workers

---

## Architecture Enhancement Roadmap

The following enhancements have been evaluated and are recommended for production promotion beyond this proof-of-concept:

### Enhancement 1 — Serverless Menu Management (Replace EC2)

**Current State:** Restaurant menu updates are managed via an EC2-hosted application running continuously.

**Recommended State:** Decompose into event-driven serverless components:
```

[Admin UI] → [API Gateway] → [AWS Lambda] → [S3 Static Site Bucket] ↓ [DynamoDB / RDS] (menu item catalog)

```

**Business Value:** Eliminates idle EC2 compute cost; auto-scales to zero between menu update sessions; no OS patching required.

### Enhancement 2 — CloudFront CDN for Static Menu Delivery

**Current State:** End-users fetch HTML menu files directly from S3 (per-request billing).

**Recommended State:** Place Amazon CloudFront in front of the S3 static site origin.

**Benefits:**
- Cached content served from 400+ global edge locations (reduced latency)
- Reduced S3 `GetObject` request volume (reduced cost)
- Built-in DDoS protection via AWS Shield Standard
- Custom domain + ACM certificate support
- AWS WAF integration for request filtering

### Enhancement 3 — Replace API Gateway with Amazon Cognito + AWS SDK Direct Integration

**Current State:** API Gateway proxies HTTPS POST requests from the JavaScript client library to Kinesis Data Firehose.

**Recommended State:** Configure Amazon Cognito Identity Pool to issue temporary AWS credentials to authenticated browser sessions. The JavaScript AWS SDK calls `firehose:PutRecord` directly.

```javascript
// Example: Direct Firehose PutRecord via Cognito-authenticated SDK
const firehose = new AWS.Firehose({ region: 'us-east-1' });

firehose.putRecord({
    DeliveryStreamName: 'PUT-S3-AbCdE',
    Record: {
        Data: JSON.stringify(clickstreamEvent)
    }
}, callback);
```

**Benefits:**
- Eliminates API Gateway hosting cost
- AWS SDK provides built-in exponential backoff retry logic
- Cognito Identity Pool policy can be scoped to `firehose:PutRecord` on a specific stream ARN only

> **⚠️ TRADE-OFF:** Requires refactoring the existing JavaScript client library to integrate the AWS SDK and implement Cognito authentication flow. Evaluate against API Gateway operational cost before committing.

### Enhancement 4 — Infrastructure as Code with AWS CloudFormation

Package the entire stack as a CloudFormation template to enable repeatable, auditable, multi-account deployments for each restaurant tenant.

**Reference CloudFormation snippet (S3 static site with retention policy):**

```yaml
AWSTemplateFormatVersion: 2010-09-09

Resources:
  MenuS3Bucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      AccessControl: PublicRead
      WebsiteConfiguration:
        IndexDocument: index.html
        ErrorDocument: error.html
    DeletionPolicy: Retain    # Protect data assets on stack deletion

  MenuBucketPolicy:
    Type: 'AWS::S3::BucketPolicy'
    Properties:
      Bucket: !Ref MenuS3Bucket
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          - Sid: PublicReadForGetBucketObjects
            Effect: Allow
            Principal: '*'
            Action:
              - 's3:GetObject'
            Resource: !Sub 'arn:aws:s3:::${MenuS3Bucket}/*'

Outputs:
  StaticWebsiteURL:
    Description: Public URL for the static restaurant menu
    Value: !GetAtt MenuS3Bucket.WebsiteURL

  SecureStaticURL:
    Description: HTTPS domain for CloudFront origin configuration
    Value: !Sub 'https://${MenuS3Bucket.DomainName}'
```

### Enhancement 5 — Client-Side Retry Logic with Exponential Backoff

The JavaScript client library must implement retry logic with exponential backoff to handle transient API Gateway or Firehose errors gracefully without flooding the service during degraded availability windows.

```javascript
/**
 * Submits a clickstream event with exponential backoff retry.
 * @param {Object} event - Clickstream event payload
 * @param {number} maxRetries - Maximum retry attempts (default: 5)
 * @param {number} baseDelayMs - Initial delay in milliseconds (default: 100)
 */
async function submitClickstreamEvent(event, maxRetries = 5, baseDelayMs = 100) {
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
        try {
            const response = await fetch(API_ENDPOINT, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(event)
            });

            if (response.ok) return await response.json();

            if (response.status >= 400 && response.status < 500) {
                throw new Error(`Client error ${response.status} — not retryable`);
            }

        } catch (error) {
            if (attempt === maxRetries) throw error;

            // Exponential backoff with jitter
            const delay = baseDelayMs * Math.pow(2, attempt) + Math.random() * 50;
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }
}
```

> **ℹ️ NOTE:** If migrating to the AWS JavaScript SDK (Enhancement 3), exponential backoff is built into the SDK's default retry configuration. Manual implementation is only required when using the raw `fetch` API against API Gateway.

---

## Operational Runbook

### Troubleshooting Decision Tree
```

API Gateway returns non-200 → └─ 400 Bad Request? └─ Check: Firehose delivery stream status = Active? (takes 2-5 min to provision) └─ Check: VTL mapping template saved correctly in Integration Request? └─ Check: DeliveryStreamName in VTL matches exact Firehose stream name? └─ 403 Forbidden? └─ Check: APIGateway-Firehose role has API-Firehose policy attached? └─ Check: Execution role ARN pasted correctly in Method integration settings? └─ 500 Internal Server Error? └─ Check: VTL syntax — look for missing curly braces or unclosed strings └─ Check: CloudWatch Logs for API Gateway execution log errors

Data arrives in API Gateway (200 OK) but not in S3 → └─ Check: Buffer interval — wait 90 seconds after last test event └─ Check: S3 bucket policy — Firehose role ARN correctly specified? └─ Check: Lambda transform-data function — view CloudWatch Logs for errors └─ Check: Firehose monitoring tab in Kinesis console for delivery failure metrics

Athena returns zero results → └─ Check: S3 objects exist in the bucket under datetime-partitioned prefix? └─ Check: LOCATION path in CREATE TABLE matches exact bucket name? └─ Check: storage.location.template datehour format matches S3 prefix structure? └─ Check: Query result S3 location configured in Athena Settings?

QuickSight cannot access Athena data → └─ Check: S3 bucket selected in QuickSight Security & permissions → S3 access? └─ Check: Write permission for Athena Workgroup checkbox selected? └─ Check: SPICE import completed successfully for the dataset?

```

---

## Resource Teardown Procedure

> **⚠️ DATA LOSS WARNING:** The following steps will **permanently delete** all resources and data created in this deployment. Ensure any required data has been exported or archived before proceeding. Steps should be executed in the order presented to respect inter-service dependencies.

Execute teardown in the following sequence:

- [ ] **QuickSight** → Analyses → Delete all analyses created for this POC
- [ ] **QuickSight** → Datasets → Delete `poc-clickstream` dataset
- [ ] **QuickSight** → Account settings → Delete account (if no longer required — subscription billing ceases immediately)
- [ ] **Athena** → Query editor → Execute `DROP TABLE my_ingested_data;`
- [ ] **Athena** → Settings → Manage → Remove query result S3 path
- [ ] **API Gateway** → Select `clickstream-ingest-poc` → Actions → Delete
- [ ] **Kinesis** → Delivery streams → Select stream → Delete (confirm by typing stream name)
- [ ] **Lambda** → Select `transform-data` → Actions → Delete
- [ ] **S3** → Select bucket → Empty bucket (type `permanently delete` to confirm) → Delete bucket
- [ ] **IAM** → Roles → Delete `APIGateway-Firehose`
- [ ] **IAM** → Policies → Delete `API-Firehose`

> **✅ VERIFICATION:** After completing all steps, navigate to the AWS Billing console and confirm no unexpected charges appear for the services listed above in the next billing cycle. If a deletion failure occurs, open an AWS Support ticket referencing the specific resource ARN.

---

*Architecture designed for AWS Solutions Architect — Associate competency alignment. All service selections are justified against stated customer constraints: serverless-first, reduced operational staff, granular pay-per-use billing, JSON-native data format, and sub-minute latency tolerance for analytics workloads.*
```
