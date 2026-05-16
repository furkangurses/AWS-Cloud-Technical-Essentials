# 🌩️ Employee Directory: High Availability & Auto Scaling Infrastructure

> **Enterprise Infrastructure Documentation**  
> This document outlines the deployment and configuration standards for the High Availability (HA) and Horizontal Scaling architecture of the Employee Directory web application on AWS.

---

## 🏗️ Architecture Overview

To ensure zero downtime and optimal performance during traffic spikes, the application is deployed across multiple Availability Zones (AZs). The architecture leverages an **Application Load Balancer (ALB)** to distribute incoming HTTP traffic and an **Auto Scaling Group (ASG)** to dynamically provision or terminate EC2 compute resources based on CPU utilization.

### Core Components
- **Network Routing:** AWS Application Load Balancer (ALB)
- **Compute Provisioning:** EC2 Launch Templates
- **Elasticity:** EC2 Auto Scaling Group (Target Tracking)
- **Monitoring & Alerting:** Amazon SNS & CloudWatch

---

## 📋 Prerequisites & Environment Setup

Before initiating the deployment, ensure the underlying network and security infrastructure is provisioned:

- [x] **VPC configured** with at least two Public Subnets in different AZs (e.g., `us-west-2a`, `us-west-2b`).
- [x] **Security Groups** provisioned (`LoadBalancerSG` allowing inbound port 80/443).
- [x] **IAM Instance Profile** (`EmployeeDirectoryAppRole`) configured with S3 and DynamoDB access policies.
- [x] **S3 Buckets** prepared for static assets (`IMAGES_BUCKET`) and application code (`INSTALLATION_BUCKET`).

---

## ⚙️ Infrastructure Configuration Parameters

### 1. Target Group & Health Checks (`emp-dir-app-tg`)
The target group routes requests to healthy EC2 instances. Strict health check parameters are defined to quickly cycle out degraded instances.

| Parameter | Value | Description |
| :--- | :--- | :--- |
| **Protocol / Port** | HTTP / 80 | Standard web traffic routing |
| **Target Type** | Instances | EC2 based routing |
| **Healthy Threshold** | 2 | Consecutive checks before marking *Healthy* |
| **Unhealthy Threshold**| 5 | Consecutive checks before marking *Unhealthy* |
| **Timeout** | 20 seconds | Max response time before failure |
| **Interval** | 30 seconds | Time between health probes |

### 2. Auto Scaling Group (`emp-dir-asg`)
Configured to maintain baseline availability while allowing rapid scale-out under load.

| Capacity Metric | Value | Scaling Policy Details |
| :--- | :--- | :--- |
| **Minimum Capacity** | 2 Instances | Ensures multi-AZ redundancy at all times. |
| **Desired Capacity** | 2 Instances | Baseline operational state. |
| **Maximum Capacity** | 4 Instances | Hard cap to control compute costs. |
| **Target Tracking** | 30% CPU | Scale out if average CPU exceeds 30%. |
| **Warmup Period** | 300 seconds | Prevents over-provisioning during boot sequences. |

---

## 🚀 Deployment Strategy

### Phase 1: Load Balancer Initialization
1. Create an Application Load Balancer named `Web-Application-ALB`.
2. Map the ALB to the designated VPC and attach both Public Subnets.
3. Attach the `LoadBalancerSG` security group.
4. Set the default listener action to forward traffic to `emp-dir-app-tg`.

### Phase 2: Compute Image & Launch Template (`emp-dir-lt`)
We utilize **Amazon Linux 2023 (64-bit x86)** on `t3.micro` instances. The environment is bootstrapped dynamically via user-data. 

> **Security Note:** Key pairs are explicitly disabled for these ephemeral instances. All access should be routed through AWS Systems Manager (SSM) Session Manager if debugging is required.

#### Bootstrap Script (`user-data.sh`)
```bash
#!/bin/bash -ex

# -------------------------------------------------------------------
# Environment Variables
# -------------------------------------------------------------------
IMAGES_BUCKET="YOUR_IMAGES_BUCKET_NAME"
INSTALLATION_BUCKET="YOUR_INSTALLATION_BUCKET_NAME"
export DEFAULT_AWS_REGION="us-west-2"
export PHOTOS_BUCKET=$IMAGES_BUCKET
export SHOW_ADMIN_TOOLS=1 # Enables stress testing module

# -------------------------------------------------------------------
# System Updates & Dependencies
# -------------------------------------------------------------------
yum -y update
yum -y install stress unzip

# -------------------------------------------------------------------
# Node.js Runtime Setup (NVM & Node v20)
# -------------------------------------------------------------------
aws s3 cp s3://$INSTALLATION_BUCKET/install.sh - | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
nvm install 20

# -------------------------------------------------------------------
# Application Deployment
# -------------------------------------------------------------------
mkdir -p /var/app
aws s3 cp s3://$INSTALLATION_BUCKET/app.zip /var/app/
cd /var/app/
unzip app.zip

# Initialize and Start
npm install
npm start
```

