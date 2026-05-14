# 🚀 AWS Infrastructure Setup: EC2 Node.js Web Application

> **Infrastructure Documentation & Deployment Guide**
> This document outlines the network architecture, security configurations, and deployment procedures for provisioning a highly available Node.js web application within an AWS Virtual Private Cloud (VPC).

---

## 🏗 Architecture Overview

This setup utilizes a foundational AWS network topology designed for isolated, secure, and scalable application hosting. 

* **VPC:** Custom VPC network space with DNS resolution enabled.
* **Subnets:** Public subnet mapped to a single Availability Zone.
* **Routing:** Internet Gateway (IGW) for public web traffic and a VPC Gateway Endpoint for secure, internal Amazon S3 access.
* **Compute:** `t3.micro` running Amazon Linux 2023.
* **Storage:** S3 bucket hosting the application artifact (`app.zip`) and dependencies (`npm-cache.tar.gz`).

---

## ✅ Prerequisites

Ensure the following prerequisites are met before beginning the deployment:

- [x] AWS Account with Administrator access.
- [x] Pre-configured IAM Instance Profile (`WebAppInstanceProfile`) with S3 read access.
- [x] S3 Bucket provisioned containing the following artifacts:
  - `app.zip` (Node.js application source code)
  - `npm-cache.tar.gz` (Offline NPM dependency cache)
- [ ] Registered domain name (Optional, for production routing).

---

## 🌐 Network Configuration

### VPC & Subnets

| Resource | CIDR Block | Attributes | Description |
| :--- | :--- | :--- | :--- |
| **VPC** (`WebApp-VPC`) | `10.10.0.0/16` | DNS Resolution: `True` <br> DNS Hostnames: `True` | Core application virtual network. Supports up to 65,536 IP addresses. |
| **Public Subnet 1** | `10.10.1.0/24` | Auto-assign Public IP: `True` | Frontend tier hosting the EC2 instances. |

### Route Table Configurations

The public subnet is associated with a route table that directs outbound traffic appropriately:

| Destination | Target | Purpose |
| :--- | :--- | :--- |
| `10.10.0.0/16` | `local` | Local VPC network communication. |
| `0.0.0.0/0` | `igw-xxxxxxxx` | Internet Gateway for inbound/outbound public web access. |
| `pl-xxxxxxxx` (S3 Prefix) | `vpce-xxxxxxxx` | VPC Gateway Endpoint for secure, private routing to Amazon S3. |

---

## 🛡 Security Groups

The instance is protected by a dedicated Security Group (`WebAppSG`). It enforces the principle of least privilege.

### Inbound Rules (Ingress)
| Protocol | Port Range | Source | Description |
| :--- | :--- | :--- | :--- |
| TCP | `80` | `0.0.0.0/0` | HTTP traffic from the internet. |
| TCP | `443` | `0.0.0.0/0` | HTTPS traffic from the internet. |

### Outbound Rules (Egress)
| Protocol | Port Range | Destination | Description |
| :--- | :--- | :--- | :--- |
| All Traffic | `All` | Security Group (Self) | Internal VPC Endpoint communication. |
| TCP | `443` | S3 Prefix List | Secure outbound requests to AWS S3. |

> ⚠️ **Security Note:** SSH access (Port 22) is intentionally disabled for this deployment to reduce the attack surface. Systems management should be handled via AWS Systems Manager (SSM) Session Manager.

---

## 💻 Compute Provisioning (EC2)

To deploy the application server, launch an EC2 instance with the following specifications:

* **AMI:** Amazon Linux 2023 (`64-bit x86`)
* **Instance Type:** `t3.micro`
* **VPC:** `WebApp-VPC`
* **Subnet:** Public Subnet (`10.10.1.0/24`)
* **Public IP:** Enabled
* **IAM Role:** `LabInstanceProfile` (Required for pulling artifacts from S3)
* **Security Group:** `WebAppSG`

---

## 📜 Bootstrap Script (User Data)

We utilize an automated bootstrap script (User Data) to provision the runtime environment, download artifacts, and start the application upon boot. 

Inject the following script into the **Advanced Details -> User Data** section during instance launch. 

*📌 Note: Replace `YOUR_S3_BUCKET_NAME` with the actual name of your artifact bucket.*

