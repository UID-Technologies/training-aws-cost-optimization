
# **Module 6 — Compute Optimization: Advanced Real Scenarios**

### *(EC2, Auto Scaling, Predictive Scaling, Compute Optimizer, Automated Remediation)*

This module focuses on advanced compute optimization using real enterprise scenarios.
Participants already know EC2 basics and standard Auto Scaling.
Now they will learn how to:

* Reduce EC2 costs by 30–40% using predictive scaling
* Identify rightsizing opportunities using Compute Optimizer
* Implement automated remediation workflows (Lambda + EventBridge)
* Validate cost savings using CloudWatch and Cost Explorer

Every step is done from the **AWS Management Console**.

---

# **1. Module Overview**

Participants will learn to:

* Analyze compute utilization patterns
* Enable predictive scaling to match future demand
* Interpret Compute Optimizer recommendations
* Build automated, self-healing compute environments
* Measure cost reductions using Cost Explorer

This module simulates real FinOps and cloud operations workflows.

---

# **2. Learning Objectives**

### Predictive Scaling

* Predict future CPU load
* Scale out *before* load arrives
* Prevent overprovisioning and reduce EC2 runtime cost

### Compute Optimizer Rightsizing

* Detect oversized instances
* Identify cost savings with recommended instance types
* Validate findings via CloudWatch metrics

### Automated Compute Remediation

* Create EventBridge rules for recommendations
* Trigger Lambda to adjust launch templates
* Automatically reduce scale-out capacity and instance size

### Cost Validation

* Measure improvements in CPU utilization
* Validate fewer running instance-hours
* View cost reduction in Cost Explorer

---

# **3. Prerequisites**

### AWS Services Needed

* EC2
* Auto Scaling
* Compute Optimizer
* CloudWatch
* Lambda
* EventBridge
* IAM

### Environment

* Windows OS
* Web browser
* AWS Console

### Sample AMI

* **Amazon Linux 2023** or **Amazon Linux 2**

---

# **4. Initial Setup (Console Only)**

## Step 1 — Enable Compute Optimizer

1. Go to **AWS Console → Compute Optimizer**
2. Click **Enable**
3. Confirm activation

Compute Optimizer begins analyzing instances (full data may take 12–24 hours, but Auto Scaling Lab workloads will accelerate data collection).

---

## Step 2 — Create an Oversized Auto Scaling Group

Purpose: You intentionally create an *inefficient* ASG, so optimizations are visible.

1. Go to **EC2 Console → Auto Scaling Groups → Create Auto Scaling Group**
2. Create a **Launch Template**

   * Instance type: **t3.medium** (intentionally oversized)
   * AMI: Amazon Linux
   * Key pair: optional
   * Security group: allow HTTP
3. Create Auto Scaling Group

   * VPC: default
   * Subnets: choose **at least 2 AZs**
   * Desired capacity: **2**
   * Minimum: **2**
   * Maximum: **6**
4. Scaling Policy

   * Choose **Target Tracking Scaling Policy**
   * Metric: **CPU Utilization**
   * Target value: **50%**

This creates an ASG ready for optimization.

---

# **LAB 1 — Auto Scaling Deep Dive with Predictive Scaling**

### Objective

Enable Predictive Scaling to proactively scale EC2 instances based on forecasted demand.

---

## Step 1 — Prepare Time-Series CPU Patterns

Predictive scaling needs variation in load to learn patterns.

In real enterprise systems this data already exists.
For the lab, your ASG will naturally create baseline utilization over time.
You do **not** need to generate workload manually.

Wait 20–60 minutes for baseline CPU metrics to accumulate in CloudWatch.

---

## Step 2 — Enable Predictive Scaling

1. Go to **EC2 Console → Auto Scaling Groups**
2. Select your ASG
3. Go to **Automatic Scaling → Add Policy**
4. Choose **Predictive scaling**
5. Metric specification: **CPUUtilization**
6. Forecasting settings:

   * Forecast data points: **48 hours**
7. Scaling behavior:

   * **Maximize availability**
8. Cooldown: 300 seconds
9. Save

Predictive scaling now begins analyzing historical data and forecasting peak times.

---

## Step 3 — Monitor Predictive Scaling Behavior

After activation:

1. Go to ASG → **Activity**
2. Look for predictions and proactive scale-out actions
3. Go to **CloudWatch → Metrics → Auto Scaling**
4. Open metric:

   * **Predictive scaling forecast**

You will now see two lines:

* Actual CPU consumption
* Forecasted future demand

Expected outcomes:

* ASG scales out *before* CPU spikes
* ASG scales in more aggressively after demand drops
* Idle compute time decreases → cost savings

---

## Step 4 — Validate Cost Improvement

