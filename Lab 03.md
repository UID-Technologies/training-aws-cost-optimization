
# **MODULE 3 — ECS / EKS Cost Optimization: Containers at Scale**

## **1. Module Overview**

In this module, you will apply cost-optimization techniques to workloads running on Amazon ECS and Amazon EKS. You will learn to optimize compute provisioning, rightsize container resources, tune autoscaling, and shift workloads to Spot capacity to achieve up to **40–70% compute cost savings**.

ECS labs are **100% AWS Console Only**.
EKS labs use **AWS CLI, kubectl, and Helm on Windows Command Prompt**.

---

## **2. Learning Objectives**

### **ECS Cost Optimization**

* Migrate workloads from EC2 → Fargate → Fargate Spot
* Use ECS Capacity Providers
* Enable Container Insights for resource analysis
* Rightsize CPU/Memory allocations
* Implement ECS Service Auto Scaling for efficiency

### **EKS Cost Optimization**

* Connect to EKS using AWS CLI (Windows)
* Install and tune Cluster Autoscaler
* Install and configure Karpenter
* Use Spot nodes and instance flexibility
* Validate 40%+ cost reduction in compute spend

---

## **3. Prerequisites**

### **AWS Account Requirements**

* Region: **us-east-1**
* Permissions for ECS, EKS, EC2, CloudWatch, IAM, CloudFormation, VPC

### **Workstation Requirements (Windows)**

* AWS CLI installed
* kubectl installed
* Helm installed
* AWS credentials configured via:

  ```cmd
  aws configure
  ```

### **Sample Images**

* ECS sample:
  `public.ecr.aws/aws-containers/ecsdemo-nodejs:latest`
* EKS sample:
  NGINX or Node-based sample app