```bash
#!/bin/bash -xe

# 1. Install Node.js runtime
dnf install nodejs24 nodejs24-npm -y

# 2. Download NPM cache from S3 to optimize package installation
aws s3 cp s3://YOUR_S3_BUCKET_NAME/npm-cache.tar.gz /var/cache/npm-cache.tar.gz

# 3. Extract the NPM cache to the root directory
mkdir -p /root/.npm
tar xzf /var/cache/npm-cache.tar.gz -C /root/.npm/

# 4. Download the application source code as a zip artifact
mkdir -p /var/app/
aws s3 cp s3://YOUR_S3_BUCKET_NAME/app.zip /var/app/app.zip

# 5. Extract the application
unzip /var/app/app.zip -d /var/app/

# 6. Install Node dependencies (using the offline cache)
cd /var/app
npm install --offline

# 7. Start the application daemon
npm start
```

---


# AWS Multi-AZ Network Infrastructure & Application Deployment

![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Terraform](https://img.shields.io/badge/terraform-%235835CC.svg?style=for-the-badge&logo=terraform&logoColor=white)
![Infrastructure as Code](https://img.shields.io/badge/Infrastructure_as_Code-Solutions-blue?style=for-the-badge)

## 📋 Project Overview
This project demonstrates the architectural design and implementation of a custom **Virtual Private Cloud (VPC)** on AWS. The goal is to migrate a legacy application from a default VPC to a custom-tailored, high-availability environment spanning multiple **Availability Zones (AZs)**.

This infrastructure provides a secure foundation for the "Employee Directory Application," utilizing public/private subnet isolation, custom routing, and security group hardening.

---

## 🏗️ Architecture Design
The network is designed for high availability and fault tolerance using a **Multi-AZ strategy** within the `us-west-2` region.

### Network Topology
| Component | Specification | Description |
| :--- | :--- | :--- |
| **VPC Name** | `app-vpc` | Isolated network environment |
| **CIDR Block** | `10.1.0.0/16` | Main IP space for the infrastructure |
| **Region** | `us-west-2` | Oregon Region |
| **Availability Zones** | `us-west-2a`, `us-west-2b` | Redundancy across physical locations |

### Subnet Segmentation
| Subnet Name | CIDR | AZ | Type | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `Public-Subnet-1` | `10.1.1.0/24` | `us-west-2a` | **Public** | Load Balancers / Bastion |
| `Private-Subnet-1`| `10.1.2.0/24` | `us-west-2a` | **Private** | App Tier / DB Tier |
| `Public-Subnet-2` | `10.1.3.0/24` | `us-west-2b` | **Public** | Load Balancers / Bastion |
| `Private-Subnet-2`| `10.1.4.0/24` | `us-west-2b` | **Private** | App Tier / DB Tier |

---

## 🚀 Implementation Steps

### 1. Networking Foundation
* **VPC Provisioning:** Created the `app-vpc` with DNS hostnames enabled.
* **Subnetting:** Implemented a 4-subnet architecture to ensure strict isolation between public-facing resources and internal application logic.
* **Internet Gateway (IGW):** Deployed `app-igw` and attached it to the VPC to enable outbound/inbound internet routing.

### 2. Routing Logic
A custom **Public Route Table** (`public-route-table`) was created to manage external traffic flow.
* **Default Route:** `0.0.0.0/0` → `app-igw`.
* **Associations:** Explicitly associated with `Public-Subnet-1` and `Public-Subnet-2`.

> **Engineer's Note:** Subnets are private by default. A subnet becomes "Public" only when it is associated with a route table containing a route to an Internet Gateway.

### 3. Security Hardening
A new **Security Group** was provisioned specifically for the new VPC scope, following the principle of least privilege:
* **Inbound Rules:**
    * `Port 80 (HTTP)`: Allowed from `0.0.0.0/0`
    * `Port 443 (HTTPS)`: Allowed from `0.0.0.0/0`
* **Outbound Rules:**
    * All traffic allowed to ensure the application can fetch updates and reach AWS APIs.

---

## 💻 Application Deployment (EC2)
The **Employee Directory Application** was migrated using an AMI-based deployment strategy.

1.  **AMI Snapshotting:** Leveraged the existing production AMI for consistent software state.
2.  **Compute Instance:** Launched a `t2.micro` instance within `Public-Subnet-1`.
3.  **Networking:**
    * **Auto-assign Public IP:** Enabled for external accessibility.
    * **VPC Placement:** `app-vpc`.
4.  **IAM Role:** Associated existing IAM roles to grant the instance necessary permissions for AWS service interaction.
5.  **User Data:** Reused bootstrapping scripts to ensure the application environment is configured automatically upon boot.

---

## 🔍 Verification & Testing
After deployment, the following checks were performed:
1.  **Reachability:** Verified instance accessibility via public IPv4 address.
2.  **DNS Resolution:** Confirmed that the application handles HTTP requests correctly.
3.  **Security Audit:** Attempted SSH (Port 22) connection to ensure it is blocked as per the new SG policy.

---

## 🛠️ Tech Stack & Tools
* **Cloud Provider:** Amazon Web Services (AWS)
* **Services:** VPC, EC2, IAM, Route 53 (Internet Gateway)
* **OS:** Amazon Linux 2
* **Networking:** IPv4 Subnetting, CIDR Management

---

# AWS Infrastructure & Modernization: From EC2 Monoliths to Containerized Orchestration

## 📌 Executive Summary
This repository contains documentation, provisioning scripts, and architectural patterns for deploying scalable web applications on AWS. It covers the transition from legacy **Elastic Compute Cloud (EC2)** deployments to modern **Container Orchestration** using **Amazon ECS** and **EKS**.

---

## 🛠 Part 1: Provisioning Compute Resources (EC2)

The primary objective is to deploy a **Service Directory Application** using an immutable infrastructure approach. Below is the technical breakdown of the provisioning process.

### 1. Instance Configuration
*   **AMI (Amazon Machine Image):** Utilizing **Amazon Linux 2023**. 
    > **Engineer's Note:** In production environments, we utilize "Golden Images" to ensure security compliance and pre-installed monitoring agents (CloudWatch, CrowdStrike).
*   **Instance Type:** `t2.micro` (Bursting capable). For production-grade workloads, `t3` or `m6i` families are recommended for dedicated performance.
*   **Security Groups (Firewall):** 
    | Protocol | Port | Source | Description |
    | :--- | :--- | :--- | :--- |
    | HTTP | 80 | 0.0.0.0/0 | Public Web Traffic |
    | HTTPS | 443 | 0.0.0.0/0 | Secure Web Traffic |
    | SSH | 22 | Disabled | Access via **Session Manager (SSM)** for better security |

### 2. Identity & Access Management (IAM)
The instance is attached to an **IAM Instance Profile** (`EmployeeWebAppRole`). This follows the **Principle of Least Privilege (PoLP)**, granting the instance temporary security credentials to access:
- **Amazon S3:** For retrieving static assets and application bundles.
- **Amazon DynamoDB:** For NoSQL state management.

### 3. Automated Provisioning (User Data Script)
We use a Bash-based bootstrapping script to handle the runtime environment setup upon first boot.

```bash
#!/bin/bash
# High-Availability Web Server Provisioning Script

# 1. System Update & Dependencies
yum update -y
yum install -y python3-pip wget unzip stress

# 2. Artifact Retrieval & Deployment
wget [https://s3.amazonaws.com/infrastructure-artifacts/app-v1.zip](https://s3.amazonaws.com/infrastructure-artifacts/app-v1.zip)
unzip app-v1.zip
cd app-v1

# 3. Environment Configuration
pip3 install -r requirements.txt
export PHOTOS_BUCKET="prod-asset-store"
export AWS_DEFAULT_REGION="us-west-2"
export DYNAMO_MODE="on"

# 4. Service Launch
python3 application.py --port 80
```


---


# AWS Cloud Engineering: Serverless Architectures & Advanced Networking

This repository serves as a professional technical reference for implementing **Event-Driven Serverless Architectures** and **Cloud Networking Foundations** on AWS. It transitions theoretical course concepts into production-grade engineering practices.

---

## 🏗️ Module 1: Serverless Computing with AWS Lambda

AWS Lambda is a high-scale, event-driven compute service that abstracts underlying infrastructure management. From a DevOps perspective, it represents the pinnacle of operational efficiency by eliminating server maintenance and providing sub-millisecond billing granularity.

### 🧩 Core Characteristics
*   **Ephemeral Lifecycle:** Functions are stateless and scale horizontally in response to triggers.
*   **Managed Runtime:** AWS handles OS patching, capacity provisioning, and high availability.
*   **Execution Limit:** Maximum runtime is **15 minutes** (not suitable for long-running batch jobs or heavy ML training).
*   **Cost Efficiency:** Billed per execution duration and memory allocation; zero cost for idle time.

### 🛠️ Reference Implementation: Automated Media Processing Pipeline
Instead of manual processing, we implement a microservice that automatically handles asset normalization upon ingestion.

#### **Infrastructure Components**
| Component | Responsibility |
| :--- | :--- |
| **Trigger (S3 Event)** | Monitors `PUT` operations in the `assets/input/` prefix. |
| **Compute (Lambda)** | Executes Python 3.9 logic to perform image transformation. |
| **Identity (IAM)** | `LambdaS3FullAccess` role providing scoped permissions to the bucket. |
| **Storage (S3)** | Decoupled storage for raw (`input/`) and processed (`output/`) artifacts. |

#### **Architectural Workflow**
> **Warning:** To prevent **Recursive Invocations** (infinite loops), the Lambda function must write to a different prefix or bucket than the one that triggers it.



1.  **Ingestion:** File uploaded to `s3://media-bucket/input/`.
2.  **Notification:** S3 emits a `s3:ObjectCreated:Put` event.
3.  **Execution:** Lambda environment spins up, pulls the image, and processes it.
4.  **Persistence:** Processed image is saved to `s3://media-bucket/output/`.

---

## 🌐 Module 2: Cloud Networking & VPC Foundations

Networking is the backbone of cloud security. While AWS provides a **Default VPC** for rapid prototyping, production environments require a **Custom VPC** to enforce granular traffic control and isolation.

### 🛡️ Default vs. Custom VPC
*   **Default VPC:** Pre-configured with internet access for all instances. High risk for production databases or sensitive workloads.
*   **Custom VPC:** Built from the ground up. Allows engineers to define public/private subnets, Route Tables, and Network ACLs.

### 🔢 IP Addressing & CIDR Strategy
Effective Cloud Architects must master **Classless Inter-Domain Routing (CIDR)** to prevent address exhaustion and routing overlaps.

#### **Understanding IPv4 Notation**
An IPv4 address is a 32-bit integer, typically represented in four 8-bit octets (Dotted Decimal).
*   **Binary Example:** `11000000.10101000.00000001.00011110`
*   **Decimal Notation:** `192.168.1.30`



#### **CIDR Math for DevOps**
The `/number` suffix indicates the **Routing Prefix** (how many bits are fixed).

| CIDR | Fixed Bits | Flexible Bits | Total IP Addresses | Use Case |
| :--- | :--- | :--- | :--- | :--- |
| `/32` | 32 | 0 | 1 | Single Host/Instance |
| `/28` | 28 | 4 | 16 | Small Subnet (AWS Minimum) |
| `/24` | 24 | 8 | 256 | Standard Subnet |
| `/16` | 16 | 16 | 65,536 | Large VPC (AWS Maximum) |

---

## 🚀 Engineering Best Practices

*   **Security First:** Always use the **Principle of Least Privilege** when assigning IAM roles to Lambda functions.
*   **Cold Starts:** Monitor function concurrency and use "Provisioned Concurrency" for latency-sensitive applications.
*   **Networking Security:** Deploy Lambda functions inside a Private Subnet within a VPC if they need to access RDS databases or internal microservices.
*   **Monitoring:** Use **Amazon CloudWatch** for real-time logging and **AWS X-Ray** for distributed tracing of serverless requests.

---


# AWS Cloud Infrastructure: Virtual Private Cloud (VPC) Design & Implementation

## 1. Executive Summary
This document outlines the architectural design and deployment strategy for a **Virtual Private Cloud (VPC)** environment. The primary objective is to establish a logically isolated network partition within the AWS Cloud to host enterprise-grade applications (e.g., E-commerce platforms, Microservices) with a focus on **High Availability (HA)**, **Security Isolation**, and **Scalable Routing**.

---

## 2. Architectural Overview
The infrastructure is modeled after a multi-tier architecture, ensuring that the presentation layer and data layer are strictly separated through network segmentation.

### Core Components:
*   **Region:** `us-west-2` (Oregon)
*   **Primary CIDR Block:** `10.1.0.0/16` (65,536 Total IP addresses)
*   **Multi-AZ Deployment:** Distributed across `us-west-2a` and `us-west-2b` for fault tolerance.



---

## 3. Network Segmentation (Subnetting Strategy)
We utilize subnets to create granular security boundaries. Each tier is duplicated across two Availability Zones (AZs) to prevent a Single Point of Failure (SPOF).

| Subnet Name | CIDR Block | AZ | Type | Use Case |
| :--- | :--- | :--- | :--- | :--- |
| `sn-public-app-a` | `10.1.1.0/24` | `us-west-2a` | **Public** | ALB, Web Servers, Bastion Hosts |
| `sn-private-db-a` | `10.1.3.0/24` | `us-west-2a` | **Private** | RDS Instances, Internal APIs |
| `sn-public-app-b` | `10.1.2.0/24` | `us-west-2b` | **Public** | Redundant Web/Load Balancing |
| `sn-private-db-b` | `10.1.4.0/24` | `us-west-2b` | **Private** | Database Replication/Failover |

> **Critical Note on IP Allocation:** AWS reserves **5 IP addresses** in every subnet for internal management (Network, VPC Router, DNS, Reserved, and Broadcast). 
> *Example:* A `/24` subnet provides 251 usable IPs instead of 256.

---

## 4. Connectivity & Gateway Services

### 4.1 Internet Gateway (IGW)
The IGW serves as the VPC's edge router. It provides a target in VPC route tables for internet-routable traffic and performs Static Network Address Translation (SNAT) for public instances.
*   **Target:** `igw-app-vpx-gateway`
*   **Attachment:** Bound to `app-vpc`

### 4.2 Virtual Private Gateway (VGW) - Optional
For hybrid-cloud scenarios requiring connectivity to on-premises data centers, a **VGW** can be implemented to establish an encrypted IPsec VPN tunnel. This ensures that internal corporate traffic never traverses the public internet.

---

## 5. Routing Logic & Traffic Flow
Routing is governed by custom route tables associated at the subnet level to control egress and ingress paths.

### 5.1 Public Route Table (`rtb-public`)
Directs traffic destined for the internet (`0.0.0.0/0`) through the IGW.
*   **Destination:** `0.0.0.0/0` -> **Target:** `igw-XXXXX`
*   **Association:** `sn-public-app-a`, `sn-public-app-b`

### 5.2 Private/Main Route Table (`rtb-main`)
The default "Main" route table restricts traffic to **Local Only**. This ensures that databases and internal microservices cannot be reached from the internet, even if they have public IP addresses (unless explicitly routed).
*   **Destination:** `10.1.0.0/16` -> **Target:** `local`

---

## 6. Implementation Workflow (CLI/Console)
1.  **Initialize VPC:** Define the `10.1.0.0/16` CIDR block in the `us-west-2` region.
2.  **Subnetting:** Create public and private tiers across `us-west-2a` and `us-west-2b`.
3.  **IGW Attachment:** Create an Internet Gateway and attach it to the VPC.
4.  **Route Configuration:** 
    *   Create a custom route table for public subnets.
    *   Add a route for `0.0.0.0/0` targeting the IGW.
    *   Associate the route table with the designated public subnets.
5.  **Validation:** Deploy a test EC2 instance in the public subnet to verify internet egress via ICMP or HTTP.

---

## 7. Security Best Practices
*   **Principle of Least Privilege:** Use Private Subnets for all resources that do not require a direct inbound connection.
*   **High Availability:** Always distribute subnets across at least 2 Availability Zones.
*   **Network ACLs vs. Security Groups:** Use Security Groups for stateful instance-level security and NACLs for stateless subnet-level boundaries.

---


# AWS Enterprise Networking & Security: Infrastructure Design

This repository contains the architectural blueprints, routing logic, and security configurations for a production-grade **Amazon Virtual Private Cloud (VPC)**. The design follows the principle of **Defense-in-Depth**, ensuring that application resources (specifically the Employee Directory Application) are isolated, monitored, and resilient.

---

## 1. Core Network Topology

The infrastructure is built on a custom VPC designed to isolate compute resources while allowing controlled internet egress/ingress where necessary. 

### 1.1 Routing Architecture
Routing is managed via **Route Tables**, determining the deterministic flow of traffic between subnets and external gateways.

| Component | Target | Purpose |
| :--- | :--- | :--- |
| **Destination** | `Local` | Internal VPC communication between all subnets. |
| **0.0.0.0/0** | `Internet Gateway (IGW)` | Enables public subnet resources to reach the internet. |
| **0.0.0.0/0** | `Virtual Private Gateway (VGW)` | Routes traffic to on-premises data centers via VPN/Direct Connect. |

> [!IMPORTANT]
> By default, the **Main Route Table** allows all internal traffic. For production environments, **Custom Route Tables** are associated with specific subnets to enforce strict traffic isolation (e.g., preventing a Database subnet from having a route to the IGW).



---

## 2. Security Framework: Defense-in-Depth

We implement security at two distinct layers: the **Subnet Level (NACLs)** and the **Instance Level (Security Groups)**.

### 2.1 Network ACLs (Subnet Perimeter)
NACLs act as a **stateless** firewall. They are the first line of defense at the subnet boundary.

* **Behavior**: Stateless (Return traffic must be explicitly allowed).
* **Default**: Allows all inbound/outbound.
* **Best Practice**: Used for broad "IP Blacklisting" or "CIDR Whitelisting."

**Recommended Web Subnet Inbound Rules:**
| Rule # | Protocol | Port Range | Source | Action | Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 100 | TCP | 443 | 0.0.0.0/0 | ALLOW | HTTPS Production Traffic |
| 130 | TCP | 3389 | 192.0.2.0/24 | ALLOW | RDP from Corporate VPN |
| * | ALL | ALL | 0.0.0.0/0 | DENY | Implicit Deny All |

### 2.2 Security Groups (Instance Firewall)
Security Groups act as a **stateful** firewall for EC2 instances.

* **Behavior**: Stateful (If an inbound request is allowed, the outbound response is automatically tracked and allowed).
* **Default**: Deny all inbound, Allow all outbound.

**Tiered Application Security Model:**
1.  **Web Tier**: Allows `TCP 80/443` from `0.0.0.0/0`.
2.  **App Tier**: Allows `TCP 8080` only from the **Web Tier Security Group**.
3.  **DB Tier**: Allows `TCP 3306/5432` only from the **App Tier Security Group**.



---

## 3. Hybrid Connectivity Solutions

For deployments requiring integration with on-premises data centers, we utilize two primary connectivity models:

### 3.1 AWS VPN (Site-to-Site)
* **Use Case**: Quick setup, encrypted tunnel over the public internet.
* **Components**: Customer Gateway (On-prem) & Virtual Private Gateway (AWS).
* **Suitability**: Low-to-medium traffic volume.

### 3.2 AWS Direct Connect (DX)
* **Use Case**: High-performance, consistent throughput, and reduced latency.
* **Mechanism**: A dedicated physical connection bypassing the public internet.
* **Suitability**: Enterprise workloads, large-scale data migrations, and high-reliability requirements.



---

## 4. Operational Troubleshooting Checklist

If the **Employee Directory Application** is unreachable, DevOps engineers must verify the following layers in order:

- [ ] **Internet Gateway (IGW)**: Is the IGW attached to the VPC?
- [ ] **Route Table (RT)**: Does the subnet have a `0.0.0.0/0 -> IGW` entry?
- [ ] **Security Group (SG)**: Are Inbound rules for Port 80/443 configured for `0.0.0.0/0`?
- [ ] **Network ACL (NACL)**: Are both Inbound (Port 443) and Outbound (Ephemeral Ports 1024-65535) rules active?
- [ ] **Public IP**: Does the EC2 instance have a `Public IPv4` or `Elastic IP` assigned?
- [ ] **Instance Health**:
    -   Check `user-data` execution logs: `/var/log/cloud-init-output.log`.
    -   Verify web server status (Nginx/Apache).
- [ ] **IAM Roles**: Does the EC2 instance profile have permissions for required services (S3/DynamoDB)?

---

## 5. Deployment Methodology
This infrastructure is designed to be deployed via **Infrastructure as Code (Terraform/CloudFormation)** to ensure consistency across Development, Staging, and Production environments.

> "Security is a process, not a product. Networking is the foundation of that process."
