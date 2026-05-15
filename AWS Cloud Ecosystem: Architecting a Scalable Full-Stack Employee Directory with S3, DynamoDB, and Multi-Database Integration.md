# ☁️ AWS Cloud Integration: Employee Directory Web Application

![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Amazon S3](https://img.shields.io/badge/Amazon%20S3-569A31?style=for-the-badge&logo=Amazon-S3&logoColor=white)
![Amazon DynamoDB](https://img.shields.io/badge/Amazon%20DynamoDB-4053D6?style=for-the-badge&logo=Amazon-DynamoDB&logoColor=white)
![Status](https://img.shields.io/badge/Status-Production_Ready-brightgreen?style=for-the-badge)

> **Enterprise Architecture Project:** A scalable, highly available web-based Employee Directory application leveraging AWS native managed services for object storage and NoSQL data management.

## 📋 Project Overview

This repository contains the infrastructure configuration and deployment steps for the **Employee Directory Application**. The architecture modernizes a legacy monolithic structure by decoupling application storage and data layers, migrating them to **Amazon S3** and **Amazon DynamoDB**, respectively. 

The application is hosted on an Amazon EC2 instance within a secure VPC network and interfaces with AWS resources securely using **IAM Roles**, eliminating the need for hardcoded credentials.

---

## 🏗️ Cloud Architecture

The infrastructure consists of the following core components:

- **Compute Layer (EC2):** Hosts the Employee Directory web application inside a Public Subnet.
- **Storage Layer (S3):** Hosts static assets (employee profile photos) securely.
- **Database Layer (DynamoDB):** Serverless NoSQL database managing scalable employee metadata.
- **Security & IAM:** Role-based access control (RBAC) using `EmployeeDirectoryAppRole`.

### Architecture Flow

```text
[ Web Browser ] --> ( HTTP/HTTPS ) --> [ EC2 Instance (App) ]
                                            |
                                            |-- (IAM Role: EmployeeDirectoryAppRole)
                                            |
                                            |--> [ Amazon S3 ] (Profile Photos)
                                            |
                                            |--> [ Amazon DynamoDB ] (Employee Data)

```

----------

## 🛠️ Infrastructure Setup & Deployment Guide

Follow the steps below to provision and configure the necessary AWS services.

### Phase 1: Object Storage Provisioning (Amazon S3)

We utilize S3 to offload binary media files, optimizing our EC2 instance performance.

**1. Create the S3 Bucket**

Create a globally unique bucket to store employee profile pictures.

-   **Bucket Name:** `employee-photo-bucket-<INITIALS>-<UNIQUE_ID>` (e.g., `employee-photo-bucket-jwf-1234`)
    
-   **Region:** Must match your VPC/EC2 deployment region.
    
-   **Public Access:** Keep `Block all public access` **ENABLED**. (Security best practice; the app will fetch via IAM role).
    

**2. Configure S3 Bucket Policy (IAM Integration)**

Attach a restrictive bucket policy to allow the EC2 application role to read objects securely.

> **Security Note:** Replace `INSERT-ACCOUNT-NUMBER` and `INSERT-BUCKET-NAME` with your AWS Account ID and S3 Bucket name.

JSON

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAppRoleS3ReadAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::INSERT-ACCOUNT-NUMBER:role/EmployeeDirectoryAppRole"
      },
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::INSERT-BUCKET-NAME",
        "arn:aws:s3:::INSERT-BUCKET-NAME/*"
      ]
    }
  ]
}

```

**3. Seed S3 Storage**

Upload the baseline employee avatar assets (`.png` files) into the root of your newly created bucket.

----------

### Phase 2: Database Provisioning (Amazon DynamoDB)

Provision a NoSQL table to handle high-throughput, low-latency employee data requests.

**1. Create the DynamoDB Table**

Use the AWS Management Console or AWS CLI to create the table. Ensure exact casing.

**Configuration**

**Value**

**Data Type**

**Table Name**

`Employees`

N/A

**Partition Key**

`id`

`String`

**Capacity Mode**

Provisioned/On-Demand

N/A

### 📊 DynamoDB Schema Definition

Unlike relational databases, DynamoDB is schemaless. However, the application expects the following JSON document structure for each item:

JSON

```
{
  "id": "uuid-or-unique-string",   // Partition Key
  "name": "John Doe",              // String
  "location": "New York",          // String
  "email": "john.doe@example.org", // String
  "photo": "john_avatar.png"       // String (Matches exactly to S3 object key)
}