---

# 🏢 Employee Directory Service - Infrastructure as a Service (IaaS) Setup

> **Infrastructure Documentation & Runbook**
> This document outlines the configuration and deployment steps for achieving High Availability (HA) and automated elasticity for the Employee Directory Microservice using AWS native load balancing and scaling services.

## 🏗️ Architecture Overview

Our application leverages a highly available, fault-tolerant architecture across multiple Availability Zones (AZs) in the `us-west-2` (Oregon) region. The infrastructure utilizes state-of-the-art AWS components:

- **Frontend/Traffic Routing:** AWS Application Load Balancer (ALB)
- **Compute Layer:** Amazon EC2 instances managed by an Auto Scaling Group (ASG)
- **Storage/Database:** Amazon S3 (static assets/configs) & Amazon DynamoDB (NoSQL datastore)
- **Networking:** Custom VPC (`app-vpc`) bridging `us-west-2a` and `us-west-2b`

---

## 📋 Prerequisites

Before proceeding with the infrastructure deployment, ensure the following prerequisites are met:

- [x] AWS Administrator access (or sufficient IAM permissions for EC2, ELB, ASG, S3, and DynamoDB).
- [x] Pre-configured Virtual Private Cloud (`app-vpc`) with minimum two Public Subnets.
- [x] Provisioned S3 bucket and DynamoDB table for the Employee Directory.
- [x] Existing IAM Instance Profile Role attached with necessary permissions.
- [x] Base Application AMI or User Data script ready for bootstrapping.

---

## 🚀 Infrastructure Deployment Guide

### 1. Network & Security Configuration

First, establish the security boundaries for inbound traffic at the load balancer level.

*   **Security Group Name:** `load-balancer-sg`
*   **VPC:** `app-vpc`
*   **Inbound Rules:**
    *   Type: `HTTP`
    *   Port: `80`
    *   Source: `0.0.0.0/0` (Anywhere)
*   **Description:** Allows external HTTP traffic to reach the ALB.

> **Note:** The backend EC2 instances must be assigned a separate security group (`web-security-group`) configured to strictly accept inbound traffic *only* from the `load-balancer-sg`.

### 2. Target Group & Health Check Optimization

Create a Target Group to route requests to one or more registered targets (EC2 instances).

*   **Target Group Name:** `app-target-group`
*   **Target Type:** `Instances`
*   **VPC:** `app-vpc`

**Health Check Configuration:**
To ensure traffic is only routed to healthy nodes, apply the following strict health check parameters:

| Parameter | Value | Description |
| :--- | :--- | :--- |
| **Healthy threshold** | `2` | Consecutive successes to mark target as healthy |
| **Unhealthy threshold** | `5` | Consecutive failures to mark target as unhealthy |
| **Timeout** | `30 seconds` | Time before health check is considered failed |
| **Interval** | `40 seconds` | Frequency of health check execution |

### 3. Application Load Balancer (ALB) Provisioning

The ALB acts as the primary ingress point, distributing incoming application traffic across multiple targets.

*   **ALB Name:** `app-elb`
*   **Scheme:** `Internet-facing`
*   **Network Mapping:** `app-vpc` -> `us-west-2a` & `us-west-2b` (Public Subnets)
*   **Security Groups:** `load-balancer-sg`
*   **Listeners & Routing:** Forward Port 80 traffic to `app-target-group`

### 4. Compute Provisioning (Launch Template)

Define the blueprint for instances launched by the Auto Scaling Group.

*   **Template Name:** `app-launch-template`
*   **Description:** "A web server for the employee directory application."
*   **Instance Type:** `t2.micro` (Optimized for Free-tier / Development)
*   **Network Settings:** Assign `web-security-group`
*   **Advanced Details:**
    *   **IAM Instance Profile:** Attach the pre-configured role for DynamoDB/S3 access.
    *   **User Data (Bootstrap Script):** Inject the bash script to install dependencies, download app source from S3, and map the environment to `us-west-2`.

### 5. Auto Scaling Group (ASG) Setup

Bind the Launch Template to an ASG to automate capacity management based on traffic demands.

*   **ASG Name:** `app-asg`
*   **Launch Template:** `app-launch-template`
*   **VPC & Subnets:** `app-vpc` (Deploy across Public Subnet 1 & Public Subnet 2)
*   **Load Balancing:** Attach to existing target group -> `app-target-group`
*   **Health Checks:** Enable **ELB** health checks (inherits target group health parameters).

**Capacity & Scaling Policies:**

| Capacity Metric | Value | Target Tracking Policy | Value |
| :--- | :--- | :--- | :--- |
| **Desired Capacity** | `2` instances | **Metric Type** | `Average CPU Utilization` |
| **Minimum Capacity** | `2` instances | **Target Value** | `60%` |
| **Maximum Capacity** | `4` instances | **Warmup Time** | `300 seconds` (5 mins) |

---

## 🧪 Validation & Chaos Testing

