
# **Module 5 — Storage & Data Transfer Optimization**

This module teaches participants how to reduce AWS storage and data transfer costs, using real best practices applied in large-scale cloud environments.

You will work with:

* Amazon S3
* Glacier & Deep Archive
* CloudWatch Logs
* VPC Flow Logs
* PrivateLink (Interface Endpoints)
* Load Balancers
* EC2 test resources

No CLI is used — every step is performed through the **AWS Console**.

---

# **1. Module Overview**

Participants will learn how to:

* Identify and optimize unnecessary S3 storage cost
* Configure Intelligent-Tiering
* Configure Lifecycle rules (transition → Glacier → Deep Archive → expiration)
* Diagnose high NAT Gateway, public Internet, and cross-AZ network transfer charges
* Implement PrivateLink (Interface Endpoints) to eliminate egress and NAT charges
* Design an optimized storage + network architecture achieving 40%+ savings

---

# **2. Learning Objectives**

By the end of this module, you will be able to:

### Optimize S3 storage

* Enable Intelligent-Tiering at bucket level
* Apply Lifecycle rules for archival and expiration
* Understand cost trade-offs between S3 Standard, IA, One Zone, Glacier, Glacier Deep Archive

### Diagnose high data transfer spend

* Use VPC Flow Logs to see internal/external traffic
* Use CloudWatch Logs Insights to identify traffic from NAT Gateways, Internet egress, or cross-AZ flows

### Build PrivateLink architectures

* Create Network Load Balancer for private service exposure
* Create Endpoint Service (provider)
* Create Interface Endpoint (consumer)
* Understand cost reduction impact

### Perform full architecture optimization

* Redesign workloads to minimize storage + data transfer cost

---

# **3. Initial Setup (Console-Only)**

Before starting the labs, perform the following:

## Step 1 — Create an S3 Bucket for Testing

1. Open **S3 Console → Create Bucket**
2. Bucket name:

   ```
   storage-optimization-lab-<yourname>
   ```
3. Leave public access blocked
4. Click **Create bucket**

## Step 2 — Upload Sample Files

1. Open your S3 bucket
2. Click **Upload**
3. Add any local files (for lab purposes):

   * 50–100 MB logs
   * Several images
   * Application artifacts
4. Click **Upload**

## Step 3 — Confirm You Have a Working VPC

Use **default VPC** (recommended for labs).
No action needed unless your account has VPC disabled.

---

# **S3 LAB 1 — Intelligent-Tiering Optimization**

## Goal

Enable S3 Intelligent-Tiering for automatic cost reduction on objects with unpredictable access patterns.

### Why this matters:

Intelligent-Tiering automatically moves objects between **Frequent**, **Infrequent**, **Archive Access**, and **Deep Archive Access** tiers based on real usage.

---

## Step 1 — Convert Existing Objects to Intelligent-Tiering

1. Open **S3 Console**
2. Open bucket: `storage-optimization-lab-<yourname>`
3. Select an object (e.g., log file or image)
4. Click **Actions → Change storage class**
5. Select **Intelligent-Tiering**
6. Save changes

This immediately moves the object into the Intelligent-Tiering monitoring state.

---

## Step 2 — Configure Automatic Intelligent-Tiering for the Entire Bucket

1. Open the bucket
2. Go to **Management → Lifecycle rules → Create lifecycle rule**
3. Rule name:

   ```
   intelligent-tiering-auto
   ```
4. Rule scope: **This bucket**
5. Choose **Transition to Intelligent-Tiering**
6. Configure:

   * Move to **Intelligent-Tiering** after **1 day**
7. Leave expiration disabled
8. Click **Create rule**

All future uploads will automatically enter Intelligent-Tiering.

---

## Step 3 — Verify Tier Behavior (24–48 hours)

After one day:

1. Open object → **Properties**
2. Check **Storage Class**

   * Should show:
     `INTELLIGENT_TIERING` → Frequent Access
3. After inactivity:

   * Automatically transitions to **Infrequent** or **Archive Access** tiers

---

## S3 LAB 1 Checklist

| Task                                       | Done | Verified | Screenshot |
| ------------------------------------------ | ---- | -------- | ---------- |
| Converted objects to Intelligent-Tiering   |      |          |            |
| Created Intelligent-Tiering lifecycle rule |      |          |            |
| Verified transition in object metadata     |      |          |            |

