

# **Module 2 — RDS Cost Optimization: Real-World Lab Scenarios**

## **1. Module Overview**

This module trains participants on identifying and reducing hidden RDS costs using real-world techniques. You will explore performance bottlenecks, instance rightsizing, storage optimizations, Multi-AZ cost decisions, and backup/snapshot governance.

Participants will perform hands-on labs using the **AWS Console and AWS CLI (Windows)**.

You will learn to:

* Diagnose costly RDS workloads
* Optimize instance classes, storage types, and allocated storage
* Evaluate Multi-AZ vs Single-AZ cost differences
* Implement automated snapshot cleanup to reduce long-term cost

---

# **2. Learning Objectives**

## **Identify and Diagnose RDS Cost Drivers**

Participants will learn to:

* Detect over-provisioned instance classes (common in production)
* Diagnose high CPU, memory, and I/O consumption
* Identify storage bottlenecks and unnecessary provisioned IOPS
* Evaluate SQL query cost impact using Performance Insights

## **Optimize RDS Storage**

Learn to:

* Migrate from gp2 → gp3
* Reduce provisioned IOPS
* Enable and configure storage autoscaling

## **Evaluate High Availability Cost Impact**

Understand:

* When Multi-AZ is required vs wasteful
* How Multi-AZ doubles compute + storage cost
* How to simulate real cost differences using the Pricing Calculator

## **Optimize Backups & Snapshots**

You will:

* Reduce retention periods
* Remove unnecessary manual snapshots
* Implement a **Lambda automation** to clean snapshots older than 30 days

---

# **3. Initial Setup Steps**

## **Step 1 — Identify the RDS Instance for the Lab**

Depending on your environment:

* The instructor may provide an RDS instance
* Or you may create a basic RDS instance as follows:

**Latest AWS UI RDS Creation Steps:**

1. Open **RDS Console → Databases → Create database**
2. Select:

   * Engine: **MySQL** or **PostgreSQL**
   * Templates: **Free tier**
3. Settings:

   * DB instance identifier: `lab-rds-instance`
4. DB instance size:

   * Choose **db.t3.micro** (free tier eligible)
5. Storage:

   * Storage type: **gp2** (default)
   * Allocated storage: 20 GB
6. Connectivity:

   * Public access → **No** (recommended)
7. Monitoring: leave defaults

Create database.

This instance will be used for optimization labs.

---

## **Step 2 — Confirm IAM Permissions**

Ensure your AWS user has:

* AmazonRDSFullAccess
* CloudWatchReadOnlyAccess
* AWSLambdaBasicExecutionRole
* AmazonEventBridgeFullAccess

Participants will validate permissions by simply opening the following consoles:

* RDS
* CloudWatch
* Lambda
* EventBridge

If any denies appear, permissions must be updated.

---

## **Step 3 — Enable Performance Insights (If Disabled)**

Some RDS instances do NOT have Performance Insights enabled by default.

To enable:

1. Go to **RDS Console → Databases → Select your DB**
2. Click **Modify**
3. Scroll to **Performance Insights**
4. Enable
5. Choose retention: **7 days** (low cost)
6. Apply immediately

You will now be able to see advanced monitoring charts.

---

# **Lab 1 — Identify Expensive RDS Workloads Using Performance Insights & CloudWatch**

## **Goal**

Diagnose why an RDS instance may be costing more than necessary.

You will analyze:

* CPU + memory usage
* Expensive SQL queries
* Database load (AAS)
* Read/write IOPS
* Latencies
* Connection spikes

This simulation reflects real production troubleshooting.

---

## **Step 1 — Open Performance Insights**

1. Open **RDS Console → Databases**
2. Select your DB
3. Click the **Performance Insights** tab
4. Observe:

   * Database Load (AAS)
   * Top SQL statements
   * Top waits
   * Top users
   * CPU vs I/O load breakdown

### Why this matters:

Performance Insights identifies **exactly which SQL queries or workloads are causing cost** (due to CPU/I/O pressure).

---

## **Step 2 — Interpret the Performance Graph**

Change the graph timeline:

* 5 minutes
* 1 hour
* 24 hours

### Interpretation Guide:

