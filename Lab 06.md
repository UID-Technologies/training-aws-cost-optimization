# **Module 6 — Compute Optimization: Advanced Real-World Scenarios**

## **Scenario Context**

Acado Corp runs a wide range of compute-intensive workloads on EC2 Auto Scaling Groups (ASGs), powering:

* A high-throughput onboarding API
* Several ETL and batch data processing workloads
* Mixed-instance-family microservices
* 24×7 analytics workloads with highly variable demand

Recent FinOps review highlighted:

* **ASGs remain over-provisioned** during low traffic windows
* Scaling policies are outdated (simple CPU-based rules)
* CloudWatch alarms are conservative → slow scale-in
* Instances are oversized with low CPU/Memory usage
* Compute Optimizer recommendations haven't been implemented
* No automation exists for periodic rightsizing

### **Goal:**

Achieve **30–45% cost reduction** via predictive scaling and automated rightsizing.

This module includes **two hands-on labs** that simulate enterprise-grade compute optimization challenges.

---

# **Module Prerequisites**

### **AWS Requirements**

* IAM permissions:
  `autoscaling:*`, `cloudwatch:*`, `compute-optimizer:*`, `ssm:*`, `ec2:*`, `lambda:*`
* Auto Scaling Group already deployed
* Application Load Balancer available
* Compute Optimizer enabled

### **Tools**

* AWS CLI
* CloudWatch console access
* Ability to upload JSON files for scaling policies

---

# **Initial Setup (Recommended)**

Before starting the lab:

1. Identify an existing ASG such as `acado-api-asg`
2. Confirm CloudWatch metrics exist for CPU, RequestCount, etc.
3. Ensure the ASG has dynamic scaling disabled or minimized (to observe lab behavior clearly)
4. Install CloudWatch Agent on EC2 if memory metrics are needed

---

# **Lab 1 — Auto Scaling Deep Dive: Reduce Cost by 30% Using Predictive Scaling**

## **Objective**

Configure **Predictive Scaling** so the Auto Scaling Group scales **before** traffic changes occur and scales **down** aggressively during low-traffic periods.

Predictive scaling uses **machine learning forecasting** from AWS to reduce compute over-allocation.

---

## **Lab Prerequisites**

* An ASG connected to an ALB
* Target tracking and dynamic policies disabled (or allowed to be replaced)
* CloudWatch metrics for request count, CPU utilization, latency

---

## **Step 1 — Identify an Over-Provisioned ASG**

List ASGs:

```bash
aws autoscaling describe-auto-scaling-groups
```

Choose one (example):

```
acado-api-asg
```

Review CPU trends:

```bash
aws cloudwatch get-metric-statistics \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --period 300 \
  --statistics Average \
  --start-time 2025-01-01T00:00:00Z \
  --end-time 2025-01-02T00:00:00Z
```

Typical findings:

* Average CPU = **12–15%**
* ASG min capacity = **5 instances**
* Only **3 instances** required during peak

➡ **Clear over-provisioning**.

---

## **Step 2 — Enable Predictive Scaling**

Create predictive scaling configuration:

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name acado-api-asg \
  --policy-name predictive-scaling \
  --policy-type PredictiveScaling \
  --predictive-scaling-configuration file://predictive.json
```

Example **predictive.json**:

```json
{
  "MetricSpecifications": [
    {
      "TargetValue": 50,
      "CustomizedLoadMetricSpecification": {
        "MetricName": "RequestCount",
        "Namespace": "AWS/ApplicationELB",
        "Dimensions": [
          { "Name": "LoadBalancer", "Value": "app/acado-lb" }
        ],
        "Statistic": "Sum"
      }
    }
  ],
  "Mode": "ForecastAndScale"
}
```

✔ ASG now uses **forecasted demand** for scaling.
✔ Ideal for workloads with hourly or daily patterns.

---

## **Step 3 — Add Target Tracking Scaling (Real-Time Reaction)**

Predictive scaling handles forecasting, but **moment-to-moment traffic** must still be managed.

Apply target tracking:

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name acado-api-asg \
  --policy-name target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration file://targettracking.json
```