Once the ASG reports instances as `InService` and the Target Group marks them as `Healthy`, proceed with validation.

### End-to-End Connectivity Check
1. Retrieve the ALB DNS Name from the EC2 Management Console.
2. Navigate to `http://<ALB-DNS-NAME>` in a browser.
3. **Expected Result:** The Employee Directory UI renders correctly, successfully pulling existing data from DynamoDB and S3.

### Load Distribution & Auto-Scaling (Stress Test)
To simulate peak traffic and validate the target tracking policy:
1. Append `/info` to the ALB URL (e.g., `http://<ALB-DNS-NAME>/info`).
2. Refresh the page continuously. You should observe the responding Instance ID and AZ alternating, confirming active cross-zone load balancing.
3. Trigger the internal load generator by clicking the **Stress CPU** option (configured for 10 minutes).
4. **Monitoring:** 
   - Navigate to **Target Groups > Targets** or **Auto Scaling Groups > Activity**.
   - Within a few minutes, as average CPU exceeds 60%, the ASG will trigger a scale-out event.
   - The instance count will seamlessly scale from `2` up to the maximum capacity of `4` to handle the synthetic load.
  
---

# Enterprise Infrastructure Observability: Amazon CloudWatch & Automated Alerting Engine

An production-grade engineering reference guide for implementing end-to-end monitoring, centralized logging, and automated remediation workflows using Amazon CloudWatch and Amazon Simple Notification Service (SNS). This documentation elevates basic monitoring concepts into a highly scalable, site reliability engineering (SRE) compliant design pattern.

---

## 1. Architectural Paradigm: The Observability Framework

To maintain a highly available system, engineers must treat infrastructure monitoring as a continuous feedback loop. Below is an engineering translation of system observability using a **Commercial Kitchen Pipeline** analogy to visualize core DevOps metrics:

| SRE/DevOps Concept | Kitchen System Mapping | Engineering Definition |
| :--- | :--- | :--- |
| **Active Monitoring** | Head Chef Oversight | Continuous telemetry collection from all decoupled layers of the infrastructure footprint. |
| **Metrics** | Appliance Telemetry (e.g., Oven Temp) | Time-series data points containing numeric values with distinct timestamps (`PutMetricData`). |
| **Steady-State Baseline** | Normal Kitchen Velocity | The historical performance footprint used to distinguish normal anomalies from critical performance degradation. |
| **Threshold Alerts** | Temperature Safety Alarms | Automated assertions evaluating metrics against static boundaries over specific observation windows. |
| **Amazon CloudWatch** | Centralized Management Console | The unified engine aggregating metrics, log analytics, and reactive alarm systems. |

---

## 2. Telemetry Ingestion: CloudWatch Metrics Architecture

CloudWatch operates as a multi-dimensional time-series database. Metrics are strictly isolated and identified by structural metadata.


```

[CloudWatch Metrics Engine] │ ├── 📦 Namespace (e.g., AWS/EC2) │ │ │ └── 🏷️ Dimension: InstanceId = i-0123456789abcdef0 │ │ │ └── 📊 Metric: CPUUtilization (Timestamp + Value)

```

### Namespace and Dimension Layout
*   **Namespaces (`AWS/Service` or `Custom/AppName`):** High-level isolation containers. Metrics residing in different namespaces cannot interact or contaminate one another.
*   **Dimensions (Name/Value Pairs):** Unique identifiers that form part of the metric's structural identity. Dimensions allow fine-grained filtering (e.g., querying `CPUUtilization` filtered specifically by `InstanceId=i-08a8f7653b4ad2`).

### Monitoring Tiers: Operational Analysis


```

```
                ┌───────────────────────────────┐
                │      Monitoring Granularity   │
                └───────────────┬───────────────┘
                                │
        ┌───────────────────────┴───────────────────────┐
        ▼                                               ▼

```

┌───────────────────────┐ ┌───────────────────────┐ │ Basic Monitoring │ │ Detailed Monitoring │ │ • Interval: 5 Mins │ │ • Interval: 1 Min │ │ • Cost: Included │ │ • Cost: Additional │ │ • Scope: Standard │ │ • Scope: High-Res │ └───────────────────────┘ └───────────────────────┘

```

> ⚠️ **SRE Warning:** Basic monitoring introduces a 5-minute telemetry lag. For high-velocity auto-scaling groups or production workloads running critical application workloads, **Detailed Monitoring** must be explicitly enabled via Infrastructure as Code (IaC) to prevent delayed auto-scaling or late alerting.

---

## 3. Extending Observability: Custom Metric Integration

Hypervisor-level monitoring (Standard CloudWatch) provides zero visibility into application performance or OS internals like memory consumption and thread exhaustion. If an application hangs but maintains low CPU utilization, basic monitoring will report the system as healthy. 

To achieve a holistic observability posture, engineers must programmatically publish application-level and system-level custom metrics using the AWS CLI or `PutMetricData` API.

