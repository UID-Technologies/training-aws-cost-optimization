
# **Module 7 — Real Case Study: End-to-End AWS Cost Optimization Challenge**

### *A Capstone, Hands-On FinOps Simulation Covering All Earlier Modules*

This module is **not a theory session** — it is a **full enterprise simulation** where participants optimize a deliberately expensive AWS environment.

They must:

* Diagnose high cost
* Apply optimization techniques across **compute, storage, network, database, serverless**
* Validate improvements using Cost Explorer, CloudWatch, and architecture redesign

This is the **final and most important module** of the FinOps program.

---

# **0. Assumptions for the Scenario**

To keep the lab realistic but manageable:

### You (trainer) will either:

#### Option A

Provide a **pre-built, expensive CloudFormation environment**.

#### Option B

Have students deploy a **Terraform/CFN template** that provisions costly resources.

### Each participant has:

* Their own AWS account
* Region = **us-east-1**
* IAM role/user with **AdministratorAccess**
* Familiarity with EC2, RDS, ECS/EKS, Lambda, S3, VPC

---

# **1. Overview of the Bad (Expensive) Architecture**

Participants are shown a typical enterprise AWS environment that evolved organically over time — with many anti-patterns.

### **Core Issues in Architecture**

### API Layer

* API Gateway **REST API** (expensive) instead of HTTP API
* Application Load Balancer for ECS/EKS traffic

### Compute Layer

* ECS running on **EC2 m5.large** instances
* EKS cluster with **3 On-Demand m5.large** nodes
* Lambda functions all configured to **1 GB** memory and no optimization

### Database Layer

* RDS MySQL:

  * **db.m5.large**
  * **Multi-AZ** always enabled
  * 100 GB **gp2** storage

### Storage Layer

* S3 using **STANDARD** storage
* Logs kept forever
* No lifecycle rules

### Networking Layer

* NAT Gateway in **every AZ** (3 NAT Gateways)
* No S3 / DynamoDB **VPC Endpoints**
* Heavy data transfer over NAT

### Observability Layer

* CloudWatch Logs set to **Never Expire**
* VPC Flow Logs disabled

---

The rest of Module 7 walks participants through **fixing everything**.

---

# **Lab 1 — Deep-Dive Cost Analysis Using Cost Explorer + VPC Flow Logs**

### **Goal**

Participants create a **forensic cost investigation**, identifying:

* Top-spend services
* Usage type anomalies
* NAT Gateway heavy usage
* Data transfer offenders
* Idle compute resources

---

## **Step 1.1 — Open Cost Explorer & Identify Top Services**

1. In AWS Console → search **Cost Explorer**
2. Click **Launch Cost Explorer**
3. Time range: **Last 7 or 30 Days**
4. Group by: **Service**

Ask participants to note:

* Which 3–5 services dominate cost?
* Are EC2, NAT Gateway, RDS, S3, Lambda among the top?

---

## **Step 1.2 — Drill Into EC2 (Biggest Offender)**

1. Apply Filter → **Service = Amazon EC2**
2. Group by:

   * **Instance Type**, or
   * **Usage Type**

Participants inspect:

* Are there large hours for `m5.large`, `m5.xlarge`, etc.?
* Are CPU patterns low?

---

## **Step 1.3 — Check NAT Gateway & Data Transfer Costs**

1. Clear filters
2. Group by:

   * **Usage Type Group**
   * **Usage Type**

Look for:

* DataTransfer-Out-Internet
* Regional-Bytes
* NATGateway-Bytes
* NATGateway-Hours

Participants realize:

* Too much S3/ECR/EKS traffic is going via NAT
* NAT gateways are running in multiple AZs

---

## **Step 1.4 — Enable VPC Flow Logs (Required for Root Cause)**

1. Go to **VPC → Your VPCs → Select VPC**
2. Open **Flow Logs** tab
3. Click **Create Flow Log**
4. Options:

   * Filter: **ALL**
   * Destination: **CloudWatch Logs**
   * Log Group: `/vpc/flowlogs-app`
