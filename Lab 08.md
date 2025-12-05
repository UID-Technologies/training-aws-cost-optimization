# **Module 8 — Final Lab: Build an Automated Cost Governance Toolkit**

## **Scenario Context**

Acado Corp wants to implement **continuous, automated cost governance** across all AWS accounts.
Today, engineers manually review resources, cost anomalies, and cleanup opportunities — but:

* Idle resources accumulate
* Rightsizing recommendations remain unimplemented
* Budgets and alerts are inconsistently configured
* Orphaned snapshots and volumes grow unchecked
* No automation ensures weekly/monthly governance

Engineering leadership mandates the creation of a **FinOps Automation Toolkit** that will:

✔ Scan AWS accounts weekly for cost inefficiencies
✔ Publish findings to Slack / email / dashboards
✔ Trigger automated remediation workflows
✔ Enforce budgets and alerts
✔ Deploy governance tools across environments using IaC

This final module transforms everything learned in Modules 1–7 into a **repeatable, automated FinOps framework.**

---

# **Module 8 Deliverables**

By the end of this module, participants will build:

✔ A Lambda-based resource analyzer
✔ Budget + alerting automation
✔ Rightsizing + resource cleanup workflows
✔ IaC templates (Terraform/CloudFormation) for deployment
✔ A mini **"FinOps Governance Framework"** used across environments

---

# **Lab 1 — Build a Lambda-Based Idle Resource Detector**

## **Objective**

Develop a Lambda function that runs weekly to detect:

* Idle EC2 instances
* Underutilized RDS instances
* Unused EBS volumes
* Idle Load Balancers
* Unused or low-access S3 buckets
* Underutilized Lambda functions
* ECS/EKS services with near-zero CPU/memory usage

---

## **Step 1 — Create IAM Role**

```bash
aws iam create-role --role-name acado-finops-role \
  --assume-role-policy-document file://trust.json
```

Attach read-only permissions:

```bash
aws iam attach-role-policy \
  --role-name acado-finops-role \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

---

## **Step 2 — Write the Lambda Function**

`finops-detector.py`:

```python
import boto3, datetime

ec2 = boto3.client("ec2")
cloudwatch = boto3.client("cloudwatch")
rds = boto3.client("rds")
elbv2 = boto3.client("elbv2")

def check_ec2_idle(instance_id):
    metrics = cloudwatch.get_metric_statistics(
        Namespace="AWS/EC2",
        MetricName="CPUUtilization",
        Dimensions=[{"Name": "InstanceId", "Value": instance_id}],
        StartTime=datetime.datetime.utcnow() - datetime.timedelta(days=7),
        EndTime=datetime.datetime.utcnow(),
        Period=3600,
        Statistics=["Average"]
    )
    datapoints = metrics.get("Datapoints", [])
    avg = sum([p["Average"] for p in datapoints]) / (len(datapoints) or 1)
    return avg < 5

def lambda_handler(event, context):
    findings = []

    # EC2
    instances = ec2.describe_instances()
    for r in instances["Reservations"]:
        for inst in r["Instances"]:
            if check_ec2_idle(inst["InstanceId"]):
                findings.append({
                    "type": "EC2",
                    "resource": inst["InstanceId"],
                    "issue": "Idle for >7 days"
                })

    # TODO: Add RDS, EBS, Lambda, ECS/EKS Checks

    return {"findings": findings}
```

Deploy:

```bash
aws lambda create-function \
  --function-name acado-finops-detector \
  --handler finops-detector.lambda_handler \
  --runtime python3.12 \
  --zip-file fileb://finops.zip \
  --role arn:aws:iam::<acc>:role/acado-finops-role
```

---

## **Step 3 — Schedule Weekly Execution**

```bash
aws events put-rule \
  --name weekly-finops-check \
  --schedule-expression "cron(0 3 ? * MON *)"
```

Attach Lambda:

```bash
aws events put-targets \
  --rule weekly-finops-check \
  --targets "Id"="1","Arn"="<lambda-arn>"
```

**Outcome:**
A centralized “Cost Scanner” that detects weekly inefficiencies across the environment.

---

# **Lab 2 — Budgets & Alerts Automation (Email / SNS / Slack)**

## **Objective**

Configure **automated cost alerts** that activate when forecasts exceed thresholds.

---

## **Step 1 — Create a Monthly Budget**

```bash
aws budgets create-budget \
  --account-id <acc> \
  --budget file://monthly-budget.json