* **AAS < 1** → DB is idle → probably over-provisioned
* **CPU consistently < 20%** → instance class oversized
* **High IO waits** → switch to gp3 or reduce IOPS
* **1–2 SQL queries dominating AAS** → inefficiency in query or index

This helps determine whether cost reduction should target:

* Compute
* Storage
* Query design

---

## **Step 3 — Analyze CloudWatch Metrics**

1. Open **CloudWatch Console → Metrics → RDS**
2. View graphs for:

   * CPUUtilization
   * FreeableMemory
   * FreeStorageSpace
   * ReadIOPS / WriteIOPS
   * ReadLatency / WriteLatency
   * DatabaseConnections

### What to look for:

* CPU under 20% for long periods → Oversized instance
* Memory under 20% usage → Potential rightsizing
* IOPS fluctuating → storage mismatch
* Latency > 20 ms → insufficient storage performance
* Sudden spikes → query issues or traffic bursts

---

## **Step 4 — Record Findings**

Participants must write down:

* Current instance class
* Observed CPU/memory utilization
* IOPS + latency
* Top SQL statements
* Bottlenecks detected

---

## **Lab 1 Checklist**

| Task                          | Done | Verified | Screenshot |
| ----------------------------- | ---- | -------- | ---------- |
| Opened Performance Insights   |      |          |            |
| Identified top SQL statements |      |          |            |
| Checked CPU / AAS             |      |          |            |
| Reviewed IOPS + Latencies     |      |          |            |
| Checked DB connections        |      |          |            |
| Recorded findings             |      |          |            |

---

# **Lab 2 — Storage Auto-Scaling & IOPS Optimization Challenge**

## **Goal**

Optimize the DB’s storage and IOPS costs.

You will:

* Switch storage type (gp2 → gp3)
* Reduce IOPS if oversized
* Enable storage auto-scaling
* Analyze cost impact

---

## **Step 1 — Check Current Storage Configuration**

Open:
**RDS Console → Databases → Select DB → Configuration**

Inspect:

* Storage type: gp2, gp3, io1, or io2
* Allocated storage (GB)
* Provisioned IOPS (for io1/io2)
* Autoscaling enabled? (Yes/No)

### Cost problems to identify:

* gp2 is older and less cost-efficient
* io1/io2 may have thousands of expensive IOPS
* No autoscaling = risk of over-provisioning storage

---

## **Step 2 — Convert gp2 to gp3**

With the latest AWS UI, conversion is simple.

1. Click **Modify**
2. Scroll to **Storage**
3. Change:

   * Storage type: **gp3**
4. Keep existing allocated storage

### Why gp3?

* Cheaper than gp2
* Choose IOPS separately
* Consistent performance

Apply the change.

---

## **Step 3 — Reduce Provisioned IOPS (If Using io1/io2)**

1. Open CloudWatch → RDS → Metrics → **ReadIOPS / WriteIOPS**
2. Check if actual IOPS is < 10–20% of provisioned

If yes:

1. Modify DB
2. Reduce provisioned IOPS

This step alone may save **hundreds of dollars per month** in production systems.

---

## **Step 4 — Enable Storage Autoscaling**

1. Modify DB
2. Scroll to Storage settings
3. Turn ON:
   **Storage Autoscaling**
4. Set maximum storage (e.g., 200 GB)

### Why this matters:

Avoids needing to over-provision storage “just in case”.

---

## **Step 5 — Evaluate Savings**

Participants should compare:

* gp2 vs gp3 pricing
* Provisioned IOPS changes
* Autoscaling max vs min GB

---

## **Lab 2 Checklist**

| Task                     | Done | Verified | Screenshot |
| ------------------------ | ---- | -------- | ---------- |
| Reviewed storage + IOPS  |      |          |            |
| Converted gp2 → gp3      |      |          |            |
| Reduced provisioned IOPS |      |          |            |
| Enabled autoscaling      |      |          |            |
| Calculated savings       |      |          |            |

---

# **Lab 3 — Multi-AZ vs Single-AZ Cost Simulation**

## **Goal**

Understand when Multi-AZ is truly necessary and how much it impacts cost.

---