### Common Production Production Use-Cases
*   **Ingestion Performance:** Web page load speeds and API gateway latency profiles.
*   **Error Rate Volatility:** Frequency of `HTTP 5xx` and database connection drops.
*   **Operating System Internals:** Available RAM allocation, active disk swap usage, and system thread counts.

### Resolution Topographies
*   **Standard Resolution:** Aggregates and publishes data points at a 1-minute interval.
*   **High-Resolution Metrics:** Offers granular observation windows down to **1-second intervals**, critical for real-time traffic analysis and microsecond latency debugging.

---

## 4. Centralized Log Management Architecture

Amazon CloudWatch Logs acts as an aggregate platform for application, operating system, and security logs. It introduces a rigid structure to parse unstructured textual data streams.

### Log Directory Hierarchy
1.  **Log Event:** The atomic unit of logging. Contains an explicit native timestamp and the corresponding raw event payload message.
2.  **Log Stream:** A chronological sequence of log events originating from a single specific resource or container instance (e.g., a single EC2 host's `auth.log`).
3.  **Log Group:** A logical wrapper organizing multiple Log Streams. Retentions settings, IAM access controls, and encryption keys (KMS) are managed entirely at the **Log Group Level**.


```

📁 Log Group: /aws/production/employee-service │ ├── 📄 Log Stream: host-i-0a1b2c3d_runtime.log ──► [Timestamp] Event: Connection established. └── 📄 Log Stream: host-i-0e5f6g7h_runtime.log ──► [Timestamp] Event: Error 500 - OutOfMemory.

```

### The Unified CloudWatch Agent Pipeline
To ingest logs from EC2 instances or on-premises servers into CloudWatch Logs, the unified CloudWatch Agent must be deployed. It relies on three primary under-the-hood components:

*   **AWS CLI Plugin Execution Unit:** The underlying secure programmatic wrapper pushing log batches using encrypted TLS endpoints.
*   **Initialization Daemon Script:** A lifecycle automation script that boots and configures the monitoring daemon on OS launch.
*   **Cron-Enforced Guardian Process:** A highly resilient scheduling loop that continuously checks the health of the agent daemon, restarting it automatically if it falls over.

---

## 5. Automated Engineering Plan: Infrastructure Alerting Engine

This section details the step-by-step construction of a production-ready dashboard and an automated reactive alarm framework designed to monitor an auto-scaled EC2 compute tier hosting a production service (`production-employee-service`).

### Architectural Strategy
*   **Target Objective:** Monitor CPU Utilization, visualize trends via interactive dashboards, and configure automated notification routines via Amazon SNS.
*   **Alarm State Machine Matrix:**


```

```
              ┌────────────────────────────────┐
              │      CloudWatch Alarm State    │
              └───────────────┬────────────────┘
                              │
     ┌────────────────────────┼────────────────────────┐
     ▼                        ▼                        ▼

```

┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ │ OK │ │ ALARM │ │INSUFFICIENT_DATA│ │ Metric within │ │ Threshold breached│ │ System init / │ │ normal limits │ │ Action triggers│ │ Missing telemetry│ └─────────────────┘ └─────────────────┘ └─────────────────┘

```

### 🛠️ Step-by-Step Production Implementation Guide

#### Phase 1: Operational Visualization Engine (Dashboard Setup)
- [ ] Navigate to the **CloudWatch Management Console**.
- [ ] Select **Dashboards** via the left-side navigation panel and click **Create Dashboard**.
- [ ] Provide an enterprise-compliant naming convention: `Production-Compute-Metrics-Dashboard`.
- [ ] Select **Line Graph Widget** to visualize the historical trend of time-series data.
- [ ] Configure Data Sources: Select **Metrics** $\rightarrow$ **EC2 Namespace** $\rightarrow$ **Per-Instance Metrics**.
- [ ] Search for the specific instance identifier hosting your application stack and select the **CPUUtilization** metric identifier.
- [ ] Set your display aggregation period to `5 Minutes` (or `1 Minute` if detailed monitoring is active) and click **Save Dashboard**.

#### Phase 2: Decoupled Messaging Infrastructure (Amazon SNS Configuration)
- [ ] Navigate to **Amazon Simple Notification Service (SNS)**.
- [ ] Click **Topics** $\rightarrow$ **Create Topic**.
- [ ] Select **Standard Type** and assign an explicit naming convention: `aws-ec2-production-cpu-alerts-sns-topic`.
- [ ] Once created, enter the topic dashboard and click **Create Subscription**.
- [ ] Select Protocol: `Email`.
- [ ] Input the target engineering escalation group or pager address in the Endpoint field: `sre-alerts@enterprise.com`.
- [ ] Click **Create Subscription** and ensure the recipient confirms the subscription via the automated AWS verification email link.

#### Phase 3: Reactive Intelligence System (CloudWatch Alarm Construction)
- [ ] Return to **CloudWatch** $\rightarrow$ **Alarms** $\rightarrow$ **Create Alarm**.
- [ ] Click **Select Metric** $\rightarrow$ **EC2** $\rightarrow$ **Per-Instance Metrics** $\rightarrow$ Select `CPUUtilization` for your production target instance.
- [ ] **Define Conditions:**
    *   **Threshold Type:** `Static`.
    *   **Evaluation Condition:** `Greater than` $\rightarrow$ `70%`.
    *   **Datapoints to Alarm:** `1 out of 1` (evaluating a sustained threshold breach).
    *   **Evaluation Period window:** `5 Minutes` (Mitigates reactive, false-positive paging caused by short-lived, transient operational spikes).
- [ ] **Configure Actions:**
    *   **Alarm State Trigger:** Select `In Alarm`.
    *   **Notification Handling:** Send notification to an existing SNS Topic $\rightarrow$ Select `aws-ec2-production-cpu-alerts-sns-topic`.
- [ ] Give the alarm a clear name: `alarm-ec2-prod-peds-cpu-utilization-high`.
- [ ] Provide an engineering runbook reference within the description: *"Triggers when CPU exceeds 70% for over 5 mins. If this persists, verify application log stream for memory leaks or thread contention."*
- [ ] Click **Create Alarm**.

---

## 6. Advanced Automation: Metric Filtering & Self-Healing Workflows

To achieve a true self-healing cloud architecture, engineers can configure CloudWatch to analyze log content in real time and automatically trigger programmatic remediation tasks without human intervention.


```

┌─────────────────┐ Metric Filter ┌──────────────────┐ Triggers ┌──────────────────┐ │ CloudWatch Logs │─────────────────────►│ Custom Metric │──────────────────►│ CloudWatch Alarm │ │ (Parses 5xx Err)│ │ (Count Volatility)│ │ (State: ALARM) │ └─────────────────┘ └──────────────────┘ └────────┬─────────┘ │ │ Automated ▼ Action ┌─────────────────┐ Executes API ┌──────────────────┐ Invokes ┌──────────────────┐ │ Target Resource │◄─────────────────────│ AWS Lambda │◄──────────────────│ Amazon SNS Topic │ │ (Autoremediates)│ │ (Remediation Code)│ │ (Alert Channel) │ └─────────────────┘ └──────────────────┘ └──────────────────┘

```

### Log-to-Metric Parsing (Metric Filters)
Engineers can define **Metric Filters** to parse raw string data inside a Log Group looking for patterns like `[status_code = 500, ...]`. CloudWatch converts these matches into a numerical metric value that can be graphed and alarmed on. For instance, if more than five `HTTP 500` error logs are parsed within a 1-hour window, the system can transition a custom alarm directly into the `ALARM` state.

### Automated Mitigation Paths
CloudWatch Alarms are not limited to sending notifications to humans. They can trigger automated responses to operational incidents:
*   **EC2 System Actions:** Automatically execute a hardware `Reboot`, `Terminate`, or host `Recover` sequence.
*   **Dynamic Auto-Scaling Policies:** Trigger an Auto Scaling group policy to provision more compute power to handle traffic spikes.
*   **Serverless Remediation (Lambda Pipeline):** Route the alarm state change through an SNS T

---

# AWS Infrastructure Optimization: High Availability & Horizontal Auto-Scaling

This repository serves as an enterprise-grade architectural blueprint and operational runbook for transitioning a single-point-of-failure (SPOF) monolithic application into a highly available (HA), fault-tolerant, and horizontally scalable cloud infrastructure on AWS. 

---

## 1. Architectural Evolution

### Baseline Infrastructure (Legacy Monolith)
Originally, the core production application—an **Enterprise Directory & Identity Service**—was deployed on a single Amazon EC2 instance within a single Availability Zone (AZ). While the persistence layer utilized highly available managed services (**Amazon DynamoDB** for structured data and **Amazon S3** for static assets), the compute tier represented a critical Single Point of Failure (SPOF). If the instance or its hosting AZ degraded, the entire application suffered an outage.

### Target Architecture (High Availability & Elasticity)
To decouple the system and prevent downtime, the infrastructure was refactored into a Multi-AZ, load-balanced, auto-scaling deployment:
* **Redundancy:** Compute nodes are distributed across multiple isolated Availability Zones (Private Subnet A and Private Subnet B).
* **Traffic Management:** An Application Load Balancer (ALB) handles client ingress, abstracts the public IP addresses of backend nodes, and performs continuous layer-7 health checks.
* **Elasticity:** Automated horizontal scaling rules provision and terminate compute capacity dynamically based on workload demands.

---

## 2. High Availability & Uptime Metrics

System availability is calculated as a percentage of uptime within a given year. The table below outlines standard operational tiers versus allowable downtime windows:

| Availability Tier | SLA Notation | Maximum Allowable Downtime (Per Year) |
| :--- | :--- | :--- |
| **90.0%** | "One Nine" | 36.53 days |
| **99.0%** | "Two Nines" | 3.65 days |
| **99.9%** | "Three Nines" | 8.77 hours |
| **99.95%** | "Three and a Half Nines" | 4.38 hours |
| **99.99%** | "Four Nines" | 52.60 minutes |
| **99.995%** | "Four and a Half Nines" | 26.30 minutes |
| **99.999%** | "Five Nines" | 5.26 minutes |

### Strategic Design Trade-offs
> [!IMPORTANT]
> Achieving higher tiers of availability (e.g., "Five Nines") demands significant infrastructure redundancy, multi-region replication, and automated failover capabilities, drastically increasing operational costs. Engineering teams must align SLA targets directly with business revenue requirements to justify cross-zone data replication costs.

### Architectural High Availability Patterns
* **Active-Passive Configuration:** Traffic is routed to a single primary instance. A secondary instance remains on standby. Ideal for legacy, stateful applications where user sessions are bound to local server memory, minimizing synchronization complexities.
* **Active-Active Configuration (Implemented):** Traffic is distributed across all active instances simultaneously. This pattern excels at horizontal scalability but mandates a **stateless application design**, ensuring user session states are offloaded to external distributed layers (e.g., DynamoDB or ElastiCache).

---

## 3. Infrastructure-as-Code & Provisioning Runbook

Follow these production-grade configurations to provision the automated compute tier via the AWS Management Console or AWS CLI equivalents.

### Phase 1: Compute Launch Template Definition
The Launch Template standardizes the baseline configuration for every compute instance spawned by the auto-scaling engine.

1. Navigate to **EC2 Dashboard** > **Launch Templates** > **Create Launch Template**.
2. **Naming & Metadata:**
   * **Name:** `app-launch-template`
   * **Description:** `Production web server template for Enterprise Directory Application.`
   * **Auto Scaling Alignment:** Check the box *Help us guide you in setting up a template for use with Amazon EC2 Auto Scaling*.
3. **Machine Configuration:**
   * **Amazon Machine Image (AMI):** Amazon Linux 2 / Amazon Linux 2023 (Latest LTS)
   * **Instance Type:** `t2.micro` (or production equivalent like `m5.large` depending on baseline requirements)
4. **Networking & Security:**
   * **Security Group:** Select `web-security-group` (configured to accept traffic strictly from the Application Load Balancer via port 80/443).
5. **Advanced Instance Configurations:**
   * **IAM Instance Profile:** Select the application execution role granting necessary permissions to interact with DynamoDB and read from the target S3 bucket.
   * **User Data (Automated Bootstrapping Script):**
     ```bash
     #!/bin/bash
     # Update system packages
     yum update -y
     # Install web server and dependencies
     yum install -y httpd unzip
     systemctl start httpd
     systemctl enable httpd
     # Fetch enterprise artifact from secure storage
     aws s3 cp s3://production-assets-bucket/app-source.zip /tmp/
     unzip /tmp/app-source.zip -d /var/www/html/
     

### Phase 2: Auto Scaling Group (ASG) Configuration

The ASG defines the execution boundaries, target subnets, and integration boundaries for the fleet.

1.  Navigate to **Auto Scaling Groups** > **Create Auto Scaling Group**.
    
2.  **Group Metadata:**
    
    -   **Group Name:** `app-asg`
        
    -   **Launch Template:** Select `app-launch-template`.
        
3.  **Network Topology:**
    
    -   **VPC:** `app-vpc`
        
    -   **Subnets:** Select `Private-Subnet-A` and `Private-Subnet-B` (ensures cross-AZ high availability while keeping instances isolated from direct internet access).
        
4.  **Load Balancing Integration:**
    
    -   **Traffic Routing:** Select _Attach to an existing load balancer_.
        
    -   **Target Group Assignment:** Select `app-target-group`.
        
    -   **Health Check Types:** Enable **ELB (Elastic Load Balancing) Health Checks**. This guarantees that if an instance fails an application-level ping from the load balancer, the ASG will mark it unhealthy and replace it.
        
5.  **Group Size & Sizing Boundaries:**
    
    -   **Minimum Capacity:** `2` (Guarantees baseline high availability across 2 AZs at all times)
        
    -   **Desired Capacity:** `2`
        
    -   **Maximum Capacity:** `4` (Caps total cost while allowing infrastructure to absorb massive traffic surges)
        

### Phase 3: Target Tracking Scaling Policy Definition

Instead of rigid step-scaling manual metrics, a dynamic target tracking policy maintains fleet efficiency.

-   **Policy Type:** Target Tracking Scaling Policy
    
-   **Metric Name:** Average CPU Utilization Across Fleet
    
-   **Target Value:** `60%`
    
-   **Instance Warm-up Cooldown:** `300 seconds`
    

> This configuration dictates that Amazon CloudWatch will continuously calculate the average CPU usage across the fleet. If traffic surges and the average metric breaches 60%, CloudWatch will fire an alarm instructing the ASG to scale horizontally out towards the maximum cap of 4 instances.

----------

## 4. Load Testing & Validation Runbook

To validate the resiliency and automation of the architecture, a programmatic CPU stress test is executed to simulate production spikes (e.g., a flash sale or mass employee login event).

### Execution Steps

1.  Capture the public DNS/URL of the **Application Load Balancer**.
    
2.  Append the health and diagnostics context endpoint to your browser or curl utility: `http://<ALB-DNS-Endpoint>/info`
    
3.  Trigger the integrated programmatic load simulation tool (e.g., via the admin interface or benchmarking tool like ApacheBench):
    
    Bash
    
    ```
    # Alternative CLI simulation executing high compute cycles directly on the endpoints
    ab -n 500000 -c 1000 http://<ALB-DNS-Endpoint>/
    
    ```
    

### Metric Convergence Timeline

   [ Traffic Surge ]
          │
          ▼
┌──────────────────────────────────────┐
│  Fleet Avg CPU Breaches 60% Target   │
└──────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────┐
│ CloudWatch Alarm Enters 'ALARM' State│
└──────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────┐
│ ASG Provisions 2 New EC2 Instances   │ (Total Scale-out: 2 -> 4 Instances)
└──────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────┐
│ ALB Health Checks Pass on New Nodes │
└──────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────┐
│ Load Distributed; Fleet CPU Drops    │ (System Stabilizes Automatically)
└──────────────────────────────────────┘


### Post-Stress Scale-In Validation

Once the load simulation concludes and CPU metrics drop below the target thresholds for a sustained period, the auto-scaling engine automatically executes a **scale-in** routine. Compute nodes are gracefully drained of connections and terminated one-by-one until the fleet hits its minimum boundary conditions (`Desired: 2`), ensuring optimal cost efficiency without human intervention.

----------

## 5. Decommissioning & Infrastructure Clean-up Guardrails

> [!CAUTION] **CRITICAL OPERATIONAL PROCEDURES FOR TEARDOWN:** When tearing down or modifying this infrastructure for maintenance, **do not manually terminate individual EC2 instances**.
> 
> Because the Auto Scaling Group is programmed to maintain a strict minimum capacity of `2` healthy instances, it will treat manual termination as an unexpected infrastructure failure. The ASG's self-healing mechanisms will instantly provision new compute nodes to replace the deleted ones, resulting in unexpected resource utilization and billing.
> 
> **Correct Decommissioning Protocol:**
> 
> 1.  Target the Auto Scaling Group resource block (`app-asg`) directly.
>     
> 2.  Delete the **Auto Scaling Group** configuration first. This automatically terminates all attached child EC2 instances gracefully.
>     
> 3.  Clean up the associated **Launch Template** and **Target Groups** sequentially.

---

# 🏗️ AWS Infrastructure: High Availability & Auto Scaling Architecture

![AWS](https://img.shields.io/badge/AWS-Infrastructure-FF9900?style=for-the-badge&logo=amazonaws)
![Status](https://img.shields.io/badge/Status-Production_Ready-success?style=for-the-badge)
![Tier](https://img.shields.io/badge/Tier-Layer_4_&_7-blue?style=for-the-badge)

> **Abstract:** This documentation outlines the architectural standards, routing mechanisms, and auto-scaling policies for our distributed AWS environment. By leveraging Elastic Load Balancing (ELB) and EC2 Auto Scaling, we ensure fault tolerance, dynamic scalability, and seamless traffic distribution across private subnets.

---

## 📑 Table of Contents
1. [Architecture Overview](#1-architecture-overview)
2. [Elastic Load Balancing (ELB) Strategy](#2-elastic-load-balancing-elb-strategy)
3. [EC2 Auto Scaling (ASG) Configuration](#3-ec2-auto-scaling-asg-configuration)
4. [Scaling Policies & Capacity Management](#4-scaling-policies--capacity-management)
5. [Deployment Prerequisites Checklist](#5-deployment-prerequisites-checklist)

---

## 1. Architecture Overview

To eliminate single points of failure, our workloads are deployed across multiple Availability Zones (AZs) in private subnets. Traffic is securely distributed to these backend nodes using managed load balancers. Our infrastructure strictly adheres to a **stateless, active-active** design pattern, enabling rapid horizontal scaling without application-level refactoring.

### Core Traffic Flow
`Client Request` ➔ `Internet Gateway` ➔ `ELB (Public Subnet)` ➔ `Target Group (EC2 Instances in Private Subnet)`

---

## 2. Elastic Load Balancing (ELB) Strategy

We utilize AWS ELB to abstract node management and automatically handle throughput variances. Based on the OSI model layer requirements of our services, we route traffic using either Application Load Balancers (ALB) or Network Load Balancers (NLB).

### ⚙️ ELB Core Components

- **Listeners:** The client-facing entry point. Bound to a specific port and protocol (e.g., HTTP:80, HTTPS:443).
- **Target Groups:** Logical grouping of backend compute resources (EC2 instances, Lambda functions, or IP addresses).
- **Rules:** The mapping logic (conditions and actions) that forwards traffic from a Listener to a specific Target Group.
- **Health Checks:** Deep application-level monitoring (e.g., `/monitor` endpoint validating DB and S3 connectivity) to ensure traffic is only routed to healthy nodes.

### ⚖️ ALB vs. NLB Specification Matrix

| Feature | Application Load Balancer (ALB) | Network Load Balancer (NLB) |
| :--- | :--- | :--- |
| **OSI Layer** | Layer 7 (Application) | Layer 4 (Transport) |
| **Supported Protocols** | HTTP, HTTPS, gRPC | TCP, UDP, TLS |
| **Routing Intelligence** | Path, Host, Headers, Method, Source IP | Flow Hash (Protocol, IP, Port, Sequence) |
| **TLS Offloading** | Supported (via ACM/IAM) | Supported |
| **Client IP Preservation** | In `X-Forwarded-For` Header | Native Source IP Preservation |
| **Performance Limit** | Scales automatically (requires warmup) | Millions of req/sec (Instantaneous) |
| **IP Allocation** | Dynamic | Supports Static & Elastic IPs |
| **Routing Algorithm** | Round-robin, Least outstanding requests | Flow hash routing |
| **Authentication** | OIDC, SAML, LDAP, AD integrations | Not Supported |

> 💡 **Architectural Note on Connection Draining (Deregistration Delay):** When an instance is flagged for termination (scale-in event), ELB utilizes connection draining to block new requests while allowing active connections to gracefully terminate.

---

## 3. EC2 Auto Scaling (ASG) Configuration

Vertical scaling requires downtime and has hard hardware limits. Our architecture mandates **Horizontal Scaling**, dynamically provisioning or terminating compute instances based on real-time CloudWatch metrics.

### Component Architecture

1. **Launch Templates:** We strictly use Launch Templates (deprecating legacy Launch Configurations). These immutable, version-controlled templates define the instance baseline:
   - AMI ID & Instance Type
   - Security Groups & IAM Instance Profiles
   - EBS Volume configurations
   - SSH Key Pairs & User Data scripts
2. **Auto Scaling Groups (ASG):**
   The logical deployment boundary defining the VPC, spanning at least two private subnets across different AZs for high availability. 

### Capacity Boundaries
For production environments, ASG boundaries are strictly enforced to balance availability and cost:
* `Minimum Capacity`: Baseline required to serve low-traffic periods securely (e.g., `2`).
* `Desired Capacity`: Current active nodes (dynamically adjusted by policies).
* `Maximum Capacity`: Hard ceiling to prevent runaway budget consumption during anomaly spikes.

---

## 4. Scaling Policies & Capacity Management

We avoid traditional static provisioning ("provision for peak"). Instead, we implement dynamic scaling policies based on aggregated telemetry.

### Supported Policy Types

* **Target Tracking Scaling:** (Preferred) Automatically adjusts desired capacity to maintain a specific metric (e.g., keeping average CPU utilization exactly at 65%).
* **Step Scaling:** Triggers based on CloudWatch alarm thresholds with proportional responses (e.g., `+2` instances at 85% CPU, `+4` instances at 95% CPU). Handles continuous alarms gracefully without waiting for a full cooldown.
* **Simple Scaling:** Basic threshold triggers (`+1` instance if CPU > 70%). *Requires strict cooldown period management to prevent over-provisioning during boot sequences.*

### High Availability (Fleet Management) Pattern
For persistent baseline services (like internal monitoring daemons), we configure `Min = Max = Desired`. If a node fails a health check, the ASG automatically terminates and replaces it, ensuring the fleet count remains constant without human intervention.

---

## 5. Deployment Prerequisites Checklist

Before executing the infrastructure as code (IaC) pipelines, ensure the following prerequisites are met:

- [ ] **VPC Readiness:** At least two public subnets (for ELB) and two private subnets (for ASG targets) are provisioned.
- [ ] **Security Groups:** - [ ] ELB SG allows ingress on `80/443` from `0.0.0.0/0` (or restricted CIDRs).
  - [ ] Target EC2 SG allows ingress **only** from the ELB Security Group.
- [ ] **Health Checks:** `/monitor` or `/health` endpoint is implemented in the application layer and returns `HTTP 200`.
- [ ] **State Management:** Application is verified as completely stateless (sessions stored in DynamoDB/ElastiCache, not local memory).
- [ ] **Certificates:** Valid ACM SSL certificate is provisioned if deploying an HTTPS listener.

---

### 💻 Infrastructure Snippet Example (Target Group Definition)

```json
{
  "TargetGroupName": "prod-app-tg",
  "Protocol": "HTTP",
  "Port": 80,
  "VpcId": "vpc-0a1b2c3d4e5f6g7h8",
  "HealthCheckProtocol": "HTTP",
  "HealthCheckPort": "traffic-port",
  "HealthCheckPath": "/api/v1/health",
  "Matcher": {
    "HttpCode": "200"
  },
  "TargetType": "instance"
}
```
