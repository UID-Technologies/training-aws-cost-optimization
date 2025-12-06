
# **Module 8 — Final Lab: Build an Automated Cost Governance Toolkit**

## **1. Module Overview**

In this final module, you will build a **lightweight, automated cost governance framework** that continuously:

* Monitors cloud usage
* Detects idle or underutilized resources
* Raises alerts proactively
* Performs automated cleanup or rightsizing
* Enforces budgets and spending guidelines
* Operates using serverless, scalable AWS-native tools

This module integrates everything learned in Modules 1–7 and transforms it into a **production-ready cost governance system**.

---

## **2. Learning Objectives**

You will learn to:

### **Automated Cost Detection**

* Detect idle EC2 instances
* Identify unattached EBS volumes
* Identify unused Elastic IPs
* Identify idle RDS instances
* Detect old snapshots consuming storage

### **Cost Governance**

* Create monthly/forecast-based AWS Budgets
* Notify through SNS → Email or Slack
* Monitor spend automatically

### **Optimization Automation**

* Event-driven rightsizing
* Automated cleanup workflows
* Tag governance & compliance checks

### **Governance as Code**

* Deploy budgets, alerts, routers, lambdas, and EventBridge rules
  using:

  * Terraform
  * CloudFormation

---

## **3. Initial Setup Steps**

Before starting the labs, perform these setup tasks.

### **Step 1 — Create a Governance S3 Bucket (using Windows Command Prompt + AWS CLI)**

```cmd
aws s3 mb s3://governance-toolkit-<yourname>
```

This bucket will store:

* JSON reports
* Configuration snapshots
* Optional logs and exported metrics

### **Step 2 — Create IAM Role for Governance Lambdas**

Go to the **IAM Console → Roles → Create Role**
Select: **Lambda** as trusted entity.

Attach the following policies:

| Policy                         | Purpose                |
| ------------------------------ | ---------------------- |
| AmazonEC2ReadOnlyAccess        | Inspect EC2 resources  |
| AmazonRDSReadOnlyAccess        | Inspect RDS databases  |
| AmazonEC2FullAccess (optional) | Stop/delete EC2 or EBS |
| AmazonCloudWatchLogsFullAccess | Write logs             |
| AmazonS3FullAccess             | Export reports         |
| AWSLambdaBasicExecutionRole    | Logging                |

Role Name: **CostGovernanceLambdaRole**

---

# **Lab 1 — Detect Idle or Underutilized AWS Resources**

## **Goal**

Create a Lambda that automatically scans your AWS account for:

* Idle EC2 instances (<5% CPU over 7 days)
* Unused EBS volumes
* Idle RDS instances
* Old snapshots (>30 days)
* Unused Elastic IPs

Produces a JSON report you can save to S3 or trigger cleanup automation.

---

## **Step-by-Step Implementation**

### **Step 1 — Write Lambda Script (Windows Command Prompt)**

Create the file:

```cmd
notepad detect_idle_resources.py
```

Paste the script:

```python
import boto3
import datetime

ec2 = boto3.client('ec2')
cloudwatch = boto3.client('cloudwatch')
rds = boto3.client('rds')

def get_ec2_cpu(instance_id):
    metrics = cloudwatch.get_metric_statistics(
        Namespace='AWS/EC2',
        MetricName='CPUUtilization',
        Dimensions=[{'Name':'InstanceId', 'Value':instance_id}],
        StartTime=datetime.datetime.utcnow() - datetime.timedelta(days=7),
        EndTime=datetime.datetime.utcnow(),
        Period=3600,
        Statistics=['Average']
    )
    dps = metrics.get('Datapoints', [])
    if not dps:
        return None
    return sum(dp['Average'] for dp in dps) / len(dps)

def lambda_handler(event, context):
    findings = []

    # EC2 idle
    for r in ec2.describe_instances()['Reservations']:
        for inst in r['Instances']:
            if inst['State']['Name'] == 'running':
                cpu = get_ec2_cpu(inst['InstanceId'])
                if cpu is not None and cpu < 5:
                    findings.append({
                        'resource': inst['InstanceId'],
                        'type': 'EC2',
                        'issue': 'Idle (CPU <5%)'
                    })

    # EBS unattached
    for vol in ec2.describe_volumes()['Volumes']:
        if len(vol.get('Attachments', [])) == 0:
            findings.append({
                'resource': vol['VolumeId'],
                'type': 'EBS',
                'issue': 'Unattached volume'
            })

    # Unused Elastic IPs
    for addr in ec2.describe_addresses()['Addresses']:
        if 'InstanceId' not in addr:
            findings.append({
                'resource': addr['PublicIp'],
                'type': 'EIP',
                'issue': 'Unassociated Elastic IP'
            })

    # RDS idle
    for db in rds.describe_db_instances()['DBInstances']:
        if db['DBInstanceStatus'] == 'available':
            findings.append({
                'resource': db['DBInstanceIdentifier'],
                'type': 'RDS',
                'issue': 'Needs CPU/connection evaluation'
            })

    return findings
```

### **Step 2 — Zip & Create Lambda**

```cmd
powershell Compress-Archive detect_idle_resources.py function.zip
```

Now deploy:

```cmd
aws lambda create-function ^
 --function-name DetectIdleResources ^
 --runtime python3.9 ^
 --handler detect_idle_resources.lambda_handler ^
 --zip-file fileb://function.zip ^
 --role <IAM_ROLE_ARN>
```