## **Step 1 — Review Multi-AZ Settings**

1. Open RDS Console → Database → Configuration

Check:

* Multi-AZ: Enabled/Disabled
* Standby instance: Yes/No

### Cost rule:

Multi-AZ ≈ 2× compute + storage cost.

---

## **Step 2 — Evaluate Workload Requirements**

Participants decide whether the workload requires:

* Automatic failover
* Zero-downtime maintenance
* HA for production

If not → Single-AZ may be acceptable for dev/staging.

---

## **Step 3 — Estimate Cost Difference**

Use AWS Pricing Calculator:

1. Choose RDS engine
2. Select instance class
3. Compare **Multi-AZ** vs **Single-AZ**

Record:

* Monthly cost difference
* Annual savings

---

## **Step 4 — Modify RDS to Single-AZ (Lab Only)**

If allowed:

1. RDS → Modify
2. Choose **Single-AZ**
3. Apply immediately

This is safe for lab/test use.

---

## **Step 5 — Validate**

Ensure:

* DB status = Available
* Application can reconnect
* Metrics remain stable

---

## **Lab 3 Checklist**

| Task                               | Done | Verified | Screenshot |
| ---------------------------------- | ---- | -------- | ---------- |
| Checked Multi-AZ settings          |      |          |            |
| Performed pricing simulation       |      |          |            |
| Confirmed business requirements    |      |          |            |
| Modified to Single-AZ (if allowed) |      |          |            |
| Validated DB after change          |      |          |            |

---

# **Lab 4 — Snapshot & Backup Retention Optimization + Automation**

## **Goal**

Learn to reduce hidden storage costs from unused snapshots and excessive backup retention.

---

## **Step 1 — Review All Snapshots**

Open:
**RDS Console → Snapshots**

Look for:

* Manual snapshots older than 30–90 days
* Snapshots for deleted DBs
* Accumulated automated backups

Hidden costs usually come from old manual snapshots.

---

## **Step 2 — Update Backup Retention**

1. RDS → Modify
2. Backup Retention → Reduce to **7 days**

Long retention increases storage cost.

---

## **Step 3 — Delete Unnecessary Snapshots**

Delete:

* Old manual snapshots
* Orphan snapshots

---

## **Step 4 — Automate Cleanup (Windows CLI)**

### Create cleanup script

```cmd
notepad snapshot_cleanup.py
```

Paste:

```python
import boto3
import datetime

rds = boto3.client('rds')

def lambda_handler(event, context):
    cutoff = datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(days=30)
    snapshots = rds.describe_db_snapshots(SnapshotType='manual')['DBSnapshots']

    removed = []
    for snap in snapshots:
        if snap['SnapshotCreateTime'] < cutoff:
            rds.delete_db_snapshot(DBSnapshotIdentifier=snap['DBSnapshotIdentifier'])
            removed.append(snap['DBSnapshotIdentifier'])

    return {"deleted_snapshots": removed}
```

Zip:

```cmd
powershell Compress-Archive snapshot_cleanup.py function.zip
```

### Deploy Lambda

```cmd
aws lambda create-function --function-name SnapshotCleanup ^
 --runtime python3.9 --handler snapshot_cleanup.lambda_handler ^
 --zip-file fileb://function.zip ^
 --role <IAM_ROLE_ARN>
```

---

## **Step 5 — Schedule Weekly Cleanup (Windows CLI)**

Create rule:

```cmd
aws events put-rule --name WeeklySnapshotCleanup ^
 --schedule-expression "cron(0 3 ? * SUN *)"
```

Attach Lambda:

```cmd
aws events put-targets --rule WeeklySnapshotCleanup ^
 --targets Id=1,Arn=<LAMBDA_ARN>
```

---

## **Lab 4 Checklist**

| Task                              | Done | Verified | Screenshot |
| --------------------------------- | ---- | -------- | ---------- |
| Listed existing snapshots         |      |          |            |
| Updated retention period          |      |          |            |
| Deleted old snapshots             |      |          |            |
| Created cleanup Lambda            |      |          |            |
| Scheduled Lambda with EventBridge |      |          |            |
| Validated cleanup                 |      |          |            |

---