---

# **S3 LAB 2 — Lifecycle Policy for Glacier + Deep Archive**

## Goal

Create lifecycle rules to automatically archive old data into S3 Glacier and Glacier Deep Archive.

### Why this matters:

Deep Archive costs **up to 90% less** than S3 Standard and is ideal for logs, backups, and compliance storage.

---

## Step 1 — Create Lifecycle Archival Rule

1. Go to **S3 → Bucket → Management → Lifecycle rules → Create rule**
2. Rule name:

   ```
   archive-deepglacier
   ```

### Rule Settings:

* Filter: Prefix =

  ```
  oldfiles/
  ```
* Transition actions:

  * Move to **Glacier Flexible Retrieval** after **30 days**
  * Move to **Glacier Deep Archive** after **90 days**
* Expiration (optional):

  * Delete objects after **1 year**

3. Click **Create rule**

---

## Step 2 — Prepare Test Data

1. Create folder in the bucket: `oldfiles/`
2. Upload sample files into it

Lifecycle rules will apply automatically.

---

## Step 3 — Validate Transition (Post-Delay)

After 30+ days (or fast-forward in production):

* Objects' **Storage Class** changes accordingly
* You can simulate viewing transition expectation via UI

---

## S3 LAB 2 Checklist

| Task                                            | Done | Verified | Screenshot |
| ----------------------------------------------- | ---- | -------- | ---------- |
| Created lifecycle rule for Glacier/Deep Archive |      |          |            |
| Uploaded test files to oldfiles/                |      |          |            |
| Verified lifecycle policy future transitions    |      |          |            |

---

# **LAB 3 — Diagnose High Data Transfer Costs Using VPC Flow Logs & CloudWatch**

## Goal

Learn how to find the root cause behind NAT Gateway charges, Internet data transfer, and cross-AZ traffic.

### Why this matters:

Many AWS bills explode because of:

* NAT Gateway processing fees
* Egress to the Internet
* Cross-AZ inter-service communication
* Accessing S3 via public IP instead of a VPC endpoint

---

## Step 1 — Enable VPC Flow Logs

1. Go to **VPC Console → Your VPCs**
2. Select the **default VPC**
3. Click **Create flow log**
4. Settings:

   * Filter: **ALL**
   * Destination: **CloudWatch Logs**
   * Log Group Name: `/vpc/flowlogs`
5. Click **Create flow log**

Flow Logs now capture all network traffic metadata.

---

## Step 2 — Generate Some Traffic (Optional)

1. Launch any EC2 instance (t2.micro) in a public subnet
2. SSH into EC2 (or Access using SSM Session Manager)
3. Run commands such as:

   * Browsing external APIs
   * Downloading files
   * Calling S3

This creates traffic entries that appear in VPC Flow Logs.

---

## Step 3 — Analyze Flow Logs Using CloudWatch Logs Insights

1. Open **CloudWatch → Logs Insights**
2. Select Log Group: `/vpc/flowlogs`
3. Run query to identify highest traffic sources:

```sql
fields srcAddr, dstAddr, bytes
| sort bytes desc
| limit 20
```

To diagnose NAT Gateway traffic:

```sql
fields srcAddr, dstAddr, bytes
| filter dstAddr like /^52\./
| sort bytes desc
```

### Interpretation:

* `dstAddr` beginning with **52.x.x.x** → AWS public endpoints
* Very high bytes → expensive NAT Gateway data processing
* Non-RFC1918 IPs → Internet egress

---

## Step 4 — Identify Cost Issues

Possible findings:

### NAT Gateway overuse

EC2 instances pulling data from Internet or AWS services without VPC endpoints.

### Cross-AZ communication

If two services in different AZs talk to each other → cross-AZ cost accumulates.

### Public Egress to S3

Happens when:

* No S3 VPC endpoint exists
* Or application uses S3 public URLs

These must be corrected in architecture.

---

## LAB 3 Checklist

| Task                                             | Done | Verified | Screenshot |
| ------------------------------------------------ | ---- | -------- | ---------- |
| Enabled VPC Flow Logs                            |      |          |            |
| Ran CloudWatch Logs Insights queries             |      |          |            |
| Identified high-volume traffic sources           |      |          |            |
| Identified NAT / Internet / cross-AZ root causes |      |          |            |