```

`monthly-budget.json`:

```json
{
  "BudgetName": "Acado-Monthly-Cost",
  "BudgetLimit": { "Amount": 5000, "Unit": "USD" },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}
```

---

## **Step 2 — Attach Alert Thresholds**

```bash
aws budgets create-notification-with-subscribers \
  --account-id <acc> \
  --budget-name Acado-Monthly-Cost \
  --notification file://notify80.json \
  --subscribers file://finops-subscribers.json
```

`notify80.json`:

```json
{
  "NotificationType": "FORECASTED",
  "ComparisonOperator": "GREATER_THAN",
  "Threshold": 80
}
```

`finops-subscribers.json`:

```json
[
  { "SubscriptionType": "EMAIL", "Address": "finops@acado.com" },
  { "SubscriptionType": "SNS", "Address": "arn:aws:sns:us-east-1:123456789012:acadosns" }
]
```

---

## **Step 3 — Add Slack Alert Integration**

Create SNS → Lambda → Slack webhook bridge:

```python
import json, urllib3
http = urllib3.PoolManager()

def lambda_handler(event, context):
    http.request(
        "POST",
        "https://hooks.slack.com/services/<id>",
        body=json.dumps({"text": json.dumps(event)}),
        headers={"Content-Type": "application/json"}
    )
```

---

# **Lab 3 — Automated Rightsizing & Resource Cleanup**

## **Objective**

Create automation workflows to:

* Rightsize EC2
* Delete unused EBS volumes
* Remove old snapshots
* Clean up idle Load Balancers
* Optimize Lambda memory settings
* Identify underutilized RDS

---

## **Step 1 — Integrate Compute Optimizer**

```bash
aws compute-optimizer get-ec2-instance-recommendations > reco.json
```

Use Lambda to trigger remediation via SSM or EC2 APIs.

---

## **Step 2 — EBS Volume Cleanup**

```python
volumes = ec2.describe_volumes(Filters=[{"Name": "status", "Values": ["available"]}])
for vol in volumes["Volumes"]:
    ec2.delete_volume(VolumeId=vol["VolumeId"])
```

---

## **Step 3 — Remove Old Snapshots**

```python
for s in ec2.describe_snapshots(OwnerIds=["self"])["Snapshots"]:
    if s["StartTime"] < cutoff_date:
        ec2.delete_snapshot(SnapshotId=s["SnapshotId"])
```

---

## **Step 4 — Auto-Rightsize Lambda Memory**

Based on Power Tuning output:

```bash
aws lambda update-function-configuration \
  --function-name acado-process \
  --memory-size 1024
```

---

# **Lab 4 — Deploy the Governance Framework via IaC**

## **Objective**

Package the FinOps automation toolkit into Infrastructure-as-Code so it can be deployed across:

* Dev
* QA
* Staging
* Production

---

## **Option A — Terraform (Preferred)**

### Suggested structure:

```
modules/
  finops-lambda/
  budgets/
  notifier/
  cleanup/
environments/
  prod/
    main.tf
  dev/
    main.tf
```

Example:

```hcl
resource "aws_lambda_function" "finops" {
  function_name = "acado-finops-detector"
  role          = aws_iam_role.finops_role.arn
  handler       = "finops-detector.lambda_handler"
  runtime       = "python3.12"
  filename      = "finops.zip"
}
```

---

## **Option B — CloudFormation**

Create a single unified stack containing:

* Lambda functions
* EventBridge schedules
* IAM roles
* SNS topics
* Budgets & notifications

---

# **Module 8 Final Challenge — Build a Complete FinOps Automation Framework**

### **Goal:**

Create a fully functional **self-running automated cost governance system**.

### **Required Outcomes:**

✔ Weekly idle resource report
✔ Automated cleanup of unused assets
✔ Automatic EC2/Lambda rightsizing
✔ Monthly budget with alerts
✔ Slack/email notifications
✔ IaC-driven deployment
✔ Estimated annual savings ≥ **30%**

---

# **Module 8 Deliverables**

Participants successfully deliver:

* FinOps resource analyzer Lambda
* Budget + alert automation
* Rightsizing + cleanup workflows
* Terraform/CloudFormation governance stack
* Weekly Slack/email governance summary
* A working “Acado FinOps Governance Toolkit”

