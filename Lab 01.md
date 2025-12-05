
# **MODULE 1 — Advanced AWS Cost Visibility**

# **LAB 1 — Create a Custom AWS Cost & Usage Report (CUR) + Athena Analysis**

---

## **Objective**

By the end of this lab, participants will:

* Enable AWS Cost & Usage Report (CUR) delivery into S3
* Build metadata with AWS Glue (Crawler + Database)
* Query CUR using Athena to identify cost patterns
* Analyze high-cost services, EKS pod-level cost, and Lambda cost breakdown

This lab forms the foundation for all downstream cost analysis modules.

---

## **Prerequisites**

### **AWS Account Requirements**

* Billing access → **CUR permissions**
* IAM role/user with:

  * `s3:*`
  * `glue:*`
  * `athena:*`
  * `cur:*`
  * `billing:ViewBilling`

### **Tools Needed**

* AWS CLI (configured with credentials)
* Access to AWS Console (Billing, Glue, Athena, S3)

### **Initial Setup (Required Before Starting Lab)**

1. Enable **Athena** in the region you will use
2. Ensure **AWS Glue Service Role** exists:

   * `AWSGlueServiceRole` (or create one)
3. Ensure S3 bucket naming follows global uniqueness
4. Cost Explorer enabled (recommended but not mandatory)

---

## **Step 1 — Create an S3 Bucket for CUR Storage**

CUR must be delivered to an S3 bucket.

```bash
aws s3 mb s3://acado-cur-logs-<yourname>
```

Enable versioning to track CUR file changes:

```bash
aws s3api put-bucket-versioning \
  --bucket acado-cur-logs-<yourname> \
  --versioning-configuration Status=Enabled
```

**Validation Check**

* Go to S3 → confirm bucket exists

---

## **Step 2 — Configure Cost & Usage Report (CUR) in Billing Console**

1. Go to **AWS Console → Billing → Cost & Usage Reports**
2. Click **Create report**
3. Name it: **acado-cur-detailed**
4. Under *Report content*, enable:

   * ✔ Include resource IDs
   * ✔ Hourly granularity
   * ✔ GZIP compression
5. Delivery options:

   * Choose S3 bucket: `acado-cur-logs-<yourname>`
   * Prefix: `cur/`
6. Review → Save

**Result:** CUR files begin flowing within **24 hours**.

**Validation Check**

* Navigate to S3 bucket after 24 hours → check for `/cur/` folder

---

## **Step 3 — Create Athena Metadata via AWS Glue**

CUR files must be cataloged for Athena queries.

### **1. Create Database**

```bash
aws glue create-database --database-input "{\"Name\":\"cur_db\"}"
```

### **2. Create Glue Crawler**

```bash
aws glue create-crawler \
  --name acado-cur-crawler \
  --role AWSGlueServiceRole \
  --database-name cur_db \
  --targets "{\"S3Targets\": [{\"Path\": \"s3://acado-cur-logs-<yourname>/cur/\"}]}"
```

Start crawler:

```bash
aws glue start-crawler --name acado-cur-crawler
```

**Validation Check**

* In Glue → Databases → Tables → `acado_cur_detailed` appears

---

## **Step 4 — Query CUR Using Athena**

Go to **Athena Console → SQL Editor**
Choose database → **cur_db**
Select table → **acado_cur_detailed**

### **Query 1 — Top 10 Most Expensive AWS Services**

```sql
SELECT product_product_name,
       SUM(line_item_unblended_cost) AS cost
FROM cur_db.acado_cur_detailed
WHERE year = 2025 AND month = 01
GROUP BY product_product_name
ORDER BY cost DESC
LIMIT 10;
```

---

### **Query 2 — EKS Pod-Level Cost Analysis**

```sql
SELECT resource_id,
       SUM(line_item_unblended_cost) AS cost
FROM cur_db.acado_cur_detailed
WHERE product_product_name = 'Amazon Elastic Kubernetes Service'
GROUP BY resource_id
ORDER BY cost DESC;
```

---

### **Query 3 — Lambda Cost by Region & Usage Type**

```sql
SELECT product_region,
       product_usage_type,
       SUM(line_item_unblended_cost)
FROM cur_db.acado_cur_detailed
WHERE product_product_name LIKE '%Lambda%'
GROUP BY 1,2;
```

---

## **End of LAB 1 — Expected Deliverables**

✔ CUR dataset in S3
✔ Athena database + table created via Glue
✔ Ability to query high-cost AWS services
✔ Pod-level and Lambda-level cost insights

---

# **LAB 2 — Build Granular AWS Cost Dashboards with QuickSight**

---

## **Objective**

Build cost dashboards using QuickSight and Athena to visualize:

* Microservice-level costs
* Pod-level costs
* Lambda cost distribution
* Region-level spend
* Tag-based cost allocation

---

## **Prerequisites**

* QuickSight Enterprise enabled
* Access to Athena & S3 from QuickSight
* CUR dataset available via Athena (completed LAB 1)

---

## **Step 1 — Set Up QuickSight**

