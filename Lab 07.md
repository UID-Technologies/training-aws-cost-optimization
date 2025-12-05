# **Module 7 — Real Case Study: End-to-End Cost Optimization Challenge**

## **Scenario Context: “Acado Enterprise Platform”**

Acado Corp is running a large, multi-tier production environment powering:

* A global learning platform
* High-traffic APIs
* Data analytics pipelines
* Containerized microservices
* Serverless automation workloads

Over several years, the environment has grown without structured governance. Recent finance reports show:

### **⚠️ AWS cost increased by 40–55% month-over-month.**

Participants are provided with an **intentionally inefficient, overly expensive architecture** that includes:

✔ Oversized RDS instances
✔ ECS/EKS clusters running on On-Demand compute only
✔ Lambda pipelines using synchronous flows
✔ S3 buckets with no lifecycle rules
✔ NAT gateways driving thousands in data transfer charges
✔ No PrivateLink/VPC endpoints configured
✔ CloudWatch logs left with infinite retention
✔ Snapshots retained for years beyond compliance requirements

Your mission:

# **🎯 Perform a complete end-to-end cost optimization across the entire stack and rebuild a highly optimized architecture.**

---

# **Module 7 Prerequisites**

### **AWS Knowledge Required**

Participants must understand:

* EC2 Auto Scaling & Predictive Scaling
* EKS/ECS workloads & Spot strategies
* Lambda optimization (async, memory tuning)
* VPC networking (NAT, PrivateLink, endpoints)
* S3 lifecycle & tiering
* RDS rightsizing & Performance Insights
* CloudWatch metrics & Flow Logs
* CUR/Cost Explorer basics

### **Environment Provided**

Participants receive:

1. **Inefficient architecture diagram**
2. **Cost & Usage (CUR) dataset**
3. **CloudWatch dashboards**
4. **RDS Performance Insights export**
5. **S3 storage class breakdown**
6. **EKS/ECS CPU/Memory usage reports**
7. **VPC Flow Logs (egress + cross-AZ patterns)**
8. **LAMBDAs + API Gateway configs**

---

# **Challenge Structure**

Participants will complete **five phases**, moving from discovery → diagnosis → optimization → redesign → presentation.

---

# **Phase 1 — Deep Architecture & Cost Analysis**

### **Objective:**

Identify cost drivers across compute, storage, serverless, RDS, networking, and observability.

---

## **Step 1 — Compute (EKS/ECS/EC2)**

Analyze:

* CPU/memory utilization
* Replica counts vs actual load
* EKS node group instance types
* Absence of Spot capacity
* Over-provisioned container requests/limits
* Low bin-packing efficiency

**Common finding:**
EKS cluster uses **100% On-Demand**, and containers request **5× required resources**.

---

## **Step 2 — RDS PostgreSQL Analysis**

Review:

* CPU/memory utilization
* Storage & IOPS
* Connection counts
* Backup retention
* Multi-AZ configuration
* Bursting vs provisioned IOPS

**Typical issue:**
`db.m5.4xlarge` at **18% CPU** with overprovisioned IOPS.

---

## **Step 3 — Serverless Pipeline Analysis**

Check:

* Lambda memory & timeout configuration
* Synchronous invocation chains
* Retry behavior causing cost spikes
* REST API instead of HTTP API
* Lack of concurrency limits

**Symptom:**
Lambda execution duration inflated due to upstream blocking.

---

## **Step 4 — S3 & Storage Analysis**

Investigate:

* Storage class mix
* Lifecycle retention
* Glacier usage
* Cross-region replication
* Orphaned logs & backups
* Duplicate datasets

**Finding:**
Over **120 TB** of data in S3 Standard.

---

## **Step 5 — Networking Cost Analysis**

Review:

* NAT Gateway flow
* Public S3 access
* Cross-AZ EC2 <→ RDS traffic
* EKS pulling images over the Internet
* Absence of VPC endpoints

**Finding:**
Monthly NAT Gateway charges: **$3,000–$5,000**.

---

# **Phase 2 — Identify Hidden Cost Drains**

Participants uncover:

### ✔ Over-provisioned compute everywhere

### ✔ No lifecycle or cold data strategy

### ✔ High NAT Gateway & egress charges

### ✔ Lambda chains causing inflated duration

### ✔ CloudWatch logs stored indefinitely

### ✔ Cross-AZ traffic caused by poor topology

These hidden leakages often account for **50%+ of total AWS cost**.

---

# **Phase 3 — Apply Cost Optimization Techniques**

Participants must apply techniques learned across Modules 1–6.

---

# **1. Compute Optimization (EKS/ECS/EC2)**

Actions:

* Enable **Karpenter** or tune Cluster Autoscaler
* Introduce **Spot node groups**
* Convert ECS → **Fargate Spot**
* Reduce CPU/memory requests/limits
* Resize EC2 nodes
* Use **Graviton** where possible

**Expected Savings:** 20–40%

---

# **2. RDS Optimization**

Actions:

* Downsize from `m5.4xlarge` → `m5.2xlarge`
* Reduce/auto-scale IOPS
* Enable **RDS Proxy**
* Apply shorter backup retention
* Enable storage alerts
* Evaluate Multi-AZ vs Single-AZ cost tradeoff

**Expected Savings:** 25–35%

---

# **3. Serverless Optimization**

Actions:

* Use **Lambda Power Tuning**
* Convert sync → async patterns
* Migrate REST → **HTTP API**
* Apply concurrency limits
* Reduce CloudWatch retention to 7–30 days

**Expected Savings:** 25–50%

---

# **4. S3 & Storage Optimization**

Actions:

* Apply Intelligent-Tiering
* Add lifecycle rules (Glacier / Deep Archive)
* Remove duplicate data
* Analyze S3 Access Logs
* Review replication & cross-region flow

**Expected Savings:** 30–60%

---

# **5. Networking Optimization**

Actions:

* Use **S3 Gateway Endpoints** & **PrivateLink**
* Reduce NAT Gateway dependency
* Tune ALB routing
* Place compute close to storage (same AZ)
* Remove public S3 usage

**Expected Savings:** 30–70%

---

# **Phase 4 — Rebuild the Optimized Architecture**

Participants deliver a **fully redesigned AWS architecture** featuring:

* Karpenter-based EKS with Spot + On-Demand mix
* Predictive ASG scaling for EC2 fleets
* Event-driven, async serverless workflows
* S3 lifecycle + Glacier Deep Archive governance
* Right-sized RDS with connection pooling
* PrivateLink-enabled VPC with minimal NAT usage
* CloudWatch logs stored with optimal retention
* Automated tagging & cost governance toolkit

Deliverables:

* Optimized Architecture Diagram
* Final cost projection (Before → After)
* ROI analysis
* Migration/transition plan

---

# **Phase 5 — Final Presentation**

Teams present:

### ✔ Cost Savings (% Reduction Achieved)

### ✔ Before vs After Architecture Diagrams

### ✔ Before vs After Cost Metrics

### ✔ Breakdown of Savings by Category

### ✔ Key Risks & Mitigation

### ✔ Long-Term Governance Recommendations

Evaluation Criteria:

* Cost impact
* Technical feasibility
* Completeness of redesign
* Clarity of communication
* Alignment with FinOps best practices

---

# **Module 7 Deliverables**

Participants complete:

* Full-stack cost review
* Multi-dimensional optimization plan
* Redesigned architecture
* Verified cost reduction ≥40%
* Final presentation + documentation
* Demonstrated expertise in enterprise-scale AWS cost engineering

This is the **capstone module** validating the participant’s ability to perform **real-world AWS cost optimization at scale**.

---