```

----------

### Phase 3: Application Configuration

Link the configured AWS backing services to the front-end application.

1.  Navigate to the Employee Directory **Web Application URL**.
    
2.  Go to **Administration > Configuration**.
    
3.  Locate the **S3 Bucket** field and input your bucket name (`employee-photo-bucket-INITIALS-NUM`).
    
4.  Save the configuration. The UI should now report:
    
    -   `S3 Access Enabled: True`
        
    -   `DynamoDB Status: Connected`
        

----------

## ✅ Deployment Validation & QA Checklist

Use this checklist to verify infrastructure integrity before marking the deployment as production-ready:

-   [ ] **S3 Configuration:** Bucket is created, private, and seeded with `.png` files.
    
-   [ ] **IAM Security:** Bucket policy successfully attached to `EmployeeDirectoryAppRole`.
    
-   [ ] **Database Provisioning:** `Employees` table is `Active` with `id` as the string partition key.
    
-   [ ] **Application Link:** Web App Configuration panel reflects the correct S3 bucket.
    
-   [ ] **End-to-End Test (Create):** Create a new employee record via the Web UI and verify it appears in the DynamoDB console.
    
-   [ ] **End-to-End Test (Update):** Modify a location or email in the DynamoDB console (JSON/Form view) and verify changes propagate to the live web interface.
    

----------

## 🔧 Troubleshooting Operations

**Issue: S3 Access Denied or Images not loading**

> _Resolution:_ Verify that there are no leading spaces before the first `{` in your JSON bucket policy. Ensure the `AWSAccountID` and `Bucket Name` exactly match your environment ARN structure.

**Issue: Cannot modify the 'id' field in DynamoDB**

> _Resolution:_ The `id` acts as the primary Partition Key and is immutable. To change an ID, you must issue a `Delete` operation followed by a `PutItem` operation with the new key.

---

# AWS Full-Stack Integration: Compute, Storage, and NoSQL Persistence

## 📌 Project Overview
This repository documents the deployment and integration of a high-availability **Employee Directory Application**. The objective is to transition from a localized state to a fully decoupled architecture utilizing **Amazon EC2** for compute, **Amazon S3** for binary object storage (images), and **Amazon DynamoDB** for NoSQL data persistence.

---

## 🛠 Tech Stack & Architecture
| Component | Service | Role |
| :--- | :--- | :--- |
| **Compute** | Amazon EC2 | Hosts the Python/Node.js application logic. |
| **Storage** | Amazon S3 | Stores employee profile pictures (Blob Storage). |
| **Database** | Amazon DynamoDB | Stores employee metadata and S3 object references. |
| **Security** | IAM Roles | Provides fine-grained access to S3 and DynamoDB without hardcoded credentials. |

---

## 🚀 Execution Workflow

### 1. Compute Provisioning (Cloning Strategy)
To ensure environment parity, we utilize the "Launch More Like This" strategy from a validated S3-enabled baseline.

*   **Instance Baseline:** `employee-directory-app-s3` (v2.0)
*   **Identification Tag:** `employee-directory-app-dynamodb`
*   **Networking:** 
    *   **Auto-assign Public IP:** `Enabled` (Crucial for external validation).
    *   **IAM Role:** Ensure the `EC2-S3-DynamoDB-FullAccess` (or equivalent) role is attached.
*   **Verification:** Wait for `2/2 Status Checks` before proceeding to the application layer.

### 2. NoSQL Data Layer Configuration
We deploy a serverless DynamoDB table designed for high-concurrency access and schema flexibility.

| Attribute | Value | Description |
| :--- | :--- | :--- |
| **Table Name** | `Employees` | Case-sensitive identifier used by the application logic. |
| **Partition Key** | `id` (String) | Unique identifier for the employee record. |
| **Capacity Mode** | On-Demand/Default | Optimized for fluctuating traffic patterns. |

> **DevOps Pro-Tip:** In a production environment, always enable **Point-in-Time Recovery (PITR)** and use **Tags** (e.g., `Environment: Production`, `Owner: DevOps`) for cost allocation and disaster recovery.

---

## 🧪 Integration Testing & Validation

### Step 1: Frontend Accessibility
Retrieve the **Public IPv4** from the EC2 Dashboard and access the application via a web browser.
`http://<EC2_PUBLIC_IP>`

### Step 2: Data Injection
Submit a new entry via the application UI to trigger the following internal events:
1.  **S3 Upload:** The application pushes the image file to the designated S3 bucket.
2.  **DynamoDB PutItem:** The application writes the metadata (Name, Job Title, etc.) and the S3 Object Key to the `Employees` table.

### Step 3: Persistence Verification
| Layer | Verification Method | Expected Outcome |
| :--- | :--- | :--- |
| **Storage (S3)** | S3 Management Console | Presence of the new `.jpg`/`.png` object with a unique UUID. |
| **Database (NoSQL)** | DynamoDB -> Explore Items | New record containing the `id`, `name`, `location`, and `badge_list`. |

---

## 🧹 Resource Cleanup & Cost Optimization
To prevent unnecessary AWS billing (Elastic IP and Compute charges):

1.  Navigate to **EC2 Instances**.
2.  Select `employee-directory-app-dynamodb`.
3.  Action: **Instance State -> Stop** (Keep the EBS volume and DynamoDB table for the next development phase).

---

## 📜 Documentation & Standards
*   **Naming Conventions:** All resources follow the `<Project>-<Service>-<Environment>` format.
*   **Security:** This deployment adheres to the **Principle of Least Privilege (PoLP)** by using IAM Instance Profiles instead of Access Keys.

---


# AWS Infrastructure: Scalable Object Storage with Amazon S3

## 📌 Executive Summary
This documentation covers the architectural implementation, security protocols, and lifecycle management of **Amazon Simple Storage Service (S3)**. Unlike Block Storage (EBS), S3 provides a highly scalable, distributed object storage layer designed for "storage for the internet," offering industry-leading durability and availability for unstructured data.

---

## 🏗️ Architecture Overview

### 1. Object Storage vs. Block Storage
In high-scale DevOps environments, **Amazon EBS** (Block Storage) often hits limitations regarding multi-attach capabilities and volume size. **Amazon S3** solves this by decoupling storage from compute.

| Feature | Amazon EBS | Amazon S3 |
| :--- | :--- | :--- |
| **Type** | Block Storage | Object Storage |
| **Access** | Attached to EC2 (via NVMe/SCSI) | Web-based API/URL |
| **Scalability** | Manual resizing required | Virtually infinite |
| **Structure** | Hierarchical (File System) | Flat (Key-Value) |

### 2. Core Components
*   **Buckets:** Fundamental containers for data. They are **Region-specific** but exist within a **Global Namespace**.
    *   *Constraint:* Names must be globally unique and DNS-compliant.
*   **Objects:** The data units. Each object consists of:
    *   **Data:** The actual file (up to 5TB).
    *   **Metadata:** Key-value pairs describing the object.
    *   **Object Key:** The unique identifier (path) within the bucket.
*   **Durability & Availability:** Designed for **99.999999999% (11 9's)** of durability and **99.99%** availability by distributing data across a minimum of three Availability Zones (AZs).

---

## 🔒 Security & Access Control

By default, all S3 resources are **private**. Unauthorized access is denied unless explicitly granted.

### 1. Access Management Strategies
*   **IAM Policies:** Attached to Users, Groups, or Roles. Used for centralized permission management.
*   **Bucket Policies:** Attached directly to the bucket. Used for cross-account access or defining public read-only permissions.
*   **Access Control Lists (ACLs):** A legacy method for managing granular object-level access (generally superseded by Bucket Policies).

### 2. Infrastructure Hardening: Public Access Block
AWS implements **Block Public Access (BPA)** at the account and bucket level to prevent accidental data exposure. To make an asset public, one must:
1. Disable "Block all public access."
2. Enable ACLs (if using object-level permissions).
3. Explicitly apply the `public-read` permission.

### 3. Encryption
*   **Encryption at Rest:**
    *   **Server-Side Encryption (SSE):** S3 handles encryption before writing to disk (SSE-S3, SSE-KMS, or SSE-C).
    *   **Client-Side Encryption:** Data is encrypted before transit to AWS.
*   **Encryption in Transit:** Enforced via **TLS (HTTPS)**.

---

## 🛠️ Data Integrity & Versioning

To prevent accidental overwrites or deletions, **S3 Versioning** should be implemented.
*   **Stateful Tracking:** Each object version receives a unique Version ID.
*   **Delete Markers:** Deleting an object creates a marker rather than removing the data. Reverting is as simple as deleting the marker.
*   **Bucket States:** `Unversioned` (Default), `Versioning-Enabled`, or `Versioning-Suspended`.

---

## 💰 Cost Optimization: Storage Classes

A professional Cloud Engineer optimizes for cost based on data access patterns.

| Storage Class | Use Case | Retention / Access |
| :--- | :--- | :--- |
| **S3 Standard** | Active, frequently accessed data | Millisecond access |
| **S3 Intelligent-Tiering** | Unknown or changing access patterns | Auto-shifts between tiers |
| **S3 Standard-IA** | Long-lived, infrequently accessed | Lower storage cost, retrieval fee |
| **S3 One Zone-IA** | Non-critical, reproducible data | Single AZ (Cheaper) |
| **S3 Glacier Instant** | Archival data needing millisecond access | Long-term Archive |
| **S3 Glacier Deep Archive** | Compliance/Regulatory data (Years) | 12-hour retrieval; Lowest cost |

---

## ⚙️ Automation: S3 Lifecycle Management

To reduce manual overhead, **Lifecycle Policies** automate transitions between storage classes.

```json
{
  "Rules": [
    {
      "ID": "MoveToArchive",
      "Prefix": "logs/",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 365
      }
    }
  ]
}
```

---


# AWS Cloud Infrastructure Integration: Object Storage & Relational Database Implementation

## 📌 Project Overview
This project demonstrates the implementation of a scalable, decoupled cloud architecture using **Amazon Web Services (AWS)**. The primary objective is to transition a monolithic application into a cloud-native state by integrating **Amazon S3** for persistent object storage and **Amazon RDS** for managed relational data management.

The architecture emphasizes **High Availability (HA)**, **Security via IAM Policies**, and **Operational Excellence** through managed services.

---

## 🏗 Architecture Components

### 1. Object Storage Layer (Amazon S3)
A centralized repository for application assets (e.g., employee media, documents).
- **Security:** Implemented via Bucket Policies to restrict access to specific IAM roles rather than public exposure.
- **Integration:** Application-level connectivity via environment variables and User Data scripts.

### 2. Compute Layer (Amazon EC2)
Application servers deployed within a VPC.
- **Provisioning:** Utilizing "Launch More Like This" (Cloning) to maintain configuration consistency.
- **Bootstrap:** User Data scripts used for automated runtime configuration.

### 3. Database Layer (Amazon RDS)
A managed MySQL environment providing relational data persistence.
- **Engine Selection:** MySQL (Standard) for cost-efficiency or Amazon Aurora for high-performance requirements.
- **Resilience:** Multi-AZ deployments for automated failover and zero-downtime maintenance.

---

## 🚀 Implementation Walkthrough

### Phase 1: S3 Bucket Provisioning & Security
To ensure least-privilege access, we define a specific bucket policy allowing our application role to interact with objects.

1. **Create Bucket:** `corp-asset-storage-v1-[suffix]`
2. **Region:** `us-west-2` (Ensuring parity with compute resources).
3. **Bucket Policy Deployment:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::YOUR_ACCOUNT_ID:role/AppExecutionRole"
            },
            "Action": [
                "s3:PutObject",
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::corp-asset-storage-v1-[suffix]/*"
        }
    ]
}

```

### Phase 2: Compute Instance Refinement

We clone the existing application server and inject the S3 configuration via **EC2 User Data**.

-   **Workflow:** `EC2 Console` -> `Instance Actions` -> `Image and Templates` -> `Launch More Like This`.
    
-   **Instance Name:** `production-app-s3-enabled`
    
-   **User Data Script:**
    

Bash

```
#!/bin/bash
# Inject S3 Bucket Name for App Configuration
echo "S3_BUCKET_NAME=corp-asset-storage-v1-963" >> /etc/environment
# Restart Application Service
systemctl restart app-service

```


----------

## 🛡 Security & High Availability (HA)

### Network Security

-   **Private Subnets:** RDS instances are deployed in private subnets with no direct IGW (Internet Gateway) route.
    
-   **Security Groups:** - **App-SG:** Allows Port 80/443 from ELB.
    
    -   **DB-SG:** Allows Port 3306 **only** from App-SG.
        

### Data Resiliency

> [!IMPORTANT]
> 
> **Multi-AZ Deployment:** In the event of an Availability Zone failure, RDS automatically flips the DNS CNAME record to the standby instance. No application code changes are required as the endpoint remains identical.

-   **Automated Backups:** 0-35 day retention (Point-in-Time Recovery).
    
-   **Manual Snapshots:** Used for long-term compliance and baseline versioning.
    

----------

## 🛠 Operational Checklist

-   [x] S3 Bucket created and Policy attached.
    
-   [x] IAM Role associated with EC2 for S3 API access.
    
-   [x] RDS Instance status: `Available`.
    
-   [x] Security Group rules verified (Port 3306 connectivity).
    
-   [x] Public IP verified for the new App Instance.

---


# AWS Cloud Infrastructure: Amazon DynamoDB Deep Dive

## Executive Summary
This documentation explores **Amazon DynamoDB**, a fully managed, serverless NoSQL database service designed for high-performance applications. Unlike traditional relational databases (RDBMS), DynamoDB provides seamless horizontal scaling and millisecond latency, making it the industry standard for high-traffic e-commerce, gaming, and microservices architectures.

---

## 1. Core Architecture & Philosophy
DynamoDB is built on the principle of **Serverless Data Management**. It eliminates the operational overhead of provisioning instances, patching software, or managing underlying storage.

### Key Characteristics:
*   **Serverless:** No infrastructure to manage; automatic scaling based on traffic.
*   **Multi-AZ Redundancy:** Data is synchronously replicated across multiple Availability Zones (AZs) for high availability and durability.
*   **Predictable Performance:** Maintains single-digit millisecond response times regardless of the data volume (from 10 items to 10 billion items).
*   **SSD-Backed:** All data is stored on Solid State Drives for high-speed I/O.



---

## 2. Technical Components
To operate DynamoDB effectively, it is essential to understand its data hierarchy:

| Component | Description | Equivalent in RDBMS |
| :--- | :--- | :--- |
| **Table** | A collection of data items. | Table |
| **Item** | A group of attributes, uniquely identifiable. | Row / Record |
| **Attribute** | A fundamental data element (e.g., String, Number). | Column / Field |

### Primary Keys & Indexing
Each item in a table is uniquely identified by a **Primary Key**. 
1.  **Partition Key (Simple Primary Key):** Uses a single attribute (e.g., `EmployeeID`) to determine the internal partition where data is stored.
2.  **Composite Primary Key:** Combines a Partition Key and a **Sort Key** (e.g., `DepartmentID` + `EmployeeID`) for more complex querying patterns.

---

## 3. Relational (SQL) vs. Non-Relational (NoSQL)
In modern DevOps and Cloud engineering, choosing the right database depends on the access pattern.

### Relational Database (e.g., Amazon RDS / Aurora)
*   **Schema:** Rigid and predefined.
*   **Relationships:** Complex joins across multiple tables.
*   **Scaling:** Primarily vertical (larger instances); horizontal scaling is complex.
*   **Use Case:** ERP, CRM, and complex financial transactions.

### NoSQL Database (Amazon DynamoDB)
*   **Schema:** Flexible; items in the same table can have different attributes.
*   **Relationships:** Focused on flat data structures and single-table design.
*   **Scaling:** Native horizontal scaling through partitioning.
*   **Use Case:** Session management, IoT telemetry, high-speed lookups, and mobile backends.



---

## 4. AWS Purpose-Built Database Portfolio
Modern architectures move away from "one-size-fits-all" databases. AWS provides specialized engines for specific workloads:

| Database Type | Industry Use Cases | Recommended AWS Service |
| :--- | :--- | :--- |
| **Relational** | Traditional Web Apps, ERP, E-commerce | **Amazon RDS, Aurora** |
| **Key-Value** | High-traffic web, real-time bidding, gaming | **Amazon DynamoDB** |
| **In-Memory** | Caching, Leaderboards, Session stores | **ElastiCache (Redis/Memcached)** |
| **Document** | Content Management, User Profiles | **Amazon DocumentDB** |
| **Graph** | Fraud Detection, Social Networks | **Amazon Neptune** |
| **Time-Series** | IoT Telemetry, DevOps Monitoring | **Amazon Timestream** |
| **Ledger** | Banking Transactions, Supply Chain | **Amazon QLDB** |



---

## 5. Engineering Implementation: Employee Metadata Service
In this scenario, we replace a legacy RDS instance with a DynamoDB table to handle a global employee lookup service.

### Infrastructure Strategy:
> **Task:** Create a high-availability lookup table for employee metadata.
> **Constraint:** The application requires a table named `Employees` with a unique identifier `EmployeeID`.

### Deployment Steps:
1.  **Table Creation:** Define the table name as `Employees`.
2.  **Schema Definition:** Set `EmployeeID` (String) as the **Partition Key**.
3.  **Capacity Mode:** Choose between **On-Demand** (for unpredictable workloads) or **Provisioned** (for cost-optimized, steady-state workloads).
4.  **Verification:** Use the AWS CLI or Console to perform a `Scan` or `Query` operation to verify data ingestion.

```bash
# Example AWS CLI command to put an item into the table
aws dynamodb put-item \
    --table-name Employees \
    --item '{
        "EmployeeID": {"S": "ENG-1024"},
        "FullName": {"S": "John Doe"},
        "Role": {"S": "DevOps Engineer"},
        "Department": {"S": "Cloud Infrastructure"}
    }'
```
