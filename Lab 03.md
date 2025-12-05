
# **Module 3 — ECS / EKS Cost Optimization: Containers at Scale**

## **Module Objective**

By the end of this module, participants will:

* Optimize ECS/EKS compute spend through right-sizing, Spot usage, autoscaling tuning, and intelligent provisioning
* Migrate workloads from EC2-based ECS tasks to Fargate Spot
* Analyze CPU/memory usage for container rightsizing using Container Insights
* Tune EKS Cluster Autoscaler for efficiency during fluctuations
* Implement Karpenter for Spot-driven, cost-aware provisioning
* Achieve **35–40% compute cost reduction** without impacting SLOs

---

## **Module Prerequisites**

### **AWS Requirements**

* Admin or PowerUser privileges
* AWS CLI installed and configured
* kubectl configured for EKS cluster
* ECS cluster with sample services deployed
* EKS cluster with Cluster Autoscaler or Karpenter support
* CloudWatch Container Insights enabled (optional but recommended)

### **Tools Needed**

* AWS CLI
* kubectl
* Helm (for Karpenter installation)
* IAM roles for ECS/EKS management

---

## **Initial Setup Required**

Before beginning the labs:

1. Ensure ECS cluster **`acado-ecs-prod`** exists
2. Ensure EKS cluster **`acado-eks`** is accessible using kubectl
3. Ensure CloudWatch Container Insights is enabled (if not, Lab 2 enables it)
4. Ensure you have permissions:

   * `ecs:*`
   * `eks:*`
   * `ec2:*`
   * `iam:*`
   * `cloudwatch:*`

---

# **ECS LAB — Cost Optimization for Elastic Container Service**

---

# **Lab 1 — Migrate ECS Tasks from EC2 Launch Type → Fargate Spot**

## **Objective**

Replace EC2-based ECS workloads with **Fargate Spot** to reduce compute cost by **60–70%** while maintaining reliability.

---

## **Prerequisites**

* ECS cluster using EC2 launch type
* Running ECS service such as `user-service`
* IAM permissions for ECS capacity providers

---

## **Step 1 — Identify Services Running on EC2 Launch Type**

```bash
aws ecs list-services --cluster acado-ecs-prod
```

Describe a specific service:

```bash
aws ecs describe-services \
  --cluster acado-ecs-prod \
  --services user-service
```

Check launch type in the output:

Expected:

```json
"launchType": "EC2"
```

---

## **Step 2 — Create Fargate + Fargate Spot Capacity Provider**

```bash
aws ecs create-capacity-provider \
  --name acado-fargate-spot \
  --auto-scaling-group-provider "managedScaling={status=ENABLED}"
```

Attach capacity providers:

```bash
aws ecs put-cluster-capacity-providers \
  --cluster acado-ecs-prod \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1
```

---

## **Step 3 — Update ECS Service to Use Fargate Spot**

```bash
aws ecs update-service \
  --cluster acado-ecs-prod \
  --service user-service \
  --capacity-provider-strategy capacityProvider=FARGATE_SPOT,weight=1
```

### **Expected Outcome**

✔ ECS tasks shift to **Fargate Spot**
✔ Compute cost reduced by **60–70%**
✔ EC2 capacity no longer required

---

# **Lab 2 — Rightsizing ECS Tasks Using Container Insights**

## **Objective**

Detect over-provisioned ECS container resources and optimize CPU/memory to prevent cost waste.

---

## **Prerequisites**

* CloudWatch Container Insights enabled
* ECS services running in production

---

## **Step 1 — Enable Container Insights (If Not Enabled)**

```bash
aws ecs put-account-setting-default \
  --name containerInsights \
  --value enabled
```

### Validation

* Go to CloudWatch → Container Insights → ECS

---

## **Step 2 — Identify Over-Provisioned Tasks**

Navigate:

**CloudWatch → Container Insights → ECS → Performance Monitoring**

Look for:

* CPU Reservation > **4x actual usage**
* Memory Reservation > **3x actual consumption**

### Example Observed:

| Resource | Requested | Actual Avg |
| -------- | --------- | ---------- |
| CPU      | 1024      | 80         |
| Memory   | 2048 MB   | 300 MB     |

This indicates **massive over-provisioning**.

