# **Module 5 — Storage & Data Transfer Optimization Lab**

## **Scenario Context**

Acado Corp handles large-scale data ingestion pipelines, distributing logs, media, AI training datasets, and analytics snapshots across S3. A recent audit revealed:

* Huge amounts of cold data stored in **S3 Standard**
* No lifecycle rules for archival or cleanup
* High **inter-AZ and cross-region data transfer costs**
* Public S3 access causing **internet egress** charges
* Analytics workloads downloading unnecessary multi-GB datasets
* Services not using VPC endpoints or PrivateLink

### **Goal:**

Reduce storage and data transfer cost by **30–60%** through lifecycle automation, intelligent tiering, flow log diagnostics, and VPC-PrivateLink architecture.

This module includes **four hands-on labs + a final challenge**.

---

## **Module Prerequisites**

### **AWS Account Requirements**

* IAM permissions:
  `s3:*`, `logs:*`, `ec2:*`, `cloudwatch:*`, `vpc:*`
* Billing access (read-only)
* AWS CLI configured
* Ability to create VPC, S3 buckets, endpoints

### **Services that must exist (or be created during lab)**

* An S3 bucket: `acado-data-prod`
* A VPC with at least 2 subnets and route tables
* Existing workloads accessing S3 or APIs (for flow log analysis)

---

## **Initial Setup (Recommended)**

Before starting the labs:

1. Ensure VPC Flow Logs IAM Role exists.
2. Have sample files to upload to S3.
3. Create a simple internal API (API Gateway/Lambda) for PrivateLink lab.
4. Enable CloudWatch Logs Insights for querying network logs.

---

# **Lab 1 — S3 Intelligent-Tiering Automation**

## **Objective**

Automatically optimize S3 storage cost using **Intelligent-Tiering**, ensuring all new and existing objects transition without changing application code.

---

## **Lab Prerequisites**

* S3 bucket: `acado-data-prod`
* Lifecycle permissions
* S3 read/write IAM permissions

---

## **Step 1 — Check Current Storage Class**

```bash
aws s3api get-bucket-location --bucket acado-data-prod
```

---

## **Step 2 — Apply Intelligent-Tiering to Existing Objects**

```bash
aws s3 cp s3://acado-data-prod s3://acado-data-prod \
  --recursive \
  --storage-class INTELLIGENT_TIERING
```

---

## **Step 3 — Enable Intelligent-Tiering via Lifecycle Rule** *(recommended)*

Create **intelligent-tiering-rule.json**:

```json
{
  "Rules": [
    {
      "ID": "MoveToIntelligentTiering",
      "Prefix": "",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 0,
          "StorageClass": "INTELLIGENT_TIERING"
        }
      ]
    }
  ]
}
```

Apply the rule:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket acado-data-prod \
  --lifecycle-configuration file://intelligent-tiering-rule.json
```

---

## **Validation**

```bash
aws s3api get-bucket-lifecycle-configuration \
  --bucket acado-data-prod
```

**Outcome:**
All S3 data dynamically and automatically transitions between tiers, reducing cost by **20–40%**.

---

# **Lab 2 — Glacier Deep Archive Lifecycle Policies**

## **Objective**

Automate long-term archival of data (e.g., logs older than 90 days) into **Glacier Deep Archive** for ~95% cost savings.

---

## **Lab Prerequisites**

* Lifecycle permission on S3
* Data available for archival testing

---

## **Step 1 — Create a Lifecycle Policy**

Create **glacier-lifecycle.json**:

```json
{
  "Rules": [
    {
      "ID": "ArchiveOldFiles",
      "Prefix": "",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ]
    }
  ]
}
```

Apply:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket acado-data-prod \
  --lifecycle-configuration file://glacier-lifecycle.json
```

---

## **Step 2 — Validate Policy**

```bash
aws s3api get-bucket-lifecycle-configuration \
  --bucket acado-data-prod
```

---

## **Step 3 — Test Archival Behavior**

Upload file:

```bash
echo "sample" > test.txt
aws s3 cp test.txt s3://acado-data-prod/archives/test.txt
```

Lifecycle will move objects automatically based on rules.

**Outcome:**
Ultra-cold data is automatically archived to **Deep Archive**, reducing storage cost drastically.

---

