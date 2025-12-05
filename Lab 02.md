
# **Module 2 — RDS Cost Optimization: Real-World Lab Scenarios**

### **Scenario Context**

Acado Corp uses several PostgreSQL and MySQL RDS instances for transactional workloads, reporting systems, authentication services, and analytics pipelines. Finance notices a **20–35% rise** in RDS cost month-over-month due to:

* Over-provisioned instance classes
* Underutilized IOPS
* High storage consumption
* Suboptimal backup retention
* Use of Multi-AZ where Single-AZ would suffice
* Frequent failover triggering cross-AZ replication costs
* Lack of automated cleanup for snapshots

Your goal: Perform **hands-on analysis** and **real-world optimization** of RDS spend.

---

# **Lab 1 — Identify Expensive RDS Workloads Using Performance Insights & CloudWatch**

### **Objective**

Use Performance Insights + CloudWatch Metrics to detect:

* Over-sized instances
* High IOPS workloads
* Slow queries inflating compute time
* Low CPU utilization (under-provisioning risk)
* High DB connections causing burst scaling

---

## **Step 1 — Enable Performance Insights (if not already enabled)**

```bash
aws rds modify-db-instance \
  --db-instance-identifier acado-prod-db \
  --enable-performance-insights \
  --performance-insights-retention-period 7 \
  --apply-immediately
```

---

## **Step 2 — Analyze Database Load (DB Load, Top Wait Events)**

Go to:

**RDS Console → Databases → Performance Insights**

Check:

* **DB Load vs vCPU count**
* **Top SQL Queries**
* **Top wait events (IO, locks, commit latency)**
* **I/O pressure**

If DB Load is **consistently < 2–3** on an 8-vCPU instance → you are over-provisioned.

---

## **Step 3 — Use CloudWatch Metrics for Additional Insight**

### CPU Utilization:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=acado-prod-db \
  --statistics Average \
  --period 300 \
  --start-time 2025-01-01T00:00:00Z \
  --end-time 2025-01-02T00:00:00Z
```

Signals:

* CPU consistently < 20% → oversized
* CPU > 80% → consider scaling or optimization

### IOPS:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ReadIOPS \
  --dimensions Name=DBInstanceIdentifier,Value=acado-prod-db \
  --statistics Average \
  --period 300 \
  --start-time 2025-01-01T00:00:00Z \
  --end-time 2025-01-02T00:00:00Z
```

Compare allocated vs consumed IOPS.

---

## **Expected Outcome**

You identify whether your database is:

✔ Over-provisioned
✔ Under-provisioned
✔ Storage/IO bound
✔ Query bound

This forms the foundation for cost optimization.

---

# **Lab 2 — Storage Auto-Scaling & IOPS Optimization Challenge**

### **Objective**

Fine-tune storage type, size, and IOPS to reduce cost.

---

## **Step 1 — Check Current Storage Type**

```bash
aws rds describe-db-instances \
  --db-instance-identifier acado-prod-db \
  --query "DBInstances[*].{Type:StorageType,Allocated:AllocatedStorage,Iops:Iops}"
```

Possible issues:

* Using **io1/io2** instead of **gp3** without needing guaranteed IOPS
* Over-provisioned size
* Auto-scaling threshold misconfigured

---

## **Step 2 — Migrate from io1/io2 → gp3 (Up to 20–40% Savings)**

```bash
aws rds modify-db-instance \
  --db-instance-identifier acado-prod-db \
  --storage-type gp3 \
  --apply-immediately
```

Then tune gp3 IOPS:

```bash
aws rds modify-db-instance \
  --db-instance-identifier acado-prod-db \
  --iops 3000 \
  --apply-immediately
```

Recommended values based on real utilization.

---

## **Step 3 — Resize Storage Auto-Scaling Threshold**

```bash
aws rds modify-db-instance \
  --db-instance-identifier acado-prod-db \
  --max-allocated-storage 2000 \
  --apply-immediately
```

This reduces unexpected auto-expansion and cost overruns.

---

## **Expected Outcome**

✔ Right-sized storage type
✔ Lower IOPS cost
✔ Reduced baseline storage cost
✔ Predictable scaling

---

# **Lab 3 — Multi-AZ vs Single-AZ Reconfiguration Cost Simulation**

### **Objective**

Understand when Multi-AZ is necessary, and simulate cost impact of switching modes.

---

## **Step 1 — Check Current Deployment Mode**

```bash
aws rds describe-db-instances \
  --db-instance-identifier acado-prod-db \
  --query "DBInstances[*].MultiAZ"
```

---

## **Step 2 — Evaluate Business Requirements**

Multi-AZ recommended for:

* Production transaction systems
* Health/Finance workloads
* Mission-critical APIs
* 24/7 SLA commitments

Single-AZ acceptable for:

* Dev/Test
* Internal batch workloads
* Analytics ETL
* Non-critical staging

---

## **Step 3 — Simulate Cost Difference**

For example:

**db.m5.large Multi-AZ:**
≈ $0.384/hr × 2 instances = $0.768/hr

**Single-AZ:**
≈ $0.384/hr → **50% cheaper**

---

## **Step 4 — Convert Multi-AZ → Single-AZ (for test environments)**

```bash
aws rds modify-db-instance \
  --db-instance-identifier acado-staging-db \
  --no-multi-az \
  --apply-immediately
```

Or enable Multi-AZ for production:

```bash
aws rds modify-db-instance \
  --db-instance-identifier acado-prod-db \
  --multi-az \
  --apply-immediately
```

---

## **Expected Outcome**

✔ Understand cost trade-offs
✔ Apply Multi-AZ only where business requires it
✔ Optimize staging & non-critical workloads

---

# **Lab 4 — Snapshot & Backup Retention Optimization (Automated Cleanup)**

### **Objective**

Reduce snapshot/storage cost through automation.

---

## **Step 1 — Check Current Backup Retention**

```bash
aws rds describe-db-instances \
  --db-instance-identifier acado-prod-db \
  --query "DBInstances[*].BackupRetentionPeriod"
```

Recommended:

* Production: **7–14 days**
* Dev/Test: **1–3 days**

Update retention:

```bash
aws rds modify-db-instance \
  --db-instance-identifier acado-prod-db \
  --backup-retention-period 7 \
  --apply-immediately
```

---

## **Step 2 — Automate Old Snapshot Cleanup with Lambda**

Lambda code:

```python
import boto3, datetime

rds = boto3.client("rds")
cutoff = datetime.datetime.utcnow() - datetime.timedelta(days=15)

def lambda_handler(event, context):
    snapshots = rds.describe_db_snapshots(SnapshotType="manual")["DBSnapshots"]

    for snap in snapshots:
        if snap["SnapshotCreateTime"].replace(tzinfo=None) < cutoff:
            rds.delete_db_snapshot(DBSnapshotIdentifier=snap["DBSnapshotIdentifier"])
```

---

## **Step 3 — Schedule Weekly Cleanup**

```bash
aws events put-rule \
  --name delete-old-snapshots \
  --schedule-expression "cron(0 1 ? * SUN *)"
```

---

## **Expected Outcome**

✔ Snapshot storage reduced
✔ Retention policies aligned with business needs
✔ No accidental long-term storage costs

---

# **Module 2 Deliverables**

By completing this module, participants will achieve:

* RDS workload analysis via Performance Insights
* Storage & IOPS right-sizing
* Cost simulation for Multi-AZ vs Single-AZ
* Automated snapshot + backup cleanup
* Practical understanding of RDS cost levers

