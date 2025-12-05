# **Module 4 — Serverless Architecture Optimization**

## **Scenario Context**

Acado Corp relies heavily on serverless systems to handle ingestion pipelines, user workflows, and background event processing. Recent cost audits highlight major inefficiencies:

* Many Lambdas are **over-provisioned** with excessive memory
* Several functions run **synchronously**, causing compute spikes
* API Gateway REST APIs are still in use despite **higher per-request pricing**
* Blocking workflows cause longer Lambda execution time
* Developers lack visibility into optimal memory/performance settings

### **Goal:**

Reduce overall serverless spend by **25–45%** while ensuring equal or better performance.

This module includes **three hands-on labs** and a **final optimization challenge**.

---

## **Module Prerequisites**

### **AWS Requirements**

* IAM permissions:
  `lambda:*`, `apigateway:*`, `stepfunctions:*`, `cloudwatch:*`, `events:*`, `sqs:*`
* AWS CLI configured
* A working Lambda function (e.g., `acado-process-data`)
* API Gateway REST API for testing migration
* Step Functions permissions to deploy Lambda Power Tuning state machine

### **Tools Needed**

* AWS CLI
* Code editor (VS Code recommended)
* cURL or Postman for API testing

---

## **Initial Setup (Optional but Recommended)**

Before starting the labs:

1. Create a sample Lambda function:
   `acado-process-data`
2. Create a simple REST API in API Gateway
3. Ensure CloudWatch Logs are enabled
4. Ensure at least one Lambda workflow has downstream calls (for Lab 2)

---

# **Lab 1 — Lambda Cost Profiling with AWS Lambda Power Tuning**

## **Objective**

Optimize Lambda execution cost using the **Lambda Power Tuning** tool to determine the ideal memory configuration for price-performance.

---

## **Prerequisites**

* A deployed Lambda function
* Step Functions enabled in the region
* IAM privileges for SAR, Lambda, and Step Functions

---

## **Step 1 — Deploy the Lambda Power Tuning State Machine**

Use the Serverless Application Repository:

```bash
aws serverlessrepo create-cloud-formation-template \
  --application-id arn:aws:serverlessrepo:us-east-1:451282441545:applications/lambda-power-tuning \
  --semantic-version 4.1.0
```

Deploy the resulting CloudFormation template using:

* AWS Console **or**
* `aws cloudformation deploy`

---

## **Step 2 — Run a Power Tuning Execution**

Prepare the tuning input:

```json
{
  "lambdaARN": "arn:aws:lambda:us-east-1:123456789012:function:acado-process-data",
  "powerValues": [128, 256, 512, 1024, 1536, 3008],
  "num": 50,
  "payload": "{\"action\":\"test\"}"
}
```

Run the execution:

```bash
aws stepfunctions start-execution \
  --state-machine-arn <state-machine-arn> \
  --input file://power-tuning-input.json
```

---

## **Step 3 — Review the Tuning Report**

Go to:

**AWS Console → Step Functions → Execution Results**

The output shows:

* Duration per memory tier
* Cost per invocation
* Graph comparison
* **Recommended memory allocation**

### Example:

| Memory  | Duration | Cost            |
| ------- | -------- | --------------- |
| 128 MB  | 1900 ms  | High            |
| 512 MB  | 520 ms   | Medium          |
| 1024 MB | 260 ms   | **Lowest Cost** |

### **Outcome**

Update Lambda memory settings accordingly for both cost and performance improvements.

---

# **Lab 2 — Convert Synchronous Workloads to Event-Driven Architecture**

## **Objective**

Transform blocking workflows into event-driven pipelines to reduce Lambda runtime, improve performance, and eliminate cascading failures.

---

## **Prerequisites**

* At least two Lambda functions: A, B, C
* Appropriate IAM roles for EventBridge or SQS
* CloudWatch Logs enabled

---

## **Step 1 — Identify a Synchronous Workflow**

Typical current architecture:

```
Client → API Gateway → Lambda A → Lambda B → Lambda C → Response
```

