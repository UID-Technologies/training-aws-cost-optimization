

# **MODULE 4 — Serverless Architecture Optimization**


---

# **1. Module Overview**

In this module, learners will optimize serverless workloads using ONLY the AWS Console:

* Tune AWS Lambda cost/performance
* Use AWS Step Functions to perform **Lambda Power Tuning**
* Convert synchronous workflows into efficient **event-driven pipelines** using SQS
* Optimize API cost by migrating from **REST API → HTTP API**
* Achieve **35–50% total cost reduction** in serverless operations

No CLI, no PowerShell, no CloudShell — **100% Console-Based**.

---

# **2. Learning Objectives**

By the end of this module, learners will be able to:

### ✔ Profile Lambda performance & cost

Using a manually built **Lambda Power Tuner** state machine.

### ✔ Determine optimal Lambda memory

by evaluating multiple memory configurations.

### ✔ Build asynchronous workflows

Using Amazon SQS + Lambda triggers.

### ✔ Optimize API Gateway cost

by switching from REST API to HTTP API (up to 70% cheaper).

### ✔ Implement real-world cost reduction strategies

in serverless applications.

---

# **3. Prerequisites**

* AWS Account (Free Tier compatible)
* Region: **us-east-1**
* IAM permissions:

  * Lambda
  * Step Functions
  * CloudFormation (not required here, but helpful)
  * API Gateway
  * SQS
  * CloudWatch Logs
  * IAM Role creation

Workstation: **Windows with a browser only**

---

# **4. Initial Setup (Do Once)**

---

## **Step 1 — Create S3 Bucket for Artifacts (Console Only)**

1. Open **AWS Console → S3**
2. Click **Create bucket**
3. Name:
   `serverless-cost-lab-<yourname>`
4. Region: **us-east-1**
5. Leave defaults
6. Click **Create bucket**

**Checklist**

* [ ] S3 bucket created
* [ ] Screenshot captured

---

## **Step 2 — Create Baseline Lambda Function (ComputeLambda)**

This Lambda function will be used throughout the optimization labs.

---

### **2.1 — Create Lambda Function**

1. Go to **AWS Console → Lambda**
2. Click **Create function**
3. Choose **Author from scratch**
4. Name:
   `ComputeLambda`
5. Runtime: **Node.js 18.x**
6. Permissions:
   ✔ *Create a new role with basic Lambda permissions*
7. Click **Create function**

---

### **2.2 — Add CPU-Heavy Code for Profiling**

In the Code Editor:

Replace code with:

```javascript
exports.handler = async () => {
    let total = 0;
    for (let i = 0; i < 30_000_000; i++) {
        total += i;
    }
    return {
        statusCode: 200,
        body: JSON.stringify({ total })
    };
};
```

Click **Deploy**.

---

### **2.3 — Test the Function**

1. Click **Test**
2. Create new test event → keep default `{}`
3. Execute
4. Note the **Duration** (baseline)

**Checklist**

* [ ] ComputeLambda created
* [ ] Code deployed
* [ ] Test succeeded
* [ ] Baseline time recorded

---



# **SERVERLESS LAB 1 — Lambda Cost Profiling Using Step Functions Power Tuning**



Since SAR and one-click deploy do not work in all accounts, we use the **guaranteed working** console-only manual method.

---

# **Step 1 — Create a Power Tuning Step Function Manually**

1. Open **AWS Console → Step Functions**
2. Click **Create state machine**
3. Choose:
   ✔ **Author with code** (Standard workflow)

---

# **Step 2 — Paste the Working Lambda Power Tuning Definition**

Delete any existing code and paste this **validated definition**:

```json
{
  "Comment": "AWS Lambda Power Tuning - Manual Version Without ArraySort",
  "StartAt": "Init",
  "States": {
    "Init": {
      "Type": "Pass",
      "ResultPath": "$.settings",
      "Parameters": {
        "lambdaArn.$": "$.lambdaArn",
        "powerValues.$": "$.powerValues",
        "num.$": "$.num",
        "payload.$": "$.payload"
      },
      "Next": "MapStates"
    },
    "MapStates": {
      "Type": "Map",
      "ItemsPath": "$.settings.powerValues",
      "Parameters": {
        "value.$": "$$.Map.Item.Value",
        "lambdaArn.$": "$.settings.lambdaArn",
        "num.$": "$.settings.num",
        "payload.$": "$.settings.payload"
      },
      "Iterator": {
        "StartAt": "RunPowerTest",
        "States": {
          "RunPowerTest": {
            "Type": "Task",
            "Resource": "arn:aws:states:::lambda:invoke",
            "Retry": [
              {
                "ErrorEquals": [
                  "Lambda.ServiceException",
                  "Lambda.AWSLambdaException",
                  "Lambda.SdkClientException"
                ],
                "IntervalSeconds": 2,
                "MaxAttempts": 6,
                "BackoffRate": 2
              }
            ],
            "Parameters": {
              "FunctionName.$": "$.lambdaArn",
              "Payload": {
                "power.$": "$.value",
                "num.$": "$.num",
                "payload.$": "$.payload"
              }
            },
            "ResultSelector": {
              "value.$": "$.Payload.value",
              "duration.$": "$.Payload.duration",
              "cost.$": "$.Payload.cost"
            },
            "End": true
          }
        }
      },
      "Next": "Done"
    },
    "Done": {
      "Type": "Succeed"
    }
  }
}
```

