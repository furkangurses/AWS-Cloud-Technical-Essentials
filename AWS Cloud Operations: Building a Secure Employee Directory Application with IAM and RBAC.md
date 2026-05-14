# 🛡️ AWS Cloud Operations: IAM & RBAC Implementation Guide

## 📋 Executive Summary

As our cloud infrastructure scales, adhering to the **Principle of Least Privilege (PoLP)** is non-negotiable. This repository contains the Standard Operating Procedure (SOP) for configuring **Role-Based Access Control (RBAC)** within our AWS environment using Identity and Access Management (IAM). 

This guide walks engineers through auditing existing IAM entities, attaching policy-driven groups, and validating permissions across our EC2 compute and S3 storage resources.

---

## 🎯 Engineering Objectives

By completing this operational runbook, the engineer will be able to:

- [x] Audit legacy IAM users and group configurations.
- [x] Inspect AWS Managed Policies vs. Inline Custom Policies.
- [x] Provision access by mapping logical engineering roles to IAM groups.
- [x] Validate cross-service access limitations (S3 vs. EC2) via console testing.
- [x] Execute state-change operations (Start/Stop instances) based on authorized privileges.

---

## 🏗️ RBAC Matrix (Target State)

Below is the target architecture for our IAM hierarchy. This matrix maps the generic lab entities to real-world operational personas.

| IAM User | Corporate Persona | IAM Group | Associated Policy | Permitted Actions |
| :--- | :--- | :--- | :--- | :--- |
| `user-1` | 🗄️ **Data Lake Auditor** | `S3-Support` | `AmazonS3ReadOnlyAccess` (Managed) | `s3:Get*`, `s3:List*` |
| `user-2` | 🛠️ **Tier-1 Cloud Support** | `EC2-Support` | `AmazonEC2ReadOnlyAccess` (Managed) | `ec2:Describe*` |
| `user-3` | 🚀 **Lead Cloud Engineer** | `EC2-Admin` | `EC2-Admin-Policy` (Inline) | `ec2:Describe*`, `ec2:Start*`, `ec2:Stop*` |

---

## 🚀 Execution Runbook

### Phase 1: Environment Audit & Policy Inspection

Before modifying infrastructure, we must inspect the current IAM state.

1. Navigate to the **AWS IAM Dashboard** via the Management Console.
2. Go to **Users** and verify the existence of the service accounts (`user-1`, `user-2`, `user-3`).
   > 🔍 **Audit Check:** Select `user-1`. Navigate to the **Permissions** and **Groups** tabs. You should verify that the user currently has **zero** attached permissions or group memberships.
3. Go to **User groups** and inspect the target groups:
   - **`EC2-Support`:** Inspect the `AmazonEC2ReadOnlyAccess` managed policy. Notice how the JSON structure allows specific `Action` definitions (`cloudwatch:ListMetrics`, `ec2:Describe*`) against any `Resource` (`*`).
   - **`S3-Support`:** Inspect the `AmazonS3ReadOnlyAccess` policy.
   - **`EC2-Admin`:** Inspect the inline policy `EC2-Admin-Policy`. 
   
   *DevOps Pro-Tip: Inline policies maintain a strict 1:1 relationship with the principal entity, ensuring granular control without affecting the broader AWS account.*

---

### Phase 2: Role Assignment & Provisioning

Map the engineers to their respective security groups based on our RBAC Matrix.

#### Step 2.1: Provision Data Lake Auditor (`user-1`)
1. In the IAM Console, navigate to **User groups** -> `S3-Support`.
2. Select **Add users**, check the box for `user-1`, and confirm.

#### Step 2.2: Provision Tier-1 Cloud Support (`user-2`)
1. Navigate to **User groups** -> `EC2-Support`.
2. Select **Add users**, check the box for `user-2`, and confirm.

#### Step 2.3: Provision Lead Cloud Engineer (`user-3`)
1. Navigate to **User groups** -> `EC2-Admin`.
2. Select **Add users**, check the box for `user-3`, and confirm.

> 🛠️ **CLI Alternative (For CI/CD Pipelines):**
> If you were automating this via an infrastructure pipeline, the backend execution would look like this:
> ```bash
> aws iam add-user-to-group --user-name user-1 --group-name S3-Support
> aws iam add-user-to-group --user-name user-2 --group-name EC2-Support
> aws iam add-user-to-group --user-name user-3 --group-name EC2-Admin
> ```

---

### Phase 3: Penetration & Access Validation

We must now verify that our RBAC rules are successfully enforcing Least Privilege. 

**Prerequisite:** Locate your account's IAM Sign-in URL from the IAM Dashboard:
`https://<ACCOUNT_ID>.signin.aws.amazon.com/console`

#### Test 1: S3 Read-Only Validation (`user-1`)
- Open an Incognito/Private browser window.
- Log in as `user-1`.
- Navigate to **S3**. You should successfully see the target `s3bucket`.
- Navigate to **EC2** -> **Instances**. 
- 🛑 **Expected Result:** `You are not authorized to perform this operation.` (The RBAC boundary successfully blocked EC2 access).

#### Test 2: EC2 Read-Only Validation (`user-2`)
- Log out, and log back in as `user-2`.
- Navigate to **EC2** -> **Instances**. You should see the running instances.
- Select an instance, click **Instance state** -> **Stop instance**.
- 🛑 **Expected Result:** An error block. The API request `ec2:StopInstances` was explicitly denied by the Read-Only policy.
- Navigate to **S3**. 
- 🛑 **Expected Result:** Access Denied. S3 bucket enumeration fails.