1. Go to **Cost Explorer → EC2 Instance Usage**
2. Compare:

   * Before predictive scaling
   * After predictive scaling

Look for:

* Fewer unneeded scale-up events
* Reduced On-Demand instance-hours
* Better CPU utilization percentage

Target improvement: **~30% cost reduction**

---

## LAB 1 Checklist

| Task                       | Done | Verified | Screenshot |
| -------------------------- | ---- | -------- | ---------- |
| Baseline ASG created       |      |          |            |
| Predictive scaling enabled |      |          |            |
| Forecast graphs observed   |      |          |            |
| Activity logs validated    |      |          |            |
| Cost Explorer improvements |      |          |            |

---

# **LAB 2 — Rightsizing Compute Using Compute Optimizer**

### Objective

Use Compute Optimizer & CloudWatch to identify oversized instances and optimize them.

---

## Step 1 — Review Compute Optimizer Recommendations

1. Open **Compute Optimizer Console**
2. Select **EC2 instances**
3. Look for categories:

   * Over-provisioned
   * Under-provisioned
   * Optimized

Select an instance from the ASG.

You will see a details panel with:

* Current instance type
* Recommended instance type(s)
* Estimated monthly savings
* Performance risk level

Example recommendation:

* Current: **t3.medium**
* Recommended: **t3.small** (40% cheaper)
* Or: **t4g.small** (60% cheaper if ARM is supported)

---

## Step 2 — Validate With CloudWatch Metrics

1. Open **CloudWatch Console → Metrics → EC2 → Per-Instance Metrics**
2. Select the instance
3. Review:

   * CPUUtilization
   * NetworkIn / NetworkOut
   * DiskReadBytes / DiskWriteBytes

Expected findings:

* CPU < 20% average
* Network low
* Storage low

This proves overprovisioning.

---

## Step 3 — Create an Automated Rightsizing Workflow

Purpose: Automatically resize instances based on Compute Optimizer events.

### Step 3A — Create EventBridge Rule

1. Go to **EventBridge Console → Rules → Create rule**
2. Name:

   ```
   AutoRightsizeRule
   ```
3. Event pattern:
   Choose **AWS Services** → **Compute Optimizer** →
   Event type: **RecommendationCreated**

Save rule.

### Step 3B — Create a Lambda Function (Console)

1. Go to **Lambda Console → Create function**
2. Author from scratch
3. Name:

   ```
   AutoRightsizeEC2
   ```
4. Runtime: Python 3.9
5. Role: basic Lambda permissions

Paste logic (high-level workflow):

* Parse recommendation
* Modify appropriate launch template
* Trigger ASG instance refresh

Note:
Since CLI cannot be used, the lab focuses on *workflow understanding*, not actual execution.

---

## Step 3C — Connect Lambda to EventBridge

1. Open EventBridge rule
2. Add target → Lambda Function → `AutoRightsizeEC2`
3. Save configuration

Now when Compute Optimizer publishes a new recommendation, the Lambda will run.

---

## Step 4 — Validate Automated Remediation

1. Open **EventBridge → Monitoring → Recent Invocations**
2. Confirm Lambda was triggered
3. Open **Lambda → Monitor → Logs**
4. Confirm recommendation was received

ASG will update itself based on the automated workflow design (conceptual lab purpose).

---

## LAB 2 Checklist

| Task                               | Done | Verified | Screenshot |
| ---------------------------------- | ---- | -------- | ---------- |
| Checked Compute Optimizer findings |      |          |            |
| Validated CloudWatch metrics       |      |          |            |
| Created EventBridge rule           |      |          |            |
| Created Lambda function            |      |          |            |
| Linked rule → Lambda               |      |          |            |
| Validated automated workflow       |      |          |            |

---

# **Module 6 Final Challenge — Reduce Compute Cost by 30–40%**

Students must combine all optimizations to produce measurable cost savings.

---

## Challenge Requirements

### Predictive Scaling

* Forecasting enabled
* Proactive scale-out observed
* Scaling events optimized

### Rightsizing

* Compute Optimizer used
* At least one rightsizing recommendation documented

### Automation Workflow

* EventBridge → Lambda correction pipeline sketched
* Trigger flow validated

### Cost Improvement

* Cost Explorer screenshots showing improvements
* CPU utilization improvements
* Fewer idle compute hours

---

## Deliverables

### Before Optimization

* ASG configuration
* Instance type
* CPU metrics
* Cost Explorer baseline

### After Optimization

* Predictive scaling graphs
* New ASG behavior logs
* Rightsizing data
* Final cost reduction summary

**Goal: Achieve 30% minimum cost reduction; 40%+ results qualify for Excellence Award.**

---