1. Open QuickSight
2. Choose **Enterprise Edition**
3. Enable data sources:

   * S3 bucket permissions
   * Athena access

**Validation Check**

* “Athena” appears as a selectable dataset source

---

## **Step 2 — Create Athena-Based Dataset**

1. Click **New Dataset → Athena**
2. Select database: **cur_db**
3. Select table: **acado_cur_detailed**
4. Import as SPICE or Direct Query

---

## **Step 3 — Build Visualizations**

### **A. Microservice / Pod Cost Breakdown**

* Fields: `resource_id`, `line_item_unblended_cost`
* Visual: **Bar Chart**
* Filter: `resource_id contains "pod"`

---

### **B. Lambda Cost Heatmap**

* Fields: `product_usage_type`, `line_item_unblended_cost`
* Visual: **Heatmap**

---

### **C. Region-Level Cost Analysis**

* Fields: `product_region`, `line_item_unblended_cost`
* Visual: **Geospatial Map**

---

### **D. Tag-Based Cost Allocation**

* Fields:

  * `resource_tags_user_application`
  * `line_item_unblended_cost`

Visual: **Tree Map** or **Stacked Bar Chart**

---

## **Step 4 — Publish the Dashboard**

1. Click **Share**
2. Assign access to **FinOps / DevOps teams**
3. Export to PDF for monthly reporting

---

# **LAB 3 — Enforce AWS Tagging Strategy Using AWS Config + Lambda**

---

## **Objective**

Automatically detect untagged or incorrectly tagged AWS resources using AWS Config and custom Lambda rules.

---

## **Prerequisites**

* AWS Config enabled
* IAM Role for Config + Lambda
* Basic Python familiarity

---

## **Initial Setup Required**

* Create S3 bucket for Config logs:
  `acado-config-logs-<yourname>`
* Ensure Config Recorder is enabled

---

## **Step 1 — Enable AWS Config**

1. Open **AWS Config**
2. Choose:

   * Record **all resource types**
   * Delivery channel → S3 bucket: `acado-config-logs-<yourname>`

---

## **Step 2 — Create Lambda for Tag Validation**

Paste into Lambda console or deploy via CLI:

```python
import json

def lambda_handler(event, context):
    invoking_event = json.loads(event['invokingEvent'])
    configuration = invoking_event['configurationItem']
    
    required_tags = ["Application", "Owner", "Environment"]
    missing = [tag for tag in required_tags 
               if tag not in configuration.get("tags", {})]

    if missing:
        return {
            "compliance_type": "NON_COMPLIANT",
            "annotation": f"Missing tags: {missing}"
        }

    return {
        "compliance_type": "COMPLIANT",
        "annotation": "All required tags present"
    }
```

---

## **Step 3 — Create Custom Config Rule**

1. Go to **AWS Config → Rules → Add rule**
2. Choose **Custom Lambda Rule**
3. Trigger type: **Configuration changes**
4. Select Lambda function
5. Name rule: **acado-required-tags**

---

## **Step 4 — Test the Rule**

### **A. Create an Untagged EC2 Instance**

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef \
  --instance-type t3.micro
```

Expected: **NON_COMPLIANT**

---

### **B. Apply Correct Tags**

```bash
aws ec2 create-tags \
  --resources <instance-id> \
  --tags Key=Application,Value=Acado \
         Key=Owner,Value=DevOps \
         Key=Environment,Value=Prod
```

Expected: **COMPLIANT**

---

# **LAB 4 — Configure AWS Cost Anomaly Detection + Simulate Cost Spike**

---

## **Objective**

Set up automated cost anomaly detection and simulate a safe anomaly to validate the detection system.

---

## **Prerequisites**

* IAM permissions for Billing & SNS
* Understanding of EC2 or Lambda workloads

---

## **Step 1 — Create Cost Anomaly Monitor**

1. Open **Billing → Cost Anomaly Detection**
2. Click **Create Monitor**
3. Choose:

   * **Service Monitor** (recommended)
4. Set threshold: **>$10 daily anomaly**
5. Add SNS/email alerts

---

## **Step 2 — Simulate a Safe Cost Spike**

### **Option A — Start multiple EC2 instances**

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef \
  --instance-type t3.micro \
  --count 5
```

### **Option B — Invoke a Lambda many times**

```bash
for i in {1..5000}; do
  aws lambda invoke \
    --function-name MyFn \
    --payload '{}' out.txt > /dev/null
done
```

**Wait 4–6 hours for anomaly detection.**

---

## **Step 3 — Analyze the Anomaly**

Use Athena or Cost Explorer:

```sql
SELECT product_service_code,
       SUM(unblended_cost) AS cost
FROM cost_and_usage
WHERE usage_date = CURRENT_DATE - 1
GROUP BY product_service_code
ORDER BY cost DESC;
```

---

# **Module 1 — Final Deliverables**

Participants complete:

* CUR → Athena pipeline
* Multi-dimensional cost dashboards
* Tag governance automation via AWS Config
* Cost anomaly detection workflow
* Validation of anomalies
