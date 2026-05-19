# Enterprise Hybrid Cloud Infrastructure — AWS Reference Architecture

[![Infrastructure CI](https://img.shields.io/badge/Infrastructure_CI-Passing-brightgreen?style=flat-square&logo=githubactions)](https://github.com)
[![Security Scan](https://img.shields.io/badge/Security_Scan-Clean-brightgreen?style=flat-square&logo=springsecurity)](https://github.com)
[![Compliance](https://img.shields.io/badge/Compliance-SOC2%20%7C%20PCI--DSS%20%7C%20HIPAA-blue?style=flat-square&logo=shieldsdotio)](https://github.com)
[![IaC Coverage](https://img.shields.io/badge/IaC_Coverage-CloudFormation%20%7C%20CDK-orange?style=flat-square&logo=amazonaws)](https://github.com)
[![DR Strategy](https://img.shields.io/badge/DR_Strategy-Warm_Standby-yellow?style=flat-square&logo=amazonaws)](https://github.com)
[![License](https://img.shields.io/badge/License-Enterprise_Proprietary-red?style=flat-square)](https://github.com)

---

> **⚠️ PRODUCTION-CRITICAL DOCUMENT:** This repository defines the authoritative reference architecture for a regulated, enterprise-grade hybrid cloud deployment bridging on-premises data center infrastructure with AWS. All configuration decisions documented herein have direct impact on **availability SLAs**, **data residency compliance**, and **RPO/RTO commitments**. Changes must pass the full Architecture Review Board (ARB) process before implementation.

---

## Table of Contents

- [Executive Architecture Summary](#executive-architecture-summary)
- [Business Requirements & Constraints](#business-requirements--constraints)
- [High-Level Architecture Diagram](#high-level-architecture-diagram)
- [Core Design Principles](#core-design-principles)
- [Infrastructure Components](#infrastructure-components)
  - [Network Connectivity Layer](#1-network-connectivity-layer)
  - [Compute & Container Orchestration](#2-compute--container-orchestration)
  - [Relational Database Platform](#3-relational-database-platform)
  - [Hybrid Storage Platform](#4-hybrid-storage-platform)
  - [Operational Management Plane](#5-operational-management-plane)
  - [Data Protection & Backup](#6-data-protection--backup)
- [Database Migration Runbook](#database-migration-runbook)
- [Disaster Recovery Strategy](#disaster-recovery-strategy)
- [Architecture Optimizations & Scaling](#architecture-optimizations--scaling)
- [Security Posture](#security-posture)
- [Environment Configuration Reference](#environment-configuration-reference)
- [Implementation Checklist](#implementation-checklist)
- [Operational Runbooks](#operational-runbooks)
- [Further Reading & Official References](#further-reading--official-references)

---

## Executive Architecture Summary

This document describes the production hybrid cloud architecture for an enterprise insurance-sector workload migrating internal applications and stateful data platforms from a self-managed on-premises data center to AWS. The architecture is designed around three non-negotiable operational mandates:

1. **Resilience & Fault Tolerance** — No single point of failure is acceptable. All stateful and stateless tiers are distributed across a minimum of two AWS Availability Zones.
2. **Consistent Operational Tooling** — The same management, observability, and orchestration toolchain must span both on-premises and AWS-hosted resources to eliminate operational fragmentation.
3. **Compliance-Grade Network Isolation** — All data in transit between the corporate data center and AWS must traverse a private, dedicated network fabric. Public internet exposure for workload traffic is explicitly prohibited.

The architecture operates within a **single AWS Account**, a **single AWS VPC**, and a **single AWS Region** at initial launch, with DR and multi-Region extensions documented in the [Disaster Recovery Strategy](#disaster-recovery-strategy) section.

---

## Business Requirements & Constraints

| Requirement | Category | Architectural Decision |
|---|---|---|
| High-volume, low-latency data transfer between DC and AWS | Network | AWS Direct Connect (dedicated connection) |
| No public internet exposure for workload traffic | Security / Network | Private subnets + NAT Gateway egress-only |
| PostgreSQL database migration with zero downtime | Database | AWS DMS (homogeneous migration) to Amazon RDS |
| High availability for the production database | Database | Amazon RDS Multi-AZ deployment |
| NFS-based file access for on-premises applications | Storage | AWS Storage Gateway — S3 File Gateway |
| Low-latency local writes for on-premises file producers | Storage | Storage Gateway local cache (on-premises appliance) |
| Automatic storage cost optimization via tiering | Storage | S3 Lifecycle Policies + S3 Intelligent-Tiering |
| Uniform container orchestration across DC and AWS | Compute | Amazon ECS + Amazon ECS Anywhere |
| No on-premises application refactoring | Application | Protocol-compatible services (NFS via Storage Gateway) |
| Centralized operational management (patching, scripts, params) | Operations | AWS Systems Manager (SSM Agent on all managed nodes) |
| Centralized backup management across all environments | Data Protection | AWS Backup |
| Internal applications only — no internet ingress required | Network | Private subnets (larger allocation) over public subnets |

---

## High-Level Architecture Diagram
```

┌──────────────────────────────────────────────────────────────────────────────────────┐ │ CORPORATE DATA CENTER │ │ │ │ ┌──────────────────┐ ┌─────────────────────┐ ┌──────────────────────────────┐ │ │ │ On-Prem ECS │ │ Storage Gateway │ │ On-Prem VMs / Servers │ │ │ │ (ECS Anywhere) │ │ S3 File Gateway │ │ (SSM Agent Installed) │ │ │ │ + SSM Agent │ │ NFS / Local Cache │ │ │ │ │ └────────┬─────────┘ └──────────┬──────────┘ └──────────────────────────────┘ │ │ │ │ │ │ └────────────────────────┼───────────────────────────┐ │ │ │ │ │ │ ┌─────────▼──────────┐ │ │ │ │ Customer Router │ │ │ │ └─────────┬──────────┘ │ │ └────────────────────────────────────┼───────────────────────────┼──────────────────────┘ │ │ ┌──────────▼──────────┐ ┌────────────▼──────────────┐ │ AWS Direct Connect │ │ AWS Site-to-Site VPN │ │ (Primary Path) │ │ (Failover Path - IPSec) │ └──────────┬──────────┘ └────────────┬──────────────┘ │ │ └────────────┬───────────────┘ │ ┌─────────────────────────────────────────────────▼────────────────────────────────────┐ │ AWS REGION (e.g., us-east-1) │ │ │ │ ┌─────────────────────────────────────────────────────────────────────────────────┐ │ │ │ VPC (10.x.x.x/16) │ │ │ │ │ │ │ │ ┌──────────────────────────────┐ ┌──────────────────────────────────────┐ │ │ │ │ │ Availability Zone A │ │ Availability Zone B │ │ │ │ │ │ │ │ │ │ │ │ │ │ ┌──────────────────────┐ │ │ ┌────────────────────────────────┐ │ │ │ │ │ │ │ Public Subnet (A) │ │ │ │ Public Subnet (B) │ │ │ │ │ │ │ │ NAT Gateway │ │ │ │ NAT Gateway │ │ │ │ │ │ │ └──────────────────────┘ │ │ └────────────────────────────────┘ │ │ │ │ │ │ │ │ │ │ │ │ │ │ ┌──────────────────────┐ │ │ ┌────────────────────────────────┐ │ │ │ │ │ │ │ Private Subnet (A) │ │ │ │ Private Subnet (B) │ │ │ │ │ │ │ │ ECS EC2 Cluster │ │ │ │ ECS EC2 Cluster │ │ │ │ │ │ │ │ (App Containers) │ │ │ │ (App Containers) │ │ │ │ │ │ │ │ ALB Target Group │ │ │ │ ALB Target Group │ │ │ │ │ │ │ └──────────────────────┘ │ │ └────────────────────────────────┘ │ │ │ │ │ │ │ │ │ │ │ │ │ │ ┌──────────────────────┐ │ │ ┌────────────────────────────────┐ │ │ │ │ │ │ │ RDS Private Subnet │ │ │ │ RDS Private Subnet (Standby) │ │ │ │ │ │ │ │ PostgreSQL Primary │◄───┼───┼──│ PostgreSQL Multi-AZ Standby │ │ │ │ │ │ │ └──────────────────────┘ │ │ └────────────────────────────────┘ │ │ │ │ │ └──────────────────────────────┘ └──────────────────────────────────────┘ │ │ │ │ │ │ │ │ ┌───────────────────┐ ┌───────────────────┐ ┌──────────────────────────┐ │ │ │ │ │ Amazon S3 │ │ AWS Systems Mgr │ │ AWS DMS Replication │ │ │ │ │ │ (File Storage │ │ (SSM / Run Cmd / │ │ Instance (Migration) │ │ │ │ │ │ + Lifecycle) │ │ Patch / Params) │ │ │ │ │ │ │ └───────────────────┘ └───────────────────┘ └──────────────────────────┘ │ │ │ │ │ │ │ │ ┌───────────────────┐ ┌───────────────────┐ │ │ │ │ │ Amazon ECR │ │ AWS Backup │ │ │ │ │ │ (Container Reg.) │ │ (Centralized DP) │ │ │ │ │ └───────────────────┘ └───────────────────┘ │ │ │ └─────────────────────────────────────────────────────────────────────────────────┘ │ └──────────────────────────────────────────────────────────────────────────────────────┘

```

---

## Core Design Principles

| Pillar | Principle | Implementation |
|---|---|---|
| **Reliability** | Eliminate AZ-level SPOF across all tiers | ECS across 2+ AZs, RDS Multi-AZ, dual NAT Gateways |
| **Security** | Zero data traversal over public internet | Direct Connect primary + IPSec VPN failover; private subnets |
| **Performance** | Consistent, dedicated throughput for DC↔AWS path | AWS Direct Connect (non-shared, dedicated bandwidth) |
| **Operational Excellence** | Unified tooling across hybrid boundary | ECS Anywhere + SSM Agent on all managed nodes |
| **Cost Optimization** | Automated data lifecycle and storage tiering | S3 Lifecycle Policies + S3 Intelligent-Tiering |
| **Sustainability** | Right-size compute; autoscale to demand | ECS Cluster Autoscaling + RDS Storage Autoscaling |

---

## Infrastructure Components

### 1. Network Connectivity Layer

#### Primary: AWS Direct Connect

AWS Direct Connect provides a **private, dedicated network connection** between the corporate data center and the AWS Cloud. Traffic traversing this link never touches the public internet, remaining on the AWS global network backbone for the entirety of its transit.

**Why Direct Connect over AWS Site-to-Site VPN (as primary)?**

| Attribute | AWS Direct Connect | AWS Site-to-Site VPN |
|---|---|---|
| Transport medium | Dedicated private fiber | Public internet (IPSec tunnel) |
| Throughput consistency | Guaranteed (1 Gbps / 10 Gbps / 100 Gbps options) | Variable (subject to internet congestion) |
| Latency predictability | Deterministic, low jitter | Variable |
| Encryption in transit | Optional MACsec (Layer 2) | IPSec (mandatory) |
| Best for | High-volume, latency-sensitive workloads | Cost-sensitive or secondary/failover links |
| Cost | Higher (port + partner fees) | Lower |

> **⚠️ PRODUCTION NOTE:** A single Direct Connect connection introduces a physical dependency on the on-premises router. For enterprise SLA compliance, always provision a **redundant failover path**. The recommended architecture implements an **AWS Site-to-Site VPN (IPSec) as an active failover** for the Direct Connect circuit. In the event of a Direct Connect failure, all VPC traffic automatically routes through the VPN. Traffic destined for public AWS endpoints (e.g., Amazon S3, public APIs) routes over the internet regardless of the failover state.

**Recommended Redundancy Pattern:**
```

DC Router (Primary) ──► AWS Direct Connect ──► Virtual Private Gateway ──► VPC DC Router (Failover) ──► Site-to-Site VPN ──► Virtual Private Gateway ──► VPC

```

#### VPN Architecture: AWS Client VPN vs. Site-to-Site VPN

| Service | Use Case | Relevant to This Architecture |
|---|---|---|
| **AWS Site-to-Site VPN** | Connect entire remote network (DC / branch) to a VPC | ✅ Yes — used as Direct Connect failover |
| **AWS Client VPN** | Connect individual administrators/endpoints (laptops) to AWS or DC | ❌ Out of scope for workload connectivity |

#### Subnet Allocation Strategy

Since all migrated applications are **internal-only** (no public internet ingress required), the VPC subnet allocation heavily favors private subnets:

| Subnet Type | Purpose | Sizing Recommendation |
|---|---|---|
| **Private Subnets (per AZ)** | ECS workloads, RDS instances, DMS | Large CIDR blocks (e.g., `/21` or `/20`) |
| **Public Subnets (per AZ)** | NAT Gateway egress, ALB (if internet-facing ALB needed later) | Small CIDR blocks (e.g., `/28` or `/27`) |

> **⚠️ SECURITY NOTE:** ECS container instances, RDS DB instances, and the DMS replication instance must reside exclusively in **private subnets**. Egress to the public internet (for OS patching, dependency downloads, AWS API calls) is routed through NAT Gateways deployed in the public subnets of each Availability Zone. Direct internet ingress to private subnet resources must be blocked at the Security Group and NACL layers.

---

### 2. Compute & Container Orchestration

#### Amazon ECS — Cloud-Hosted Containers

Amazon Elastic Container Service (ECS) manages the lifecycle of containerized application workloads running on AWS. The compute backing for the ECS cluster uses **EC2-based capacity** (vs. AWS Fargate), providing:

- Full SSH access to underlying EC2 instances (parity with on-premises operational patterns).
- Ability to bring custom AMIs (including hardened, CIS-benchmarked golden images).
- Control over instance type selection for workload-specific optimization (memory-optimized, compute-optimized).

**Cluster Scaling Architecture:**
```

Application Load Balancer (ALB) │ ▼ ECS Service (Task Definitions) │ ├── ECS Cluster Auto Scaling (Managed via Capacity Provider) │ └── EC2 Auto Scaling Group (adds/removes EC2 nodes) │ └── ECS Service Auto Scaling (adds/removes container Tasks) ├── Target Tracking Policy (recommended) └── Step Scaling Policy (secondary)

```

> **⚠️ OPERATIONAL NOTE:** EC2 Auto Scaling and ECS Service Auto Scaling are **independent mechanisms** that must be configured together. Scaling EC2 nodes without scaling ECS tasks (or vice versa) will result in capacity mismatches — either underutilized nodes or task placement failures. Use ECS Cluster Auto Scaling with a **managed scaling Capacity Provider** to couple these two dimensions automatically.

#### Amazon ECS Anywhere — On-Premises Container Orchestration

ECS Anywhere extends Amazon ECS management to **customer-managed on-premises infrastructure**, enabling identical task definition authoring, deployment pipelines, and observability tooling across the hybrid boundary.

**ECS Anywhere Bootstrap Sequence (per on-premises server):**

```bash
# Step 1: Generate SSM activation key and ID from the AWS Console or CLI
aws ssm create-activation \
  --iam-role AmazonECSAnywhereIAMRole \
  --registration-limit 10 \
  --region us-east-1

# Step 2: Execute the official ECS Anywhere installation script on the target node
# (Replace ACTIVATION_ID, ACTIVATION_CODE, and CLUSTER_NAME with generated values)
curl --proto "https" -o "/tmp/ecs-anywhere-install.sh" \
  "https://amazon-ecs-agent.s3.amazonaws.com/ecs-anywhere-install-latest.sh"

bash /tmp/ecs-anywhere-install.sh \
  --region us-east-1 \
  --cluster CLUSTER_NAME \
  --activation-id ACTIVATION_ID \
  --activation-code ACTIVATION_CODE
```

> **⚠️ PREREQUISITE:** Both the **AWS SSM Agent** and the **Amazon ECS Agent** must be installed and operational on all on-premises nodes registered with ECS Anywhere. The SSM Agent is the transport mechanism through which ECS Anywhere communicates with the AWS control plane.

**Impact Assessment: ECS Anywhere Adoption**

| Item | Impact | Notes |
|---|---|---|
| Container image code changes | None | Container images are unchanged |
| Task definitions | Requires authoring | New task definitions must be written for all on-premises workloads |
| Networking | Requires planning | Outbound HTTPS (443) to AWS SSM and ECS endpoints required |
| IAM roles | Required | ECS Anywhere IAM Role must be provisioned |

#### Amazon ECR — Container Registry

Amazon Elastic Container Registry (ECR) serves as the private, encrypted, and IAM-integrated container image repository. It replaces Docker Hub for production workloads, eliminating dependency on a third-party public registry and enabling:

- Image vulnerability scanning (Amazon ECR Enhanced Scanning via Amazon Inspector).
- Cross-region replication for DR alignment.
- IAM-based access control — no registry credentials to rotate.

---

### 3. Relational Database Platform

#### Amazon RDS — PostgreSQL (Multi-AZ Deployment)

Amazon RDS is the managed relational database service used to host the migrated PostgreSQL workload. The **Multi-AZ deployment mode** is mandatory for production to ensure automatic failover and eliminate the database tier as a single point of failure.

**Multi-AZ Deployment Behavior:**
```

```
                ┌─────────────────────────┐
```

Applications ────►│ RDS DNS Endpoint │ (via DNS CNAME) │ (Stable — never changes)│ └────────────┬────────────┘ │ ┌──────────────────┴──────────────────┐ │ │ ┌───────────▼───────────┐ ┌───────────────▼──────────────┐ │ PRIMARY DB Instance │ │ STANDBY DB Instance │ │ Availability Zone A │◄─sync──►│ Availability Zone B │ │ (Serves all traffic) │ │ (Hot standby, no reads) │ └───────────────────────┘ └──────────────────────────────┘ │ Failure Detected │ ▼ ┌───────────────────────┐ │ Automatic Failover │ ← DNS updated by RDS; no endpoint │ (Typically < 60s) │ change for applications │ Standby Promoted │ └───────────────────────┘

```

> **⚠️ PRODUCTION NOTE:** The single-standby Multi-AZ configuration provides automatic failover in most scenarios but does not provide read-scale. For higher availability commitments on **MySQL or PostgreSQL**, evaluate the **Multi-AZ DB Cluster** (two readable standbys across 3 AZs), which delivers automatic failovers typically completing in **under 35 seconds** and up to **2x lower write latency** compared to the single-standby model.

#### RDS Read Replicas — Workload Isolation

Read replicas are used to offload read-heavy or CPU-intensive query workloads (e.g., business intelligence reports, analytics extracts) from the primary DB instance, preserving primary instance health and stability.

| Configuration | Use Case |
|---|---|
| **Same-Region Read Replica** | Offload BI/reporting queries, reduce primary CPU utilization |
| **Cross-Region Read Replica** | DR readiness; replica promotable to standalone DB in secondary Region |

> **⚠️ CRITICAL DISTINCTION:** Read replicas are **not a substitute for Multi-AZ**. Read replicas use asynchronous replication and can have replica lag. Multi-AZ standby instances use **synchronous** replication and are exclusively for automatic failover — they do not serve read traffic in the default single-standby configuration.

#### RDS Storage Configuration Reference

| Storage Type | IOPS Profile | Latency | Best For | Recommendation |
|---|---|---|---|---|
| **General Purpose SSD (gp3)** | Baseline 3,000 IOPS; burst to 16,000 | Single-digit ms | General OLTP workloads | ✅ Default starting point |
| **Provisioned IOPS SSD (io2)** | Up to 256,000 IOPS | Sub-ms consistent | I/O-intensive, latency-sensitive DBs | Use for high-throughput production |
| **Magnetic** | Low, variable | High | Legacy / archival (not recommended) | ❌ Do not use for new deployments |

> **⚠️ SCALING NOTE:** Enable **RDS Storage Auto Scaling** at provisioning time. Define a `MaxAllocatedStorage` ceiling appropriate to your compliance and cost budget. Storage auto scaling operates with near-zero downtime and eliminates the operational risk of manual intervention during unexpected growth events.

---

### 4. Hybrid Storage Platform

#### AWS Storage Gateway — Amazon S3 File Gateway

The S3 File Gateway satisfies the unique hybrid storage requirement: on-premises applications must write files using the **NFS protocol** without code refactoring, while the files must reside durably in **Amazon S3** for cloud-native access by AWS-hosted analytics and ML container workloads.

**Data Flow Architecture:**
```

On-Premises Application │ │ NFS (v3 or v4.1) write ▼ ┌─────────────────────────────────────────────────────────┐ │ Storage Gateway Appliance (On-Premises VM) │ │ - VMware ESXi / Hyper-V / Linux KVM / Hardware │ │ - Local cache (low-latency read/write for recent data) │ │ - Asynchronous upload to S3 (handles WAN latency) │ └─────────────────────────────────────┬───────────────────┘ │ HTTPS over Direct Connect ▼ ┌──────────────────────┐ │ Amazon S3 Bucket │ │ (Durable object │ │ store) │ └──────────┬─────────────┘ │ ▼ AWS-hosted ECS containers (Analytics / ML) access data directly via native S3 SDK/API

```

**Latency Characteristics:**

| Operation | Latency Profile | Mechanism |
|---|---|---|
| On-premises NFS write | Low (local cache, sub-ms) | Write hits the local Storage Gateway cache disk |
| Upload to S3 | Asynchronous (background) | Gateway uploads to S3 over Direct Connect after write ack |
| On-premises NFS read (cached data) | Low (local cache) | Served from local cache if within cache window |
| On-premises NFS read (cold data) | Higher (fetched from S3) | Retrieved from S3 over Direct Connect |
| AWS container S3 read | Low (native S3 API) | Direct S3 API — no gateway in the read path |

> **⚠️ OPERATIONAL NOTE:** The local cache disk on the Storage Gateway appliance is a critical performance component. Under-provisioning the cache disk will result in cache thrashing and increased read latency as cold data is fetched from S3. Size the cache based on the **working set** of recently written files that are likely to be re-read within the on-premises environment.

#### S3 Data Lifecycle & Cost Optimization

Configure S3 Lifecycle Policies aligned to the access patterns declared by the business:

| Time Since Object Creation | Access Pattern | Recommended S3 Storage Class | Action |
|---|---|---|---|
| 0 – 7 days | Active processing window | `S3 Standard` | No transition |
| 7 – 365 days | Occasional reference access | `S3 Standard-IA` or `S3 Intelligent-Tiering` | Transition |
| 365+ days | Archival / compliance retention | `S3 Glacier Flexible Retrieval` | Transition |
| Per retention policy | Beyond legal hold period | N/A | Expire (delete) |

> **⚠️ COST NOTE:** When enabling Cross-Region Replication (CRR) for DR, storage costs are replicated across both Regions. Lifecycle policies become **doubly important** when replicating, as unmanaged object accumulation in a secondary Region will compound storage costs without delivering proportional operational value.

---

### 5. Operational Management Plane

#### AWS Systems Manager

AWS Systems Manager is the unified, agent-based management plane governing all EC2 instances, on-premises servers, and ECS Anywhere nodes across the hybrid environment. The **SSM Agent** must be installed and IAM-authorized on every managed node.

**Key SSM Capabilities in Use:**

| SSM Feature | Operational Function | Hybrid Scope |
|---|---|---|
| **Run Command** | Execute scripts/commands across fleet without SSH | ✅ AWS + On-Premises |
| **Patch Manager** | Automated OS patching within defined Maintenance Windows | ✅ AWS + On-Premises |
| **Maintenance Windows** | Schedule operational tasks (patching, scripts) on defined cadence | ✅ AWS + On-Premises |
| **Parameter Store** | Externalized configuration key-value store (avoids config hardcoding) | ✅ AWS + On-Premises |
| **Session Manager** | Browser/CLI-based shell access (eliminates SSH bastion requirement) | ✅ AWS + On-Premises |
| **Inventory** | Collect and query software/hardware inventory across fleet | ✅ AWS + On-Premises |

**Parameter Store Usage Pattern:**

```bash
# Writing a parameter (e.g., S3 bucket name for application config)
aws ssm put-parameter \
  --name "/prod/insurance-platform/s3-data-bucket" \
  --value "corp-insurance-data-prod-use1" \
  --type "String" \
  --tier "Standard"

# Writing a sensitive parameter (e.g., DB connection string) — encrypted via KMS
aws ssm put-parameter \
  --name "/prod/insurance-platform/db-connection-string" \
  --value "postgresql://user:pass@rds-endpoint:5432/insurancedb" \
  --type "SecureString" \
  --key-id "arn:aws:kms:us-east-1:123456789012:key/YOUR-KMS-KEY-ID"

# Reading in application code / ECS task startup scripts
aws ssm get-parameter \
  --name "/prod/insurance-platform/s3-data-bucket" \
  --query "Parameter.Value" \
  --output text
```

> **⚠️ SECURITY NOTE:** Never pass sensitive configuration (database credentials, API keys, encryption keys) via environment variables in ECS Task Definitions as plaintext. Use **SSM Parameter Store SecureString** parameters or **AWS Secrets Manager** and reference them in the Task Definition using the `secrets` block. The ECS agent will inject the resolved value into the container at runtime without it appearing in plaintext in the task definition or CloudTrail logs.

---

### 6. Data Protection & Backup

#### AWS Backup — Centralized Data Protection

AWS Backup provides policy-based, centralized backup management across all supported AWS resources and hybrid workloads within this architecture, including:

- Amazon RDS DB instances (PostgreSQL)
- Amazon EBS volumes (EC2 instance root and data volumes)
- Amazon EFS file systems
- Amazon S3 buckets
- AWS Storage Gateway volumes
- VMware workloads running on-premises

**Backup Plan Configuration Reference:**

| Resource | RPO Target | Backup Frequency | Retention | Vault |
|---|---|---|---|---|
| RDS PostgreSQL (Primary) | 1 hour | Continuous (Point-in-Time Recovery) + Daily snapshot | 35 days | Production Vault |
| EC2 EBS Volumes | 4 hours | Every 4 hours (using DLM) | 14 days | Production Vault |
| S3 Bucket (data) | 24 hours | Daily (S3 Backup via AWS Backup) | 90 days | Production Vault |
| Storage Gateway Volumes | 24 hours | Daily | 30 days | Production Vault |
| On-Premises VMware | 24 hours | Daily | 14 days | Production Vault |

---

## Database Migration Runbook

> **⚠️ MIGRATION CRITICAL:** This is a **homogeneous migration** (PostgreSQL on-premises → Amazon RDS PostgreSQL). No schema conversion tooling (AWS SCT) is required. The source database **remains fully operational** throughout the migration — applications must not be switched to the RDS target until the DMS replication lag reaches near-zero and a formal cutover window has been approved by the DBA lead and Change Advisory Board.

### AWS DMS Architecture
```

┌──────────────────────────────┐ │ Source: On-Premises PostgreSQL │ │ (Fully operational during │ │ entire migration) │ └───────────────┬──────────────┘ │ CDC (Change Data Capture) via logical replication ▼ ┌──────────────────────────────────────────┐ │ AWS DMS Replication Instance │ │ (Deployed in private subnet, AWS) │ │ - Full load phase │ │ - Ongoing replication phase (CDC) │ └───────────────┬──────────────────────────┘ │ ▼ ┌──────────────────────────────┐ │ Target: Amazon RDS │ │ PostgreSQL (Multi-AZ) │ │ (Receives full load + │ │ ongoing CDC until cutover) │ └──────────────────────────────┘

```

### Migration Phase Sequence

- [ ] **Phase 1 — Pre-Migration Validation**
  - [ ] Baseline the source PostgreSQL version; confirm RDS target runs identical engine version
  - [ ] Audit source schema for DMS compatibility (LOB columns, unsupported data types)
  - [ ] Enable logical replication on source (`wal_level = logical`, `max_replication_slots ≥ 1`)
  - [ ] Provision DMS Replication Instance in private subnet (size proportional to DB volume and change rate)
  - [ ] Create DMS Source Endpoint (on-premises PostgreSQL) and Target Endpoint (RDS)
  - [ ] Test endpoint connectivity and validate IAM permissions
- [ ] **Phase 2 — Full Load**
  - [ ] Create DMS migration task with `Start task on create = Full load + CDC`
  - [ ] Monitor DMS task metrics: `FullLoadThroughputRows`, `CDCThroughputRows`, `CDCLatencySource`, `CDCLatencyTarget`
  - [ ] Validate row counts between source and target after full load completes
- [ ] **Phase 3 — Ongoing CDC (Pre-Cutover)**
  - [ ] Monitor `CDCLatencyTarget` — this must approach near-zero before scheduling cutover
  - [ ] Validate data integrity with application-level query spot checks on RDS
- [ ] **Phase 4 — Cutover**
  - [ ] Initiate application maintenance window (coordinate with stakeholders)
  - [ ] Quiesce all writes to source database
  - [ ] Confirm DMS replication lag reaches 0
  - [ ] Update application connection strings to point to RDS endpoint (DNS CNAME or Parameter Store update)
  - [ ] Validate application connectivity and functionality against RDS
  - [ ] Stop DMS task
- [ ] **Phase 5 — Post-Cutover**
  - [ ] Monitor RDS CloudWatch metrics for 72-hour stabilization window
  - [ ] Decommission DMS replication instance after confirmation
  - [ ] Archive DMS task logs

---

## Disaster Recovery Strategy

Four DR strategies are available, ordered from lowest to highest recovery speed (and cost):

| Strategy | RTO | RPO | Cost Profile | Architecture Complexity |
|---|---|---|---|---|
| **Backup & Restore** | Hours | Hours (last backup point) | Lowest | Lowest |
| **Pilot Light** | 10–30 min | Minutes | Low-Medium | Medium |
| **Warm Standby** | Minutes | Seconds–Minutes | Medium-High | High |
| **Multi-Site Active/Active** | Near-zero | Near-zero | Highest | Highest |

> **🏗️ RECOMMENDED FOR THIS CUSTOMER:** Begin with **Warm Standby** in a secondary AWS Region, given enterprise-grade uptime commitments. Maintain a reduced-capacity ECS cluster and a promoted-ready RDS cross-Region read replica in the DR Region. Trigger automated failover via Route 53 health checks.

**DR Data Replication Matrix:**

| Data Store | Replication Mechanism | RPO Impact |
|---|---|---|
| Amazon RDS PostgreSQL | Cross-Region Read Replica (async) | Minutes of potential lag |
| Amazon S3 (file data) | S3 Cross-Region Replication (CRR) | Near-real-time (seconds to minutes) |
| ECS Container Images (ECR) | ECR Cross-Region Replication | Near-real-time |
| Application Config (SSM) | Manual sync or SSM cross-region parameter mirroring | On-demand |

> **⚠️ IaC MANDATE:** Multi-Region deployments without Infrastructure as Code are operationally unacceptable. All infrastructure in the primary Region must be defined in **AWS CloudFormation** or **AWS CDK** and version-controlled. The same IaC templates must be parameterized for secondary Region deployment, ensuring configuration parity. Drift between Regions is a compliance violation.

---

## Architecture Optimizations & Scaling

### Identified Enhancement Areas

| Area | Current State | Recommended Enhancement | Priority |
|---|---|---|---|
| Direct Connect redundancy | Single circuit | Add Site-to-Site VPN as IPSec failover | 🔴 High |
| RDS storage scaling | Manual provisioning | Enable RDS Storage Auto Scaling | 🔴 High |
| S3 cost optimization | Standard storage class | Enable S3 Intelligent-Tiering + Lifecycle Policies | 🟡 Medium |
| S3 DR replication | Single Region | Enable S3 Cross-Region Replication (CRR) | 🟡 Medium |
| RDS cross-region DR | Single Region | Provision cross-region RDS Read Replica | 🟡 Medium |
| Container scaling | EC2 Auto Scaling only | Add ECS Cluster Autoscaling Capacity Provider | 🔴 High |
| Container image registry | Docker Hub (external dependency) | Migrate to Amazon ECR | 🟡 Medium |
| Multi-Region DR | Not implemented | Implement Warm Standby in secondary Region | 🟢 Low (future phase) |

---

## Security Posture

| Security Control | Implementation | Standard |
|---|---|---|
| Network isolation | All workloads in private subnets; no direct internet ingress | VPC Security Groups + NACLs |
| Transit encryption (DC↔AWS) | Direct Connect MACsec (L2) + VPN IPSec (L3 failover) | TLS 1.2+, AES-256 |
| Data at rest encryption | RDS encryption (KMS), S3 SSE-KMS, EBS encryption | AWS KMS CMK |
| Secrets management | SSM Parameter Store SecureString / AWS Secrets Manager | No plaintext credentials in code or configs |
| Container image scanning | Amazon ECR Enhanced Scanning (Amazon Inspector) | CVE threshold gates in CI/CD pipeline |
| Privileged access | Session Manager (no SSH bastion; no inbound port 22 required) | AWS SSM Session Manager |
| Patch compliance | Systems Manager Patch Manager with compliance reporting | CIS Benchmark Level 2 |
| Backup integrity | AWS Backup with vault lock (WORM) for compliance retention | Immutable backup vaults |
| IAM least privilege | Task-level IAM roles for ECS tasks; no wildcard permissions | IAM Access Analyzer |

---

## Environment Configuration Reference

| Parameter | Example Value | Store | Notes |
|---|---|---|---|
| `RDS_ENDPOINT` | `corp-insurance-db.cluster-xxxxxx.us-east-1.rds.amazonaws.com` | SSM SecureString | DNS CNAME — stable across failover |
| `RDS_PORT` | `5432` | SSM String | Standard PostgreSQL port |
| `RDS_DATABASE_NAME` | `insurance_core` | SSM String | |
| `RDS_MASTER_USERNAME` | `dbadmin` | SSM String | |
| `RDS_MASTER_PASSWORD` | `[REDACTED]` | SSM SecureString (KMS) | Rotate via Secrets Manager |
| `S3_DATA_BUCKET_NAME` | `corp-insurance-data-prod-use1` | SSM String | |
| `S3_BUCKET_REGION` | `us-east-1` | SSM String | |
| `ECS_CLUSTER_NAME` | `insurance-platform-prod-cluster` | SSM String | |
| `ECR_REGISTRY_URI` | `123456789012.dkr.ecr.us-east-1.amazonaws.com` | SSM String | |
| `STORAGE_GATEWAY_ARN` | `arn:aws:storagegateway:us-east-1:...` | SSM String | File Gateway ARN |
| `SSM_ACTIVATION_ID` | `[ECS Anywhere activation ID]` | Secrets Manager | Rotate after node registration |
| `DIRECT_CONNECT_ID` | `dxcon-xxxxxxxx` | Documentation | Physical circuit ID |
| `VPN_TUNNEL_1_IP` | `[AWS VPN endpoint IP]` | Documentation | For router config reference |

---

## Implementation Checklist

### Network & Connectivity
- [ ] Provision AWS Direct Connect connection via partner or AWS Console
- [ ] Configure Virtual Private Gateway and associate with VPC
- [ ] Validate BGP peering between on-premises router and Direct Connect
- [ ] Provision Site-to-Site VPN as IPSec failover path
- [ ] Configure route priority: Direct Connect preferred, VPN as failover
- [ ] Validate automatic failover behavior in non-production

### VPC & Subnet Architecture
- [ ] Design and document VPC CIDR allocation (no overlap with on-premises ranges)
- [ ] Provision private subnets (large) in minimum 2 AZs
- [ ] Provision public subnets (small) in same 2 AZs for NAT Gateways
- [ ] Deploy NAT Gateway in each public subnet (AZ-local for resilience)
- [ ] Configure route tables: private subnet routes egress via NAT Gateway

### Compute & Containers
- [ ] Author ECS Task Definitions for all cloud-hosted container workloads
- [ ] Create ECS Cluster with EC2 capacity provider and Auto Scaling group
- [ ] Configure ECS Cluster Auto Scaling (managed scaling capacity provider)
- [ ] Configure ECS Service Auto Scaling (target tracking policy per service)
- [ ] Generate SSM Activation for on-premises ECS Anywhere nodes
- [ ] Install SSM Agent and ECS Agent on all on-premises nodes
- [ ] Register on-premises nodes to ECS Cluster as EXTERNAL instances
- [ ] Author ECS Task Definitions for on-premises container workloads
- [ ] Configure Amazon ECR repositories and ECR cross-region replication

### Database
- [ ] Provision Amazon RDS PostgreSQL Multi-AZ DB instance
- [ ] Enable RDS Storage Auto Scaling with defined `MaxAllocatedStorage`
- [ ] Enable RDS Performance Insights
- [ ] Enable automated backups (35-day retention for PITR)
- [ ] Provision cross-region RDS Read Replica in DR Region
- [ ] Configure DMS Replication Instance and endpoints
- [ ] Validate DMS connectivity to source (on-premises) and target (RDS)
- [ ] Execute DMS full load and validate row count parity
- [ ] Monitor CDC replication lag pre-cutover
- [ ] Execute database cutover during approved maintenance window

### Storage
- [ ] Deploy Storage Gateway S3 File Gateway appliance on-premises (VMware/Hyper-V/KVM)
- [ ] Activate gateway and associate with AWS account
- [ ] Create S3 File Gateway NFS file share backed by target S3 bucket
- [ ] Mount NFS file share on on-premises application servers
- [ ] Validate write path: on-premises NFS write → local cache → S3
- [ ] Configure S3 Lifecycle Policy (Standard → IA → Glacier transitions)
- [ ] Enable S3 Cross-Region Replication (CRR) to DR Region bucket
- [ ] Enable S3 Intelligent-Tiering on data bucket

### Operations & Data Protection
- [ ] Confirm SSM Agent is installed and registered on all managed nodes (AWS and on-premises)
- [ ] Create SSM Parameter Store parameters for all application configuration
- [ ] Configure SSM Maintenance Windows for patching schedule
- [ ] Configure SSM Patch Manager with compliance baseline
- [ ] Create AWS Backup plans covering all in-scope resources
- [ ] Enable AWS Backup Vault Lock for compliance-regulated retention
- [ ] Test backup restore process for RDS and EBS in non-production
- [ ] Validate cross-region backup copy jobs

### Security
- [ ] Enable RDS encryption at rest (KMS CMK)
- [ ] Enable EBS encryption at rest for all EC2 volumes (KMS CMK)
- [ ] Enable S3 SSE-KMS for all data buckets
- [ ] Configure Security Groups (least-privilege, no 0.0.0.0/0 inbound on private resources)
- [ ] Enable Amazon ECR image scanning; define vulnerability threshold policy
- [ ] Enable AWS CloudTrail (all Regions, S3 log storage with object lock)
- [ ] Enable AWS Config for resource compliance tracking
- [ ] Enable Amazon GuardDuty for threat detection
- [ ] Review IAM roles with IAM Access Analyzer; eliminate wildcard policies

---

## Operational Runbooks

### RDS Failover Testing (Quarterly)
```bash
# Initiate a manual RDS Multi-AZ failover (non-production validation)
aws rds reboot-db-instance \
  --db-instance-identifier corp-insurance-db-instance \
  --force-failover

# Monitor the failover event in CloudWatch and validate DNS resolution
# resolves to the standby's IP after failover completes
nslookup corp-insurance-db.cluster-xxxxxx.us-east-1.rds.amazonaws.com
```

### SSM Run Command — Fleet-Wide Script Execution
```bash
# Execute a command across all ECS worker nodes (tagged Environment=Production)
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters 'commands=["df -h && free -m"]' \
  --comment "Disk and memory check — production fleet" \
  --output text
```

### ECS Service Deployment (Blue/Green Pattern)
```bash
# Force new deployment of an ECS service (pulls latest task definition revision)
aws ecs update-service \
  --cluster insurance-platform-prod-cluster \
  --service insurance-core-api \
  --force-new-deployment \
  --region us-east-1
```

---

## Further Reading & Official References

| Topic | Resource |
|---|---|
| VPC network connectivity options | [Networking to Amazon VPC connectivity options](https://docs.aws.amazon.com/whitepapers/latest/aws-vpc-connectivity-options/introduction.html) |
| AWS Direct Connect overview | [What is AWS Direct Connect?](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Welcome.html) |
| AWS Direct Connect + VPN failover | [Configure Direct Connect and VPN failover with Transit Gateway](https://aws.amazon.com/premiumsupport/knowledge-center/configure-direct-connect-vpn/) |
| Amazon RDS Multi-AZ deployments | [Amazon RDS Multi-AZ deployments](https://aws.amazon.com/rds/features/multi-az/) |
| Amazon RDS read replicas | [Working with read replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html) |
| AWS DMS documentation | [What is AWS Database Migration Service?](https://docs.aws.amazon.com/dms/latest/userguide/Welcome.html) |
| Amazon S3 File Gateway | [What is Amazon S3 File Gateway?](https://docs.aws.amazon.com/filegateway/latest/files3/what-is-file-s3.html) |
| AWS Storage Gateway features | [AWS Storage Gateway features](https://aws.amazon.com/storagegateway/features/) |
| Amazon ECS Anywhere | [Amazon ECS Anywhere](https://aws.amazon.com/ecs/anywhere/) |
| ECS Cluster Auto Scaling | [Amazon ECS cluster auto scaling](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cluster-auto-scaling.html) |
| ECS Service Auto Scaling | [Service auto scaling](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-auto-scaling.html) |
| AWS Systems Manager | [What is AWS Systems Manager?](https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html) |
| AWS Backup | [AWS Backup overview](https://aws.amazon.com/backup/) |
| Disaster Recovery on AWS (Whitepaper) | [Disaster Recovery of Workloads on AWS: Recovery in the Cloud](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html) |
| Amazon EBS volume types | [Amazon EBS volume types](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-volume-types.html) |
| S3 Intelligent-Tiering | [Amazon S3 Intelligent-Tiering](https://aws.amazon.com/s3/storage-classes/intelligent-tiering/) |
| RDS Storage Auto Scaling | [Managing capacity automatically with Amazon RDS storage autoscaling](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIOPS.StorageTypes.html#USER_PIOPS.Autoscaling) |