### **Step 3 — Test Lambda**

Go to:

**Lambda Console → DetectIdleResources → Test**

Observe JSON output such as:

```json
[
  {"resource": "i-12345", "type": "EC2", "issue": "Idle (CPU <5%)"},
  {"resource": "vol-87654", "type": "EBS", "issue": "Unattached volume"}
]
```

---

## **Lab 1 Checklist**

| Task                           | ✔ Done | 🔍 Verified | 📸 Screenshot |
| ------------------------------ | ------ | ----------- | ------------- |
| IAM role created               |        |             |               |
| Lambda created & deployed      |        |             |               |
| Test executed successfully     |        |             |               |
| EC2/EBS/RDS/EIP flagged        |        |             |               |
| JSON exported to S3 (optional) |        |             |               |

---

# **Lab 2 — Create Custom AWS Budgets & Cost Alerts**

## **Goal**

Set up real-time cost monitoring using AWS Budgets + SNS.

---

## **Step 1 — Create SNS Topic (Windows Command Prompt)**

```cmd
aws sns create-topic --name CostAlertsTopic
```

Subscribe email:

```cmd
aws sns subscribe ^
 --topic-arn <TOPIC_ARN> ^
 --protocol email ^
 --notification-endpoint <your-email>
```

Verify subscription email.

---

## **Step 2 — Create AWS Budget (AWS Console Only)**

Go to:

**Billing → Budgets → Create Budget**

1. Budget Type: **Cost Budget**
2. Amount: e.g. **$50**
3. Period: **Monthly**
4. Alerts:

   * 80% actual
   * 100% actual
   * 100% forecasted

Choose SNS Topic → **CostAlertsTopic**

---

## **Lab 2 Checklist**

| Task                   | ✔ Done | 🔍 Verified | 📸 Screenshot |
| ---------------------- | ------ | ----------- | ------------- |
| SNS topic created      |        |             |               |
| Email confirmed        |        |             |               |
| Budget created         |        |             |               |
| Forecast alert created |        |             |               |

---

# **Lab 3 — Automated Rightsizing & Cleanup**

## **Goal**

Use findings from Lab 1 to automatically:

* Stop idle EC2
* Delete unattached EBS
* Send notifications

---

## **Step 1 — Create Cleanup Lambda**

Create file:

```cmd
notepad cleanup_resources.py
```

Paste:

```python
import boto3

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    findings = event.get('findings', [])
    actions = []

    for f in findings:
        if f['type'] == 'EC2':
            ec2.stop_instances(InstanceIds=[f['resource']])
            actions.append(f"Stopped EC2: {f['resource']}")

        if f['type'] == 'EBS':
            ec2.delete_volume(VolumeId=f['resource'])
            actions.append(f"Deleted EBS Volume: {f['resource']}")

    return {'actions_taken': actions}
```

Zip + deploy:

```cmd
powershell Compress-Archive cleanup_resources.py cleanup.zip
```

```cmd
aws lambda create-function ^
 --function-name CleanupResources ^
 --runtime python3.9 ^
 --handler cleanup_resources.lambda_handler ^
 --zip-file fileb://cleanup.zip ^
 --role <IAM_ROLE_ARN>
```

---

## **Step 2 — Create Scheduled Event (EventBridge)**

```cmd
aws events put-rule ^
 --name DailyCostGovernance ^
 --schedule-expression "cron(0 2 * * ? *)"
```

Add Lambda target:

```cmd
aws events put-targets ^
 --rule DailyCostGovernance ^
 --targets Id="1",Arn="<DetectIdleResourcesLambdaARN>"
```

(Advanced: chain CleanupResources using Step Functions.)

---

## **Lab 3 Checklist**

| Task                   | ✔ | 🔍 | 📸 |
| ---------------------- | - | -- | -- |
| Cleanup Lambda created |   |    |    |
| Scheduled rule created |   |    |    |
| Daily detection runs   |   |    |    |
| Cleanup verified       |   |    |    |

---

# **Lab 4 — Governance as Code (Terraform or CloudFormation)**

## **Goal**

Deploy budgets, alerts, lambdas, and rules using IaC so you can:

* Version control
* Reuse across accounts
* Deploy to organizations

---

## **Terraform Example**

Create folder:

```
mkdir governance-iac
cd governance-iac
```

Create `main.tf`:

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_sns_topic" "cost_alerts" {
  name = "CostAlertsTopic"
}

resource "aws_budgets_budget" "monthly_cost" {
  name         = "MonthlyCostBudget"
  budget_type  = "COST"
  limit_amount = "50"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    threshold = 80
    threshold_type = "PERCENTAGE"
    comparison_operator = "GREATER_THAN"
    notification_type   = "ACTUAL"

    subscriber {
      subscription_type = "EMAIL"
      address           = "your-email@example.com"
    }
  }
}
```

Deploy:

```cmd
terraform init
terraform apply
```

---

## **CloudFormation Example**

```cmd
aws cloudformation deploy ^
 --stack-name GovernanceToolkit ^
 --template-file governance.yaml ^
 --capabilities CAPABILITY_IAM
```

---

## **Lab 4 Checklist**

| Task                         | ✔ | 🔍 | 📸 |
| ---------------------------- | - | -- | -- |
| Terraform/CFN folder created |   |    |    |
| IaC deployed                 |   |    |    |
| Resources verified           |   |    |    |

---

