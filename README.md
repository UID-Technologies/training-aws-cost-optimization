# Advanced AWS Cost Optimization – Hands-on TOC

**Duration:** 1 Full Day (8 Hours)  
**Audience:** Experienced AWS practitioners  
**Format:** 100% Hands-on Labs + Case Studies + Live Troubleshooting

---

## Module 1: Hands-on Deep Dive – Advanced AWS Cost Visibility

### Lab-Focused Activities:
- Creating custom AWS Cost & Usage Report (CUR) with Athena for deep analysis
- Building dashboards using QuickSight for granular cost mapping (per pod/container/function)
- Hands-on: Tagging strategy enforcement – Detect untagged resources using Config + Lambda
- Hands-on: Using AWS Cost Anomaly Detection with real anomaly simulation

---

## Module 2: RDS Cost Optimization – Real-World Lab Scenarios

### Hands-on Labs:
- Identifying expensive RDS workloads using Performance Insights & CloudWatch Metrics
- Lab: Storage auto-scaling & IOPS optimization challenge
- Lab: Multi-AZ vs Single-AZ reconfiguration cost simulation
- Snapshots & backup retention optimization – automate cleanup with Lambda

---

## Module 3: ECS / EKS Cost Optimization – Containers at Scale

### ECS Lab:
- Hands-on: Switching from EC2 launch type → Fargate Spot for cost reduction
- Rightsizing containers using CloudWatch Container Insights
- Lab: Identify over-provisioned CPU/memory tasks and optimize deployment

### EKS Lab:
- Hands-on: Cluster Autoscaler tuning for cost efficiency
- Lab: Using Karpenter for intelligent node provisioning (optimize instance families, Spot usage)
- Lab Challenge: Bring cluster cost down by 40% without service impact

---

## Module 4: Serverless Architecture Optimization

### Hands-on Labs:
- Lambda cost profiling using AWS Lambda Power Tuning
- Lab: Convert synchronous workloads → Event-driven model to reduce compute cost
- API Gateway cost optimization hands-on (REST → HTTP API migration)
- Lab Challenge: Reduce serverless pipeline cost by optimizing Lambda memory & execution time

---

## Module 5: Storage & Data Transfer Optimization Lab

### Hands-on Activities:
- S3 Intelligent-Tiering automation lab
- Lifecycle policies for archival (Glacier Deep Archive hands-on)
- Lab: Diagnose high data transfer bills using CloudWatch + VPC Flow Logs
- Hands-on creation of PrivateLink-based architecture to eliminate cross-AZ/Internet transfer costs

---

## Module 6: Compute Optimization – Advanced Real Scenarios

> **Note:** Participants already know EC2 basics → focus on real-world advanced labs

### Hands-on Labs:
- Auto Scaling deep-dive: Reduce cost by 30% with predictive scaling
- Lab: Rightsizing compute environments using Compute Optimizer + CloudWatch → automated remediation
- Lab: Migrating workloads to Graviton instances and measuring savings
- Hands-on: Identify Spot interruption patterns & build resilient workloads

---

## Module 7: Real Case Study – End-to-End Cost Optimization Challenge

A complex, multi-tier architecture combining RDS + ECS/EKS + Serverless + S3 + Networking.

### Participants will:
- Analyze an intentionally expensive AWS architecture
- Identify hidden cost drains
- Apply optimizations across multiple services
- Rebuild a cost-efficient version of the architecture
- Present results (improvement %, recommendations, cost breakdown)

---

## Module 8: Final Lab – Build an Automated Cost Governance Toolkit

Participants will create a small toolset for ongoing optimization:

### Hands-on:
- Lambda script to detect idle or underutilized resources
- Custom budgets & alerts
- Rightsizing & unused resource cleanup automation
- Deploy governance via IaC (Terraform/CloudFormation optional)