Problems:

* Lambda A must wait for B and C
* High execution time = high cost
* Errors propagate upstream
* Poor scalability

---

## **Step 2 — Introduce EventBridge or SQS for Decoupling**

Create an EventBridge rule:

```bash
aws events put-rule \
  --name acado-serverless-rule \
  --event-pattern file://pattern.json
```

### New architecture:

Lambda A:

* Validates request
* Publishes event
* Returns response immediately

EventBridge triggers:

* Lambda B
* Lambda C
* Additional microservices

### Impact:

✔ Execution time drops from **4 seconds → 200 ms**
✔ Cost drops proportionally
✔ Improved reliability

---

## **Step 3 — Implement Asynchronous Invocation**

Modify Lambda A:

```python
import boto3, json
client = boto3.client("lambda")

def handler(event, context):
    payload = json.dumps(event)
    client.invoke(
        FunctionName="lambda-B",
        InvocationType="Event",  # async
        Payload=payload
    )
    return {"status": "queued"}
```

**Result:**
Lambda A no longer blocks; downstream compute costs now run independently.

---

## **Step 4 — Validate the Event-Driven Workflow**

Check in CloudWatch:

* Lambda B/C invocations
* No throttling
* No DLQ entries
* Latency improvements

---

# **Lab 3 — API Gateway Cost Optimization (REST → HTTP API Migration)**

## **Objective**

Replace expensive REST API endpoints with HTTP API endpoints to reduce cost by **60–72%**.

---

## **Prerequisites**

* Existing REST API in API Gateway
* Lambda integration
* No usage of unsupported REST features (request validators, complex mappings, etc.)

---

## **Step 1 — Identify REST APIs Eligible for Migration**

```bash
aws apigateway get-rest-apis
```

Eligible APIs:

* Use Lambda proxy integration
* Have simple authentication
* Do not rely on complex API Gateway REST features

---

## **Step 2 — Create an Equivalent HTTP API**

```bash
aws apigatewayv2 create-api \
  --name acado-http-api \
  --protocol-type HTTP \
  --target arn:aws:lambda:us-east-1:123456789012:function:acado-handler
```

---

## **Step 3 — Configure JWT or Lambda Authorizer**

Example:

```bash
aws apigatewayv2 create-authorizer \
  --api-id <api-id> \
  --authorizer-type JWT \
  --identity-source '$request.header.Authorization' \
  --jwt-configuration Audience=acado-app,Issuer=https://cognito-idp.us-east-1.amazonaws.com/<pool-id>
```

---

## **Step 4 — Benchmark Pricing**

### REST API

≈ **$3.50 per million requests**

### HTTP API

≈ **$1.00 per million requests**

➡ **Result:** 60–72% cost savings

---

## **Step 5 — Switch Traffic to HTTP API**

Testing:

```bash
curl -X GET <http_api_endpoint>/status
```

Migration methods:

* Weighted Route 53 routing
* Canary rollout
* Gradual traffic shifting

---

# **Lab Challenge — Reduce Serverless Pipeline Cost by 40%**

## **Goal**

Optimize the entire serverless pipeline to achieve **≥40% cost reduction**.

## **Pipeline Components**

* API Gateway REST
* Lambda A → Lambda B → Lambda C
* DynamoDB writes
* S3 archival

## **Allowed Optimization Techniques**

✔ Lambda Power Tuning
✔ Migrate REST → HTTP API
✔ Async invocation (EventBridge/SQS)
✔ Provisioned concurrency tuning
✔ Batch writes for DynamoDB
✔ S3 lifecycle policies

## **Success Criteria**

* Pipeline remains fully functional
* Latency does not increase
* Verified cost reduction ≥ 40%
* Evidence from CloudWatch, Lambda metrics, CUR

---

# **Module 4 Deliverables**

Participants will deliver:

* Optimized Lambda configurations based on tuning
* Event-driven redesign of synchronous workflows
* HTTP API migration for API cost efficiency
* Completed cost optimization challenge
* Serverless cost governance framework