#### Test 3: EC2 Administrative Validation (`user-3`)
- Log out, and log back in as `user-3`.
- Navigate to **EC2** -> **Instances**.
- Select the target EC2 instance.
- Click **Instance state** -> **Stop instance**.
- ✅ **Expected Result:** The instance state successfully transitions to `Stopping`. The inline administrative policy successfully authorized the `ec2:StopInstances` API call.

---

# AWS Cloud Infrastructure: Employee Directory Application

[![AWS](https://img.shields.io/badge/AWS-Cloud_Infrastructure-232F3E?style=for-the-badge&logo=amazon-aws)](https://aws.amazon.com/)
[![DevOps](https://img.shields.io/badge/DevOps-Security_&_Automation-FF9900?style=for-the-badge&logo=dev-to)](https://aws.amazon.com/iam/)

## 📌 Overview
This repository contains the infrastructure configuration and deployment automation for the **Employee Directory Application**. The project demonstrates a production-grade approach to identity management (IAM), compute provisioning (EC2), and automated application bootstrapping via User Data scripts.

Instead of manual configuration, we utilize **IAM Instance Profiles** to provide temporary, rotating credentials to our application, adhering to the principle of least privilege while interacting with Amazon S3 and DynamoDB.

---

## 🏗 Architecture Components
* **Compute:** Amazon EC2 (t2.micro) running Amazon Linux 2/2023.
* **Identity & Access:** IAM Roles for Service-to-Service communication.
* **Storage:** Amazon S3 (Profile Photos).
* **Database:** Amazon DynamoDB (Employee Metadata).
* **Networking:** Default VPC with custom Security Groups (HTTP/HTTPS ingress).

---

## 🔐 Identity & Access Management (IAM)

### 1. Application Role: `EmployeeWebApp`
To ensure secure programmatic access to AWS APIs without hardcoding credentials, the EC2 instance assumes this role.

| Policy Name | Purpose |
| :--- | :--- |
| `AmazonS3FullAccess` | Allows the app to upload/retrieve employee photos. |
| `AmazonDynamoDBFullAccess` | Allows CRUD operations on the employee database. |

> **Security Note:** In a production environment, these should be narrowed down from `FullAccess` to specific resource ARNs using custom inline policies.

**Trust Relationship:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

```

### 2. Administrative Access

We implement a Group-based permission model for human users:

-   **User:** `EC2Admin`
    
-   **Group:** `EC2Admins`
    
-   **Policy:** `AmazonEC2FullAccess`
    
-   **Access:** AWS Management Console enabled with enforced password rotation on first login.
    

----------

## 🚀 Deployment Workflow

### 1. Network Configuration (Security Group)

The instance is protected by a stateful firewall allowing traffic on the following ports:

-   **Inbound:** Port 80 (HTTP) & Port 443 (HTTPS)
    
-   **Outbound:** All Traffic (Allowing the app to download dependencies).
    

### 2. Provisioning the EC2 Instance

-   **AMI:** Amazon Linux 2 or Amazon Linux 2023.
    
-   **Instance Type:** `t2.micro` (Free Tier Eligible).
    
-   **IAM Instance Profile:** Attached the `EmployeeWebApp` role during launch.
    
-   **User Data:** Automated shell script to configure the environment.
    

----------

## 📜 Automation Scripts (User Data)

Depending on the chosen Amazon Machine Image (AMI), use the corresponding script in the **Advanced Details > User Data** field.

### Option A: Amazon Linux 2023 (Recommended)

Bash

```
#!/bin/bash -ex
# Update and install dependencies
yum -y install python3-pip
wget [https://aws-tc-largeobjects.s3-us-west-2.amazonaws.com/DEV-AWS-MO-GCNv2/FlaskApp.zip](https://aws-tc-largeobjects.s3-us-west-2.amazonaws.com/DEV-AWS-MO-GCNv2/FlaskApp.zip)

# Application Setup
unzip FlaskApp.zip
cd FlaskApp/
pip install -r requirements.txt
yum -y install stress

# Environment Variables
export PHOTOS_BUCKET=${SUB_PHOTOS_BUCKET}
export AWS_DEFAULT_REGION=us-east-1  # Replace with your target region
export DYNAMO_MODE=on

# Start Application
FLASK_APP=application.py /usr/local/bin/flask run --host=0.0.0.0 --port=80 

```

### Option B: Amazon Linux 2 (Legacy)


```
#!/bin/bash -ex
wget [https://aws-tc-largeobjects.s3-us-west-2.amazonaws.com/DEV-AWS-MO-GCNv2/FlaskApp.zip](https://aws-tc-largeobjects.s3-us-west-2.amazonaws.com/DEV-AWS-MO-GCNv2/FlaskApp.zip)

unzip FlaskApp.zip
cd FlaskApp/
yum -y install python3 mysql
pip3 install -r requirements.txt
amazon-linux-extras install epel
yum -y install stress

export PHOTOS_BUCKET=${SUB_PHOTOS_BUCKET}
export AWS_DEFAULT_REGION=us-east-1  # Replace with your target region
export DYNAMO_MODE=on

FLASK_APP=application.py /usr/local/bin/flask run --host=0.0.0.0 --port=80
```


---