---

## **Step 3 — Update & Register Optimized Task Definition**

Original:

```json
"cpu": 1024,
"memory": 2048
```

Optimized:

```json
"cpu": 256,
"memory": 512
```

Register new task definition:

```bash
aws ecs register-task-definition \
  --family user-service \
  --cli-input-json file://optimized-task.json
```

Update the service:

```bash
aws ecs update-service \
  --cluster acado-ecs-prod \
  --service user-service \
  --task-definition user-service:12
```

### **Expected Savings**

✔ **20–40% reduction** in CPU/memory expenses
✔ Better bin-packing and scaling behavior

---

# **EKS LAB — Cost Optimization for Kubernetes on AWS**

---

# **Lab 3 — Tune EKS Cluster Autoscaler for Cost Efficiency**

## **Objective**

Optimize scale-down behavior to remove idle nodes quicker and improve bin-packing efficiency.

---

## **Prerequisites**

* Functional EKS cluster
* Cluster Autoscaler installed or ready to install
* IAM role for Autoscaler

---

## **Step 1 — Install Cluster Autoscaler (If Not Installed)**

```bash
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

---

## **Step 2 — Tune Cost-Efficient Autoscaling Settings**

Edit deployment:

```bash
kubectl -n kube-system edit deployment cluster-autoscaler
```

Add/modify flags:

```yaml
- --scale-down-enabled=true
- --scale-down-delay-after-add=5m
- --scale-down-unneeded-time=5m
- --max-node-provision-time=10m
- --balance-similar-node-groups=true
```

### **Expected Improvements**

| Setting                       | Benefit                   |
| ----------------------------- | ------------------------- |
| `scale-down-unneeded-time=5m` | Removes idle nodes faster |
| `balance-similar-node-groups` | Improves node utilization |
| `max-node-provision-time=10m` | Allows Spot capacity      |

### **Expected Savings**

✔ **15–25% reduction** in EKS compute

---

# **Lab 4 — Implement Karpenter for Intelligent Node Provisioning**

## **Objective**

Replace CA with Karpenter for Spot-first, cost-aware provisioning.

---

## **Prerequisites**

* EKS cluster with IAM roles configured
* Helm installed
* Controller role created

---

## **Step 1 — Install Karpenter**

```bash
helm repo add karpenter https://charts.karpenter.sh
helm repo update

helm install karpenter karpenter/karpenter \
  --namespace karpenter \
  --create-namespace
```

---

## **Step 2 — Create Cost-Optimized NodePool**

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-optimized
spec:
  template:
    spec:
      instanceTypes: ["c5.large", "m5.large", "t3.large"]
      capacityType: "spot"
      subnetSelector:
        karpenter.sh/discovery: acado-eks
      securityGroupSelector:
        karpenter.sh/discovery: acado-eks
  ttlSecondsAfterEmpty: 120
```

Apply:

```bash
kubectl apply -f nodepool-spot.yaml
```

---

## **Step 3 — Deploy Load to Trigger Provisioning**

```bash
kubectl run load --image=nginx --replicas=50
```

### Karpenter Actions:

✔ Chooses **cheapest** available instance type
✔ Prefers **Spot**
✔ Consolidates workloads to minimize nodes
✔ Automatically removes idle nodes

### Expected Savings

✔ **30–50% compute reduction**

---

# **Lab Challenge — Reduce EKS Cluster Cost by 40%**

## **Goal**

Using all concepts learned, achieve **≥40% cost reduction** while keeping services healthy.

---

## **Allowed Techniques**

✔ Spot node groups
✔ Karpenter tuning
✔ Rightsizing containers
✔ Pod bin-packing optimization
✔ Consolidation of instance families
✔ Efficient requests/limits configurations

---

## **Success Criteria**

* Cluster remains **healthy**
* No increase in **latency**
* Compute cost reduced **40%+** (verified via CUR or CloudWatch)

---

# **Module 3 Deliverables**

Participants will deliver:

* Migration of ECS tasks from EC2 → Fargate Spot
* Rightsized ECS task definitions
* Tuned EKS Cluster Autoscaler
* Working Karpenter-based node provisioning
* Completion of 40% compute cost challenge
* End-to-end cost optimization strategy for container platforms