Example **targettracking.json**:

```json
{
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ASGAverageCPUUtilization"
  },
  "TargetValue": 40
}
```

---

## **Step 4 — Validate Scaling Behavior**

Check scaling events:

```bash
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name acado-api-asg
```

Expected behavior:

* **4–6 AM off-peak:** ASG scales down from 5 → 2 instances
* **Traffic spike at 9 AM:** ASG scales up **before** API traffic grows
* Fewer surprise throttles / cold starts
* Compute footprint reduced during low demand

---

## **Expected Optimization Outcome**

* **~35% off-peak cost reduction**
* Predictive scaling eliminates manual tuning
* Auto-provisioning ahead of load prevents latency issues
* Efficient scaling across weekday/weekend patterns

---

# **Lab 2 — Automated Compute Rightsizing Using Compute Optimizer + CloudWatch**

## **Objective**

Use **Compute Optimizer** to detect rightsizing opportunities, verify them with CloudWatch, and automate instance resizing using Lambda or SSM.

---

## **Lab Prerequisites**

* Compute Optimizer enabled
* EC2 instances running for >14 days
* IAM permissions for resizing EC2 instances
* CloudWatch metrics available

---

## **Step 1 — Enable Compute Optimizer (If Needed)**

```bash
aws compute-optimizer update-enrollment-status --status Active
```

Verify:

```bash
aws compute-optimizer get-enrollment-status
```

---

## **Step 2 — Fetch Rightsizing Recommendations**

```bash
aws compute-optimizer get-ec2-instance-recommendations > reco.json
```

Inspect sample output:

```json
"currentInstanceType": "m5.4xlarge",
"recommendedInstanceType": "m5.2xlarge",
"finding": "Overprovisioned"
```

Common flags:

* `"Underprovisioned"`
* `"Overprovisioned"`
* `"Optimized"`

---

## **Step 3 — Validate Findings Using CloudWatch Metrics**

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-12345 \
  --statistics Average \
  --period 300 \
  --start-time 2025-01-01T00:00:00Z \
  --end-time 2025-01-02T00:00:00Z
```

Validation criteria:

* **CPU < 20%** consistently → oversizing
* Memory utilization < 40% (requires CW Agent)
* Network & IO low → oversizing

This step confirms that Compute Optimizer didn’t flag a burst anomaly incorrectly.

---

## **Step 4 — Automate Rightsizing**

### **Option A — Lambda Automation**

```python
import boto3

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    instance_id = event["InstanceId"]
    new_type = event["NewType"]

    ec2.stop_instances(InstanceIds=[instance_id])
    waiter = ec2.get_waiter('instance_stopped')
    waiter.wait(InstanceIds=[instance_id])

    ec2.modify_instance_attribute(
        InstanceId=instance_id,
        InstanceType={'Value': new_type}
    )

    ec2.start_instances(InstanceIds=[instance_id])
    return {"status": "success"}
```

Trigger via EventBridge rule linked to Compute Optimizer findings.

---

### **Option B — SSM Automation (Standard Runbook)**

```bash
aws ssm start-automation-execution \
  --document-name AWS-ResizeInstance \
  --parameters InstanceId=i-12345,InstanceType=m5.large
```

SSM provides:

* Logs
* Approval workflows
* Versioning
* Rollback options

---

## **Step 5 — Validate Post-Resize Performance**

Key metrics to check:

* CPU now averages **30–60%**
* Memory steady within safe threshold
* Network bandwidth not saturated
* No CPU credit exhaustion (for T-series instances)
* Latency remains stable

---

# **Expected Optimization Outcome**

* Instance sizing improved → **20–40% compute savings**
* Predictable performance after rightsizing
* Automated workflows reduce operational overhead
* Continuous compute governance across environments

---

# **Module 6 Deliverables**

Participants produce:

* Working predictive scaling configuration
* Combined predictive + target tracking policy
* Compute Optimizer rightsizing report
* CloudWatch metric validation insights
* Automated rightsizing workflow (Lambda or SSM)
* Verified compute cost reduction