5. Save

Flow Logs allow participants to identify:

* Which instance is generating outbound traffic
* Who is talking across AZs or to the internet

---

## **Step 1.5 — Query Flow Logs in CloudWatch Logs Insights**

After logs accumulate:

1. Go to **CloudWatch → Logs Insights**
2. Select `/vpc/flowlogs-app`
3. Run:

```sql
fields srcAddr, dstAddr, bytes, action
| sort bytes desc
| limit 20
```

Participants observe:

* Large outbound flows from EC2 → internet
* Calls going to S3/ECR using public endpoints
* Cross-AZ communication

---

### **Deliverable for Lab 1**

Participants provide:

✔ Top 3–5 cost drivers
✔ NAT / Cross-AZ data transfer analysis
✔ Flow log findings
✔ Summary of root causes

---

# **Lab 2 — S3 & CloudWatch Logs Optimization (Intelligent-Tiering, Glacier, Retention Policies)**

### **Goal**

Reduce storage cost dramatically by:

* Moving cold data to cheaper tiers
* Automatically archiving logs
* Reducing CloudWatch Logs retention

---

## **Step 2.1 — Identify S3 Buckets with Large or Old Data**

Go to **S3 Console** and inspect:

* log buckets
* analytics buckets
* backup buckets

Participants check:

* Object age
* Total size
* Access patterns

---

## **Step 2.2 — Create Lifecycle Policies (Glacier + Deep Archive)**

1. Open a log bucket
2. Go to **Management → Lifecycle Rules → Create Rule**
3. Call it: `logs-archive`
4. Actions:

   * Transition to **Glacier Flexible Retrieval** at 30 days
   * Transition to **Deep Archive** at 90 days
5. Optional:

   * Delete after 1 year

---

## **Step 2.3 — Configure Intelligent-Tiering for Mixed Data**

For user uploads, reports, or unpredictable data:

1. Create Lifecycle Rule
2. Set transition to **Intelligent-Tiering** (0 days)

OR manually convert existing objects:

* Select → **Actions → Change storage class → Intelligent-Tiering**

---

## **Step 2.4 — Fix CloudWatch Logs Retention**

1. Go to **CloudWatch → Log Groups**
2. Select any group set to **Never Expire**
3. Edit retention → set to:

   * 7 days (API logs)
   * 14–30 days (app logs)

---

### **Deliverable for Lab 2**

✔ Lifecycle rules screenshots
✔ Storage class changes
✔ Log retention evidence
✔ Estimated $ saved

---

# **Lab 3 — Compute Optimization (ECS, EKS, Lambda)**

A hands-on reconstruction of optimized compute.

---

# **Part A — ECS Optimization (EC2 → Fargate Spot)**

## **Step 3.1 — Inspect ECS Tasks**

Participants open ECS:

* See tasks running on EC2
* Check CPU/memory usage
* Identify over-provisioning

---

## **Step 3.2 — Create Fargate-Based Task Definition**

1. ECS → Task Definitions → Create
2. Type = **Fargate**
3. CPU = **256**, Memory = **512**
4. Add container (same image as EC2 tasks)

---

## **Step 3.3 — Enable Fargate Spot via Capacity Providers**

1. ECS Cluster → **Capacity Providers**
2. Add:

   * `FARGATE`
   * `FARGATE_SPOT`
3. Default strategy:

   * FARGATE_SPOT weight **3**
   * FARGATE weight **1**

---

## **Step 3.4 — Update Service to Use Fargate Spot**

1. ECS Service → Update
2. Launch type = **Capacity Provider Strategy**
3. Select the new Fargate task definition

This alone can cut ECS cost by **60–70%**.

---

# **Part B — EKS Optimization**

## **Step 3.5 — Add Spot Node Group**

EKS Console → Your Cluster → Compute → Add node group:

* Capacity type: **Spot**
* Instance types: diversify for availability
* Min = 0, Max = 5