# **Lab 3 — Diagnose High Data Transfer Costs Using CloudWatch & VPC Flow Logs**

## **Objective**

Identify costly inter-AZ, cross-region, and internet-based data transfers using Flow Logs and CloudWatch Insights.

---

## **Lab Prerequisites**

* VPC with Flow Logs enabled
* IAM Role for FlowLogs
* CloudWatch Logs Insights access

---

## **Step 1 — Enable VPC Flow Logs**

```bash
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-123abc \
  --traffic-type ALL \
  --log-group-name acado-vpc-flow-logs \
  --deliver-logs-permission-arn arn:aws:iam::<acc>:role/FlowLogsRole
```

---

## **Step 2 — Analyze Data Egress (Internet Charges)**

```sql
fields @timestamp, srcAddr, dstAddr, bytes
| filter dstAddr != "10.*"
| stats sum(bytes) as TotalEgressBytes by srcAddr
| sort TotalEgressBytes desc
```

**Identifies:**
→ EC2/Lambda/EKS workloads sending data outside the VPC = INTERNET EGRESS COST.

---

## **Step 3 — Detect Cross-AZ Charges**

```sql
fields @timestamp, srcAz, dstAz, bytes
| filter srcAz != dstAz
| stats sum(bytes) as CrossAZTransferBytes by srcAz, dstAz
```

**Identifies:**
→ AWS services causing inter-AZ transfer (e.g., EKS nodes, RDS, S3 writes).

---

## **Step 4 — Identify S3 Transfer Hotspots**

```sql
fields @timestamp, dstAddr, bytes
| filter dstAddr like /amazonaws.com/
| stats sum(bytes) by dstAddr
```

If requests hit public S3 endpoints → high egress charges.

---

# **Lab 4 — Build a PrivateLink Architecture to Eliminate Data Transfer Costs**

## **Objective**

Use S3 and API PrivateLink to create **fully private data paths**, eliminating NAT Gateway, inter-AZ, and public egress charges.

---

## **Lab Prerequisites**

* VPC with subnets
* Route tables
* API Gateway or internal service to expose
* IAM permissions to create VPC endpoints

---

## **Step 1 — Create an S3 Gateway Endpoint**

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-123abc \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-111aaa rtb-222bbb
```

**Benefits:**
✔ No NAT required
✔ No public traffic
✔ Zero egress fees for S3 access

---

## **Step 2 — Create an Interface Endpoint (PrivateLink) for Internal APIs**

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-123abc \
  --service-name com.amazonaws.vpce.us-east-1.vpce-svc-<id> \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-001 subnet-002 \
  --security-group-ids sg-123
```

Use cases:

* Lambda → internal microservices
* EC2/ECS → private APIs
* EKS → Amazon services

---

## **Step 3 — Validate Routing**

Verify DNS resolution:

```bash
dig myapi.internal
```

Confirm:

* No public IP
* Traffic flows only via VPC endpoints
* No cross-region hops

---

## **Step 4 — Compare Costs (Before vs After)**

### Without PrivateLink:

* Internet egress
* NAT Gateway charges
* Inter-AZ transfer

### With PrivateLink:

* Zero NAT charges
* Zero egress for S3/API
* Controlled AZ-local access

**Expected Savings:** 30–50%

---

# **Module 5 — Storage & Data Transfer Optimization Challenge**

### **Goal**

Optimize the given multi-stage S3 + compute pipeline to reduce **≥40% cost**.

### **Pipeline Includes**

* S3 raw logs
* S3 → Lambda → S3 ETL
* EKS/EC2 analytics queries
* Cross-region replication
* Public S3 access patterns

### **Allowed Actions**

✔ Intelligent-Tiering
✔ Lifecycle Deep Archive
✔ PrivateLink for S3 & API Gateway
✔ Minimize cross-AZ traffic
✔ Avoid public endpoints

### **Success Criteria**

* All workloads operate correctly
* No performance degradation
* Cost reduction ≥ 40%
* Validate using CUR + Flow Logs + CloudWatch metrics

---

# **Module 5 Deliverables**

Participants complete:

* Intelligent-Tiering automation
* Deep Archive lifecycle governance
* Flow Logs + CloudWatch cost diagnostics
* End-to-end PrivateLink architecture implementation
* Final 40% optimization challenge