---

# **LAB 4 — Build PrivateLink Architecture (Console-Only)**

### *(No CLI. 100% Console Steps)*

## Goal

Create PrivateLink connection between two VPCs (provider → consumer) using only the AWS Console.

### Why PrivateLink?

* Eliminates **Internet egress cost**
* Eliminates **NAT Gateway charges**
* Eliminates **cross-AZ transfer** when properly deployed
* Provides **secure, private connectivity**

---

## Step 1 — Launch Provider EC2 Instance

1. Open **EC2 Console → Launch instance**
2. Name: `PrivateLinkProvider`
3. AMI: Amazon Linux 2
4. Instance type: t3.micro
5. Network: default VPC
6. Security group:

   * Allow inbound **TCP 80** from VPC CIDR
7. Launch instance

## Step 2 — Install Simple Web Server on Provider EC2

1. EC2 → Connect → **Session Manager**
2. Run:

   ```
   sudo yum install -y httpd
   echo "PrivateLink Test" | sudo tee /var/www/html/index.html
   sudo systemctl enable httpd
   sudo systemctl start httpd
   ```

---

## Step 3 — Create a Network Load Balancer (NLB)

1. EC2 Console → **Load Balancers → Create load balancer**
2. Choose: **Network Load Balancer**
3. Scheme: **Internal**
4. Listeners:

   * TCP 80
5. Create Target Group:

   * Type: Instance
   * Protocol: TCP 80
   * Register your EC2 instance
6. Create Load Balancer

---

## Step 4 — Create a VPC Endpoint Service (Provider)

1. Open **VPC Console → Endpoint Services**
2. Click **Create endpoint service**
3. Select **Your NLB**
4. Check: “Require acceptance for endpoint requests”
5. Create service

Copy the **Service Name** — needed for consumer.

---

## Step 5 — Create a VPC Interface Endpoint (Consumer)

1. Open **VPC Console → Endpoints → Create Endpoint**
2. Select **Other endpoint service**
3. Paste the Service Name
4. Select a VPC and subnet
5. Enable:

   * **Private DNS**
   * **Security groups** allowing port 80
6. Create endpoint

Accept the endpoint connection in Provider account (if asked).

---

## Step 6 — Validate PrivateLink Connectivity

In consumer VPC:

1. Launch EC2 instance
2. Connect via Session Manager
3. Run:

   ```
   curl http://<privatelink-dns>
   ```

Expected output:

```
PrivateLink Test
```

This traffic:

* Does NOT go to the Internet
* Does NOT use NAT Gateway
* Does NOT generate cross-AZ cost (if endpoints placed in same AZ)

---

## LAB 4 Checklist

| Task                           | Done | Verified | Screenshot |
| ------------------------------ | ---- | -------- | ---------- |
| Launched provider EC2 instance |      |          |            |
| Created NLB                    |      |          |            |
| Created Endpoint Service       |      |          |            |
| Created Interface Endpoint     |      |          |            |
| Validated PrivateLink access   |      |          |            |

---

# **Final Module 5 Challenge — Achieve 40% Storage + Data Transfer Cost Reduction**

Participants must redesign an existing architecture and demonstrate measurable cost improvements.

---

## Your Objectives

### S3 Savings

* Move active data to Intelligent-Tiering
* Archive cold data to Glacier / Deep Archive
* Expire old logs

### Network Savings

* Replace NAT Gateway traffic with VPC Endpoints or PrivateLink
* Reduce cross-AZ chatter
* Remove Internet egress patterns

---

## Deliverables

### S3 Optimization Deliverables:

* Before vs after storage class distribution
* Lifecycle rules screenshots
* Intelligent-Tiering evidence

### Data Transfer Diagnostics:

* CloudWatch Insights screenshots
* Identified root causes
* Proposed architectural fix

### PrivateLink Redesign:

* Architecture diagram
* Endpoint Service screenshot
* Interface Endpoint screenshot
* Validation test output

### Final Savings Report:

* Storage savings %
* NAT/Data transfer savings %
* Total savings >= 40%

---