---

## **Step 3.6 — Reduce On-Demand Node Group**

Reduce min nodes from 3 → 1.

Autoscaler will shift workloads to cheaper Spot nodes.

---

# **Part C — Lambda Optimization**

## **Step 3.7 — Use Lambda Power Tuning (SAR App)**

1. Open **Step Functions**
2. Execute *Lambda Power Tuning*
3. Select memory values:

   * 128, 256, 512, 1024
4. Identify best $/ms configuration
5. Update Lambda to optimal memory size

Many 1GB Lambdas drop to 256MB or 512MB → cost drops 40–60%.

---

### **Deliverable for Lab 3**

✔ ECS migrated to Fargate Spot
✔ EKS uses Spot + autoscaling
✔ Lambda memory reduced
✔ Screenshots of improvements

---

# **Lab 4 — RDS Optimization**

### **Goal**

Right-size RDS MySQL/Postgres for significant savings.

---

## **Step 4.1 — Review Utilization**

RDS → Databases → Select instance → Monitoring:

* CPU < 20%?
* Connections low?
* All signs of overprovisioning.

---

## **Step 4.2 — Modify DB for Cost Optimization**

Open **Modify**:

* Instance class → smaller (db.m5.large → db.t3.medium or db.t3.small)
* Storage → **gp3**
* Enable storage autoscaling
* Disable Multi-AZ for non-critical environments

Apply changes.

---

### **Deliverable for Lab 4**

✔ Before vs After instance class
✔ Storage changes
✔ Expected monthly savings

---

# **Lab 5 — Networking Optimization (NAT → VPC Endpoints + PrivateLink)**

### **Goal**

Eliminate NAT data charges by making all AWS traffic **private**.

---

## **Step 5.1 — Create S3 VPC Endpoint (Gateway)**

1. VPC Console → Endpoints → Create
2. Service = **S3**
3. Type = Gateway
4. Attach to route tables of private subnets

All S3 traffic now bypasses the NAT Gateway.

---

## **Step 5.2 — Add ECR & CloudWatch Interface Endpoints**

Important for ECS/EKS:

* `com.amazonaws.us-east-1.ecr.api`
* `com.amazonaws.us-east-1.ecr.dkr`
* `logs`
* `sts`

---

## **Step 5.3 — (Optional) Use PrivateLink for Internal APIs**

Expose service privately across VPCs.

---

### **Deliverable for Lab 5**

✔ VPC endpoints list
✔ Explanation of NAT savings
✔ Architecture showing private traffic

---

# **Lab 6 — Final Architecture Redesign + Cost Results Presentation**

This is the capstone deliverable.

---

## **Step 6.1 — Draw BEFORE Architecture**

Should include:

* ECS on EC2
* EKS fully On-Demand
* NAT in every AZ
* No VPC endpoints
* RDS large instance
* S3 Standard everywhere
* CloudWatch Never-Expire

---

## **Step 6.2 — Draw AFTER Architecture**

Should include:

* ECS on Fargate Spot
* EKS with Spot node groups
* NAT replaced by S3/ECR VPC endpoints
* RDS rightsized + gp3 + autoscaling
* S3 lifecycle + Intelligent-Tiering
* Lambda optimized memory

---

## **Step 6.3 — Create Cost Reduction Table**

| Area              | Before | After | Savings      |
| ----------------- | ------ | ----- | ------------ |
| Compute           | $X     | $Y    | Z%           |
| RDS               | $A     | $B    | C%           |
| S3                | $M     | $N    | P%           |
| NAT/Data Transfer | $E     | $F    | G%           |
| **Total**         |        |       | **≥ 40–50%** |

---

## **Step 6.4 — Final Team Presentation**

Teams present:

1. Original pain points
2. The optimizations applied
3. Architecture evolution
4. Before/after comparisons
5. Final cost savings score

---

# ✔ Module 7 Complete

This completes the real enterprise case study.

---

