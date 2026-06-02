# DAY 05

---

# PART 1 - HIGH AVAILABILITY: ELB & AUTO SCALING
## Scaling
### Vertical Scaling (Scale Up/Down)
- Add more CPU/RAM to an existing EC2 instance
- _Example: t3.micro → t3.xlarge_
- Simple, no code changes needed


**Problems**

Has a ceiling - you can't go beyond the biggest instance type.

Single point of failure - if that one instance dies, everything dies  

Requires downtime to resize

---

### **Horizontal Scaling (Scale Out/In) **
- Add more EC2 instances to handle load 
- Virtually unlimited scale 
- No single point of failure 
- Can scale in and out automatically (cost-efficient) 


**Problems**

Application needs to be designed to be stateless



---

## Load Balancing
| Type | Best For  | Layer |
| ----- | ----- | ----- |
| Application Load Balancer (ALB) | HTTP/HTTPS, web apps, microservices | Layer 7 |
| Network Load Balancer (NLB)  | Ultra-high performance, TCP/UDP  | Layer 4 |
| Gateway Load Balancer (GWLB) | Firewalls, deep packet inspection | Layer 3 |


We are using ALB today because: 

- It understands HTTP - can route based on URL paths  (/api/* → server A, /images/* → server B)
- Has built-in health checks - automatically stops sending traffic to unhealthy instances 
- Supports SSL termination 


Key ALB components:

- Listener - watches for incoming requests on a port (e.g. port 80) 
- Target Group - a group of EC2 instances to receive traffic 
- Health Check - ALB regularly pings each instance; if it fails, traffic stops going to it 
---

##  Auto Scaling Groups (ASG)
1. You define a Launch Template - the blueprint for new instances (AMI, instance type, user data) 

2. You set Min / Desired / Max capacity (e.g. min=2, desired=2, max=5) 

3. You attach scaling policies - rules that trigger scaling (e.g. "when CPU > 70% for 2 minutes, add 1 instance") 

4. ASG registers new instances with the ALB Target Group automatically



```
  ﻿Internet
     │
     ▼
[Application Load Balancer]   ← single entry point, distributes traffic
     │         │
     ▼         ▼
  [EC2-1]   [EC2-2]           ← target group (managed by ASG)
               ↑
         [EC2-3 added]        ← ASG adds this when CPU spikes 
```
# ALB + Auto Scaling Group [DEMO]
## Phase 1: Launch Two EC2 Instances
### Step 1: Open EC2 Console
1. Go to AWS Console
2. Search for EC2
3. Open the EC2 Dashboard
4. Click Launch Instances
---

## Step 2: Configure Instance 1
| Setting | Value |
| ----- | ----- |
| Name | `webserver-1`  |
| AMI | Amazon Linux 2023 AMI |
| Instance Type | `t2.micro`  |
| Key Pair | Create/select `demo-keypair`  |
| VPC | Default VPC |
| Subnet | `us-east-1a`  |
| Public IP | Enable |
| Security Group | `webserver-sg`  |
### Security Group Rules
| Type | Port | Source |
| ----- | ----- | ----- |
| HTTP | 80 | 0.0.0.0/0 |
| SSH | 22 | My IP |
### User Data Script
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<html><h1>Hello from WebServer 1 - $(hostname -f)</h1></html>" > /var/www/html/index.html
```
Click Launch Instance.

---

## Step 3: Configure Instance 2
Repeat the same process with these changes:

| Setting | Value |
| ----- | ----- |
| Name | `webserver-2`  |
| Subnet | `us-east-1b`  |
| Security Group | Existing `webserver-sg`  |
### User Data Script
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<html><h1>Hello from WebServer 2 - $(hostname -f)</h1></html>" > /var/www/html/index.html
```
Click Launch Instance.

---

## Step 4: Verify Instances
1. Navigate to EC2 → Instances
2. Wait until:
    - Status = Running
    - Health Checks = 2/2 Passed

3. Copy each Public IPv4 address
4. Open them in the browser and verify both web pages
---

# Phase 2: Create a Target Group
## Step 5: Create Target Group
### Navigation
```text
EC2 → Target Groups → Create Target Group
```
### Configuration
| Setting | Value |
| ----- | ----- |
| Target Type | Instances |
| Name | `demo-tg`  |
| Protocol | HTTP |
| Port | 80 |
| VPC | Default VPC |
| Health Check Path | `/`  |
### Register Targets
Select:

- `webserver-1` 
- `webserver-2` 
Then:

1. Click Include as pending below
2. Click Create target group


---

# Phase 3: Create Application Load Balancer
## Step 6: Create ALB
### Navigation
```text
EC2 → Load Balancers → Create Load Balancer
```
Select:

- Application Load Balancer
### Configuration
| Setting | Value |
| ----- | ----- |
| Name | `demo-alb`  |
| Scheme | Internet-facing |
| IP Type | IPv4 |
| VPC | Default VPC |
### Network Mapping
Enable both Availability Zones:

- `us-east-1a` 
- `us-east-1b` 
---

### Security Group
Create:

- `alb-sg` 
### Inbound Rule
| Type | Port | Source |
| ----- | ----- | ----- |
| HTTP | 80 | 0.0.0.0/0 |
---

### Listener Configuration
| Setting | Value |
| ----- | ----- |
| Protocol | HTTP |
| Port | 80 |
| Forward To | `demo-tg`  |
Click Create Load Balancer.



---

# Phase 4: Test the Load Balancer
## Step 7: Verify ALB
1. Open EC2 → Load Balancers
2. Select `demo-alb` 
3. Copy the DNS Name
Example:

```text
demo-alb-1234567890.us-east-1.elb.amazonaws.com
```
1. Open the DNS name in the browser
2. Refresh multiple times
Expected behavior:

- Requests alternate between WebServer 1 and WebServer 2


---

## Live Failure Demo
SSH into one instance:

```bash
sudo systemctl stop httpd
```
Observe:

- ALB health checks fail
- Traffic automatically routes only to the healthy instance
---

# Phase 5: Create Auto Scaling Group
## Step 8: Create Launch Template
### Navigation
```text
EC2 → Launch Templates → Create Launch Template
```
### Configuration
| Setting | Value |
| ----- | ----- |
| Name | `demo-launch-template`  |
| AMI | Amazon Linux 2023 |
| Instance Type | `t2.micro`  |
| Key Pair | `demo-keypair`  |
| Security Group | `webserver-sg`  |
### User Data Script
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<html><h1>Auto-Scaled Instance - $(hostname -f)</h1></html>" > /var/www/html/index.html
```
Click Create launch template.

---

## Step 9: Create Auto Scaling Group
### Navigation
```text
EC2 → Auto Scaling Groups → Create Auto Scaling Group
```
### Configuration
| Setting | Value |
| ----- | ----- |
| Name | `demo-asg`  |
| Launch Template | `demo-launch-template`  |
| VPC | Default VPC |
| Availability Zones | `us-east-1a`, `us-east-1b`  |
---

### Load Balancing
Select:

- Attach to an existing load balancer
- Choose `demo-tg` 
---

### Group Size
| Setting | Value |
| ----- | ----- |
| Desired Capacity | 2 |
| Minimum Capacity | 1 |
| Maximum Capacity | 4 |
---

### Scaling Policy
| Setting | Value |
| ----- | ----- |
| Policy Type | Target Tracking |
| Metric | Average CPU Utilization |
| Target Value | 50% |
Click through the remaining screens and create the Auto Scaling Group.

## Resources Cleanup
Do this after the demo to avoid AWS charges:

1. **ASG:** EC2 → Auto Scaling Groups → delete `demo-asg` 
2. **ALB:** EC2 → Load Balancers → delete `demo-alb` 
3. **Target Group:** EC2 → Target Groups → delete `demo-tg` 
4. **EC2 Instances:** EC2 → Instances → terminate `webserver-1`  and `webserver-2` 
5. **Launch Template:** EC2 → Launch Templates → delete `demo-launch-template` 


---

# PART 2 - AMAZON S3 & STORAGE
## The Three Types of Storage
### Storage Comparison
| Feature | Block Storage (EBS) | File Storage (EFS) | Object Storage (S3) |
| ----- | ----- | ----- | ----- |
| Analogy | Hard drive attached to 1 PC | Shared network drive | Infinite filing cabinet |
| Access Type | OS-level block access | Shared file access | Object/key-based access |
| Scalability | Limited | Shared scaling | Virtually unlimited |
| Multi-instance Support | No* | Yes | Yes |
| Latency | Very low | Moderate | Higher |
| Cost | Medium | Higher | Cheapest |
| File System | Yes | Yes | No (flat object structure) |
| Best Use Cases | OS disks, databases | Shared CMS/code | Images, backups, static sites |


---

# Amazon S3 Deep Dive
## Important S3 Facts
- Bucket names must be globally unique
- Single object size limit = 5 TB
- Buckets can store unlimited objects
- S3 is not a traditional file system
- "Folders" are simulated using `/`  in object keys
- S3 provides 99.999999999% durability (11 nines)
- Data is automatically replicated across multiple Availability Zones
---

# S3 Storage Classes
## Storage Classes Comparison
| Storage Class | Use Case | Retrieval Speed | Relative Cost |
| ----- | ----- | ----- | ----- |
| S3 Standard | Frequently accessed data | Instant | $$$ |
| S3 Standard-IA | Infrequently accessed data | Instant | $$ |
| S3 One Zone-IA | Infrequent data tolerant to AZ loss | Instant | $ |
| S3 Glacier Instant | Quarterly archive access | Milliseconds | $ |
| S3 Glacier Flexible | Long-term archives | Minutes to hours | ¢ |
| S3 Glacier Deep Archive | Compliance retention | Up to 12 hours | ¢¢ |
| S3 Intelligent-Tiering | Unknown/changing access | Instant | Automatic |
---

## Real Cost Example
> S3 Standard costs approximately $0.023 per GB/month while Glacier Deep Archive costs approximately $0.00099 per GB/month.

### Example Scenario
- 10 TB old compliance logs
- Standard Storage = ~$235/month
- Glacier Deep Archive = ~$10/month
- 7-year savings ≈ $16,800
---

## Lifecycle Policies
### Example Lifecycle Strategy
```text
After 30 days → Move to Standard-IA
After 90 days → Move to Glacier
After 7 years → Delete automatically
```
---

# S3 Versioning & Security
## S3 Versioning
### Features
- Stores every version of an object
- Protects against accidental deletion
- Protects against overwrites
- Delete operations create delete markers
- Older versions can be restored anytime
---

## S3 Security Layers
| Security Layer | Purpose |
| ----- | ----- |
| Bucket Policies | Define bucket-level permissions |
| ACLs | Legacy permission model |
| Block Public Access | Prevent accidental public exposure |
| IAM Policies | Control user/role access |
| Encryption | Encrypt data at rest and in transit |
---

# S3 Static Website Hosting [DEMO]
## Goal
Create:

- An S3 bucket
- Static website hosting
- Public access policy
- Versioned object storage
---

# Phase 1 - Create the S3 Bucket
## Step 1: Navigate to S3
### Navigation
```text
AWS Console → Search S3 → Open S3 Console
```
Click:

- Create Bucket
---

## Step 2: Configure Bucket
| Setting | Value |
| ----- | ----- |
| Bucket Name | `demo-static-website-[yourname]-2024`  |
| Region | us-east-1 or closest region |
| Object Ownership | ACLs Disabled |
| Block Public Access | Disable "Block all public access" |
| Versioning | Enable |
### Important Notes
- Bucket names must be globally unique
- Use lowercase letters, numbers, and hyphens only
- Confirm acknowledgement warning for public access
Click Create Bucket.



---

# Phase 2 - Create and Upload Website Files
## Step 3: Create index.html
### index.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My AWS Static Website</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            background: linear-gradient(135deg, #232f3e, #ff9900);
            color: white;
            text-align: center;
        }
        .container {
            background: rgba(0,0,0,0.4);
            padding: 40px;
            border-radius: 12px;
        }
        h1 { font-size: 2.5rem; margin-bottom: 10px; }
        p  { font-size: 1.2rem; opacity: 0.9; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚀 Hello from Amazon S3!</h1>
        <p>This static website is hosted entirely on S3.</p>
        <p>No servers. No EC2. No maintenance.</p>
        <p><strong>AWS Training Demo — Day 2</strong></p>
    </div>
</body>
</html>
```
---

## Create error.html
```html
<!DOCTYPE html>
<html>
<head><title>Page Not Found</title></head>
<body>
    <h1>404 - Page Not Found</h1>
    <p><a href="index.html">Go Home</a></p>
</body>
</html>
```
---

## Step 4: Upload Files
1. Open the bucket
2. Click Upload
3. Add both files:
    - index.html
    - error.html

4. Click Upload
5. Wait for upload success confirmation


---

# Phase 3 - Enable Static Website Hosting
## Step 5: Enable Hosting
### Navigation
```text
S3 Bucket → Properties → Static Website Hosting → Edit
```
### Configuration
| Setting | Value |
| ----- | ----- |
| Hosting Type | Host a static website |
| Index Document | `index.html`  |
| Error Document | `error.html`  |
Click Save Changes.

---

## Website Endpoint Example
```text
http://demo-static-website-yourname-2024.s3-website-us-east-1.amazonaws.com
```
---

## Expected Result
Opening the endpoint initially shows:

```text
403 Forbidden
```
because the bucket policy has not yet granted public access.



---

# Phase 4 - Add Bucket Policy
## Step 6: Configure Bucket Policy
### Navigation
```text
S3 Bucket → Permissions → Bucket Policy → Edit
```
### Bucket Policy
Replace `YOUR-BUCKET-NAME` with your actual bucket name.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
        }
    ]
}
```
Click Save Changes.

---

## Bucket Policy Breakdown
| Policy Element | Meaning |
| ----- | ----- |
| Version | Policy language version |
| Effect | Allow access |
| Principal | Everyone (`*`) |
| Action | Read/download objects |
| Resource | All objects inside bucket |


---

## Step 7: Test Website
1. Open Properties tab
2. Scroll to Static Website Hosting
3. Open Bucket Website Endpoint
4. Website should now load successfully
---

# Phase 5 - Demonstrate Versioning
## Step 8: Upload Updated Version
1. Edit `index.html` 
2. Change heading to:
```html
<h1>Version 2 - Updated!</h1>
```
1. Upload the same file again
2. Refresh website
3. Enable Show Versions in S3 object list
4. Observe multiple versions of the file


---

# Wrap-up Talking Points
- S3 static hosting costs fractions of a cent per month
- Ideal for landing pages, documentation, and frontend applications
- S3 only serves static content
- Dynamic applications still require EC2, ECS, or Lambda
- CloudFront is commonly placed in front of S3 for HTTPS and caching
- Major companies use S3 at massive scale
---

# Architecture Summary
```text
      INTERNET
         │
  ┌──────▼──────┐
  │     ALB      │
  └──┬───────┬───┘
     │       │
┌────▼───┐ ┌─▼────┐
│ EC2 AZ1│ │EC2AZ2│
└────────┘ └──────┘

      +

  ┌──────────────────┐
  │   Amazon S3      │
  │   (Static Site)  │
  └──────────────────┘
```
---

# Key AWS Services Covered
| Service | Purpose | Common Use Cases |
| ----- | ----- | ----- |
| ALB | Load balancing HTTP traffic | Multi-server web apps |
| ASG | Automatic scaling and healing | Variable traffic workloads |
| S3 | Durable object storage | Images, backups, static sites |
| EBS | Block storage for EC2 | OS disks, databases |
| EFS | Shared network file storage | Shared application data |