* Karpenter reference:
  [https://karpenter.sh/docs/](https://karpenter.sh/docs/)

---

## **4. Initial Setup**

### **Enable Cost Explorer**

AWS Console → Billing → Cost Explorer → Enable

### **Verify AWS CLI Installation**

```cmd
aws --version
aws sts get-caller-identity
```

---

# **ECS COST OPTIMIZATION LABS (Console Only)**

## **ECS LAB 1 — Migrate EC2 Launch Type to Fargate and Fargate Spot**

### **Objective**

Create a baseline EC2 workload and learn to migrate it to Fargate and Fargate Spot.

### **Step 1 — Create EC2 ECS Cluster**

1. ECS Console → Create Cluster
2. Choose **EC2 Linux + Networking**
3. Instance type: **t3.micro**
4. Number of instances: **1**
5. Create cluster

### **Step 2 — Create Over-Provisioned EC2 Task Definition**

* Name: `user-service-ec2-large`
* Launch Type: EC2
* CPU: **512** or **.5 vCPU**
* Memory: **1024**  or **1 GB**
* Container:

  * Image: `public.ecr.aws/aws-containers/ecsdemo-nodejs:latest`
  * Port: 8080

### **Step 3 — Deploy EC2 Service**

* Service name: `user-service-ec2-svc`
* Desired count: **2**
* Create ALB and test application

This forms your high-cost baseline.

---

## **ECS LAB 2 — Deploy and Optimize on Fargate + Fargate Spot**

### **Step 1 — Create Rightsized Fargate Task Definition**

* Name: `user-service-fargate`
* CPU: **256**
* Memory: **512**

### **Step 2 — Configure Capacity Providers**

Cluster → Capacity Providers → Add:

* FARGATE
* FARGATE_SPOT

Set weight distribution:

* FARGATE: 1
* FARGATE_SPOT: 3

### **Step 3 — Deploy Fargate Spot Service**

* Launch type: Capacity Provider Strategy
* Task definition: `user-service-fargate`
* Desired count: **4**

Validate tasks running on **FARGATE_SPOT**.

---

## **ECS LAB 3 — Rightsizing with Container Insights**

### **Step 1 — Enable Container Insights**

CloudWatch Console → Container Insights → Enable for ECS.

### **Step 2 — Generate Load**

Use browser auto-refresh or sample load generator from laptop.

### **Step 3 — Analyze Performance**

CloudWatch → ECS → Performance Monitoring
Review:

* CPUUtilized vs CPUReserved
* MemoryUtilized vs MemoryReserved

Expect significant over-provisioning.

### **Step 4 — Create Rightsized Revision**

* CPU: **128**  or **.25 vCPU**
* Memory: **256**  or  **0.5 GB**
  Update service to new revision.

---

## **ECS LAB 4 — Implement ECS Service Auto Scaling**

### **Step 1 — Enable Auto Scaling**

Within service → Auto Scaling:

* Policy: **Target Tracking**
* Metric: CPUUtilization
* Target: **50%**
* Min: 1
* Max: 10

Your ECS service now elastically scales without waste.

---

# **EKS COST OPTIMIZATION LABS (Windows Command Prompt)**

## **EKS LAB 1 — Setup Cluster + Install Cluster Autoscaler**

### **Objective**

Enable Kubernetes autoscaling to remove idle nodes and reduce compute waste.

---

### **Step 1 — Create EKS Cluster (Console)**

1. EKS Console → Create Cluster
2. Name: `eks-cost-lab`
3. Version: latest
4. Default VPC

Create Node Group:

* Instance type: **t3.small**
* Desired size: **2**

---

### **Step 2 — Connect kubectl to EKS (Windows CMD)**

```cmd
aws eks update-kubeconfig --region us-east-1 --name eks-cost-lab
kubectl get nodes
```

---

### **Step 3 — Deploy Sample App**

```cmd
kubectl create deployment web --image=public.ecr.aws/nginx/nginx:latest
kubectl expose deployment web --type=LoadBalancer --port=80
```

---

### **Step 4 — Install Cluster Autoscaler**

Download manifest:

```cmd
curl -o cluster-autoscaler.yaml https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

Apply:

```cmd
kubectl apply -f cluster-autoscaler.yaml
```

Patch recommended flags:
Open file:

```cmd
notepad cluster-autoscaler.yaml
```

Add under container args:

```
--balance-similar-node-groups=true
--skip-nodes-with-system-pods=false
--expander=least-waste
```

Reapply:

```cmd
kubectl apply -f cluster-autoscaler.yaml
```

Autoscaler now removes idle nodes faster.

---

## **EKS LAB 2 — Install & Configure Karpenter**

### **Objective**

Use Spot nodes and intelligent provisioning to cut costs by up to 70%.

---

### **Step 1 — Install Karpenter Infrastructure**

```cmd
curl -o karpenter.yaml https://karpenter.sh/getting-started/cloudformation.yaml
aws cloudformation deploy --stack-name Karpenter --template-file karpenter.yaml --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```

---

### **Step 2 — Install Karpenter Controller**

```cmd
helm repo add karpenter https://charts.karpenter.sh
helm repo update
helm install karpenter karpenter/karpenter ^
  --namespace karpenter --create-namespace ^
  --set controller.clusterName=eks-cost-lab ^
  --set controller.clusterEndpoint=$(aws eks describe-cluster --name eks-cost-lab --region us-east-1 --query "cluster.endpoint" --output text)
```

---

### **Step 3 — Create Spot Provisioner**

```cmd
notepad provisioner.yaml
```

Paste:

```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: spot-provisioner
spec:
  requirements:
  - key: "karpenter.sh/capacity-type"
    operator: In
    values: ["spot"]
  limits:
    resources:
      cpu: 20
  ttlSecondsAfterEmpty: 30
```

Apply:

```cmd
kubectl apply -f provisioner.yaml
```

---

### **Step 4 — Trigger Scaling**

```cmd
kubectl run load-generator --image=busybox -- sh -c "while true; do wget -q -O- http://web.default.svc.cluster.local; done"
kubectl get nodes -w
```

Watch Spot nodes provision and On-Demand nodes drain.

---

# **Final Lab Challenge — Achieve 40% Compute Cost Reduction**

Your task:

* Run workloads primarily on Spot
* Reduce idle node time to near-zero
* Maintain application availability
* Verify cost reduction in Cost Explorer


