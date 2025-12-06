

# **Module 1 — Hands-on Deep Dive: Advanced AWS Cost Visibility**

This module provides enterprise-level skills to build complete AWS cost transparency using the Cost & Usage Report (CUR), Athena, Glue, QuickSight, AWS Config, and AWS Cost Anomaly Detection.

You will perform hands-on labs to build real FinOps visibility pipelines exactly as organizations do in production.

---

# **1. Module Overview**

Participants will learn to:

* Create and configure AWS CUR (Cost & Usage Report) — the single source of truth for AWS billing
* Query detailed cost data using Athena
* Build interactive dashboards using QuickSight
* Detect untagged resources using AWS Config + Lambda
* Simulate real anomalies using AWS Cost Anomaly Detection
* Build a repeatable cost-governance workflow

This module gives you **full visibility** into where AWS costs come from and how to control them.

---

# **2. Initial Setup (AWS Console Only)**

## Step 1 — Create an S3 Bucket for Storing CUR Files

1. Open **S3 Console**
2. Click **Create bucket**
3. Bucket name:

   ```
   aws-cost-visibility-<yourname>
   ```
4. Region: Keep default
5. Disable public access (enabled by default, recommended)
6. Click **Create bucket**

Why this matters:
CUR delivers large CSV/Parquet files; a dedicated bucket keeps billing data organized and secure.

---

## Step 2 — Validate IAM Permissions

Make sure your user has Console permissions for:

* Billing → View Billing
* AWS Cost & Usage Reports
* Glue
* Athena
* QuickSight
* AWS Config
* Lambda
* EventBridge
* Cost Explorer

If not, ask your admin to attach these managed policies:

* `AWSBillingReadOnlyAccess`
* `AmazonAthenaFullAccess`
* `AWSGlueConsoleFullAccess`
* `AmazonQuickSightFullAccess`
* `AWSConfigRulesFullAccess`
* `AWSLambda_FullAccess`
* `CloudWatchReadOnlyAccess`

---

# **Lab 1 — Create AWS Cost & Usage Report (CUR) and Query Using Athena**

## Goal

Create an end-to-end FinOps pipeline:
CUR → S3 → Glue → Athena → SQL analysis

This is the foundation for all enterprise cost governance.

---

## Step 1 — Create a Cost & Usage Report (Console Only)

1. Open **AWS Console → Billing → Cost & Usage Reports**
2. Click **Create report**
3. Report name:

   ```
   EnterpriseCostReport
   ```
4. Select the checkbox: **Include resource IDs**
5. Time granularity: **Hourly**
6. Report versioning: **Overwrite existing report**
7. Compression: **GZIP**
8. S3 bucket:

   * Choose the bucket you created earlier
   * Prefix: `cur/`

Click **Next → Review → Complete**.

CUR delivery will start; first file may take up to 24 hours.

---

## Step 2 — Create AWS Glue Crawler to Discover CUR Schema

1. Open **Glue Console → Crawlers → Create crawler**
2. Name: `cur-crawler`
3. Data source: **S3**
4. Select the path:

   ```
   s3://aws-cost-visibility-<yourname>/cur/
   ```
5. IAM Role: **Create new role**
6. Target database:

   * Create database: `cost_usage_db`
7. Click **Create**
8. Run the crawler

After it completes:
A new table will appear in Glue → Data Catalog → Tables.

---

## Step 3 — Query CUR Data Using Athena

1. Open **Athena Console → Query editor**
2. Choose Data source: **AwsDataCatalog**
3. Choose database: `cost_usage_db`
4. Select the table (usually named after the CUR folder)

Run these FinOps queries:

### Query 1 — Top AWS Services by Cost

```sql
SELECT product_product_name AS service,
       SUM(line_item_unblended_cost) AS cost
FROM enterprise_cost_report
GROUP BY 1
ORDER BY 2 DESC;
```

### Query 2 — Cost by Resource ID

```sql
SELECT resource_id,
       SUM(line_item_unblended_cost) AS cost
FROM enterprise_cost_report
WHERE line_item_unblended_cost > 0
GROUP BY 1
ORDER BY 2 DESC;
```

### Query 3 — Cost by Tag (cost center / owner / team)

```sql
SELECT resource_tags_user_costcenter AS cost_center,
       SUM(line_item_unblended_cost) AS cost
FROM enterprise_cost_report
GROUP BY 1
ORDER BY 2 DESC;
```

Athena now gives you full, SQL-based billing visibility.

---

## Lab 1 Checklist

| Task                             | Done | Verified | Screenshot |
| -------------------------------- | ---- | -------- | ---------- |
| S3 bucket created                |      |          |            |
| CUR created successfully         |      |          |            |
| Glue Crawler deployed            |      |          |            |
| Table discovered in Data Catalog |      |          |            |
| Athena queries executed          |      |          |            |
| Top cost drivers identified      |      |          |            |

---

# **Lab 2 — Build QuickSight Dashboards for Granular Cost Visibility**

## Goal

Create FinOps dashboards that show:

* Cost by service / region / account
* Cost by Lambda / ECS / EKS / container
* Cost by team / project / environment
* Trend analysis

---

## Step 1 — Enable QuickSight

1. Open **QuickSight Console**
2. Choose **Standard Edition** (recommended, low cost)
3. Enable access to:

   * S3 buckets
   * Athena

---

## Step 2 — Import Athena Dataset into QuickSight