---

# **Step 3 — Permissions**

Under *Execution role*:

✔ Select **Create a new role for this state machine**

Click **Create state machine**

---

# **Step 4 — Run the Power Tuning Workflow**

1. Open your new state machine
2. Click **Start Execution**
3. Enter:

```json
{
  "lambdaArn": "arn:aws:lambda:us-east-1:<account-id>:function:ComputeLambda",
  "powerValues": [128, 256, 512, 1024],
  "num": 10,
  "payload": {}
}
```

Click **Start Execution**

---

# **Step 5 — Analyze Results**

In the **Output** tab:

You will see something like:

```json
"MapStates": [
  { "value": 128, "duration": 1200, "cost": 0.000024 },
  { "value": 256, "duration": 800, "cost": 0.000020 },
  { "value": 512, "duration": 450, "cost": 0.000018 },
  { "value": 1024, "duration": 350, "cost": 0.000022 }
]
```

Identify:

* The memory with **lowest cost**
* The memory with best cost:duration ratio (usually **512 MB**)

---

# **Step 6 — Update Lambda Memory Accordingly**

1. Open **ComputeLambda**
2. Go to **Configuration → General configuration**
3. Click **Edit**
4. Set memory to recommended value (e.g., 512 MB)
5. Save

Re-run the test → note performance improvement.

---

###  Lab 1 Checklist

* [ ] Power Tuning state machine created
* [ ] Execution successful
* [ ] Results analyzed
* [ ] Optimal memory selected
* [ ] Lambda memory updated
* [ ] Before/after duration captured

---



# **SERVERLESS LAB 2 — Convert Synchronous Workload → Event-Driven Architecture**



## **Objective**

Remove blocking synchronous Lambda calls and replace them with **SQS → Lambda** asynchronous processing.

---

## **Step 1 — Create SQS Queue**

1. Go to **SQS → Create queue**
2. Type: **Standard**
3. Name: `compute-async-queue`
4. Create

---

## **Step 2 — Create Worker Lambda (AsyncComputeLambda)**

1. **Lambda → Create function → Author from scratch**
2. Name: `AsyncComputeLambda`
3. Runtime: Node.js 18.x
4. Create role with basic permissions

Replace code:

```javascript
exports.handler = async (event) => {
    for (const record of event.Records) {
        let total = 0;
        for (let i = 0; i < 20_000_000; i++) { total += i; }
        console.log("Task:", record.body, "Total:", total);
    }
};
```

Click **Deploy**

---

## **Step 3 — Add SQS Trigger**

1. Open **AsyncComputeLambda → Configuration → Triggers**
2. Add trigger → choose **SQS**
3. Select `compute-async-queue`
4. Batch size: **5**
5. Add

---

## **Step 4 — Send Test Messages**

SQS → queue → **Send and receive messages** → send:

* `task1`
* `task2`
* `task3`

Open **CloudWatch Logs** for AsyncComputeLambda to verify.

---

### ⭐ Lab 2 Checklist

* [ ] Queue created
* [ ] Worker Lambda created
* [ ] SQS → Lambda trigger added
* [ ] Messages processed asynchronously
* [ ] Logs verified

---



# **SERVERLESS LAB 3 — API Gateway Cost Optimization**



## **Goal**

Compare cost between **REST API** and **HTTP API** and migrate to the cheaper alternative.

---

## **Step 1 — Create REST API (Baseline)**

1. API Gateway → REST API → Build
2. Create new API (“ComputeRestAPI”)
3. Create Resource `/compute`
4. Method → GET
5. Integration: **ComputeLambda**
6. Deploy API → Stage `prod`
7. Test the endpoint

---

## **Step 2 — Create HTTP API (Optimized)**

1. API Gateway → HTTP APIs → Build
2. Add integration → Lambda → ComputeLambda
3. Route: GET `/compute`
4. Create API
5. Test endpoint

HTTP API cost per million requests ~ $1
REST API cost ~ $3.50

---

## **Step 3 — Migration Decision**

Once confirmed working:

* Keep **HTTP API**
* Optionally delete **REST API**

---

### ⭐ Lab 3 Checklist

* [ ] REST API created
* [ ] HTTP API created
* [ ] Both tested
* [ ] Cost difference understood
* [ ] REST API removed (optional)

---



# **SERVERLESS LAB 4 — Final Cost Optimization Challenge**

## **Objective**

Apply all optimizations to reduce total cost by **≥40%**

---

## **Tasks**

1. Tune Lambda using Power Tuning results
2. Switch sync calls → async SQS
3. Migrate REST → HTTP API
4. Minimize logs, reduce retries
5. Capture **before vs after cost** in Cost Explorer

---

## **Deliverables**

* Power Tuning output
* Updated memory setting
* SQS async pipeline screenshot
* REST → HTTP API migration evidence
* CloudWatch duration comparison
* Cost Explorer impact screenshots

---