1. QuickSight → **Datasets → New Dataset**
2. Select **Athena**
3. Choose:

   * Database: `cost_usage_db`
   * Table: `enterprise_cost_report`
4. Import into SPICE (recommended)

---

## Step 3 — Build Visuals

### Visual 1 — Cost by AWS Service

* Visual type: **Bar chart**
* X-axis: `product_product_name`
* Value: `sum(line_item_unblended_cost)`

### Visual 2 — Cost by Tag (Team / Project)

* Group by tag: `resource_tags_user_team`
* Value: cost

### Visual 3 — Cost by Lambda Function

Filter:

```
product_product_name = 'AWS Lambda'
```

Use field: `usage_type`

### Visual 4 — Kubernetes / ECS Container Level Cost

If tags exist:

* `kubernetes:pod-name`
* `kubernetes:namespace`
* `eks:cluster-name`

Use Pivot table:

* Rows: cluster → namespace → pod → container
* Value: cost

---

## Step 4 — Publish Dashboard

1. Click **Share → Publish dashboard**
2. Name:

   ```
   Enterprise FinOps Cost Dashboard
   ```
3. Share with users or groups.

---

## Lab 2 Checklist

| Task                           | Done | Verified | Screenshot |
| ------------------------------ | ---- | -------- | ---------- |
| QuickSight enabled             |      |          |            |
| Dataset imported from Athena   |      |          |            |
| Cost-by-service visual created |      |          |            |
| Tag-based cost map created     |      |          |            |
| Lambda cost panel built        |      |          |            |
| Dashboard published            |      |          |            |

---

# **Lab 3 — Enforce Tagging Governance (AWS Config + Lambda)**

### *(Console Only — Lambda created via Console Upload)*

## Goal

Detect untagged resources and enforce tagging policies.

---

## Step 1 — Create Required-Tags AWS Config Rule

1. Open **AWS Config Console → Rules → Add rule**
2. Search: **required-tags**
3. Required tag keys:

   * `costcenter`
   * `environment`
   * `owner`
4. Resource types:

   * EC2 Instances
   * EBS Volumes
   * RDS Instances
   * Lambda Functions
   * S3 Buckets
5. Save rule

AWS Config will now evaluate compliance automatically.

---

## Step 2 — Create Tag Enforcement Lambda

### Create Lambda (Console Only)

1. Go to **Lambda Console → Create function**
2. Choose **Author from scratch**
3. Name: `TagEnforcer`
4. Runtime: **Python 3.9**
5. Role: Auto-generated basic execution role
6. Create function

### Paste the following code:

(Latest AWS UI → Code tab → Paste code)

```python
import json

def lambda_handler(event, context):
    print("Received non-compliance event:")
    print(json.dumps(event))
    return {"status": "processed"}
```

Save and deploy.

This Lambda doesn't auto-tag (lab-safe), but processes non-compliant events.

---

## Step 3 — Create EventBridge Rule to Trigger Lambda

1. Open **EventBridge Console → Rules → Create rule**
2. Name: `TagNonComplianceRule`
3. Event pattern:

   * AWS service: `AWS Config`
   * Event type: Compliance Change
   * Compliance type: NON_COMPLIANT
4. Target: **Lambda → TagEnforcer**
5. Save

Now the Lambda will run every time a resource is missing required tags.

---

## Lab 3 Checklist

| Task                                 | Done | Verified | Screenshot |
| ------------------------------------ | ---- | -------- | ---------- |
| Created required-tags Config rule    |      |          |            |
| Evaluated resources                  |      |          |            |
| Created TagEnforcer Lambda           |      |          |            |
| Linked Config → EventBridge → Lambda |      |          |            |
| Validated event flow                 |      |          |            |

---

# **Lab 4 — AWS Cost Anomaly Detection + Real Simulation**

## Goal

Enable anomaly detection and simulate a cost spike to test automated alerting.

---

## Step 1 — Enable Cost Anomaly Detection

1. Open **Cost Explorer → Anomaly Detection**
2. Click **Create alert monitor**
3. Choose:

   * **AWS Service Monitor** (recommended)
4. Add email or SNS notifications
5. Save

---

## Step 2 — Create Individual Monitors

Examples:

* EC2 cost monitor
* EKS cost monitor
* S3 data transfer cost monitor
* Lambda usage monitor

This gives service-level FinOps visibility.

---

## Step 3 — Simulate a Cost Anomaly

Examples:

* Start an EC2 instance temporarily
* Trigger multiple Lambda invocations
* Upload large data to S3
* Increase RDS connections

AWS will detect unusual daily spend compared to historical baseline.

---

## Step 4 — Review the Detected Anomaly

1. Open **Cost Explorer → Anomaly Detection**
2. Click **Alerts**
3. Review:

   * Root cause
   * Impacted resources
   * Estimated additional spend
   * Suggested actions

---

## Lab 4 Checklist

| Task                     | Done | Verified | Screenshot |
| ------------------------ | ---- | -------- | ---------- |
| Created anomaly monitors |      |          |            |
| Configured alerts        |      |          |            |
| Simulated cost anomaly   |      |          |            |
| Received alert           |      |          |            |
| Reviewed anomaly report  |      |          |            |

---

# **Final Deliverables for Module 1**

Participants submit:

| Deliverable                           | Completed |
| ------------------------------------- | --------- |
| Athena CUR query results              |           |
| QuickSight dashboard                  |           |
| Tag governance rule + Lambda workflow |           |
| Anomaly simulation results            |           |
| Summary of top 5 AWS cost drivers     |           |

---

