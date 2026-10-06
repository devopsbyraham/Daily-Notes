# AWS AMI — Amazon Machine Image

## 1. What is an AMI?

**AMI (Amazon Machine Image)** is a reusable template used to launch EC2 instances with a predefined operating system, software, configuration, and storage mapping.

> **AMI = Reusable Server Template**

```text
AMI
 ↓
Launch EC2
 ↓
Same OS + Software + Configuration
```

---

## 2. Purpose of AMI

AMI is mainly used to:

- Create identical EC2 servers
- Create reusable server templates
- Migrate servers between AWS Regions
- Create standardized production servers
- Build Golden AMIs
- Recover/recreate EC2 environments
- Support Auto Scaling deployments

---

# 3. AMI Sources / Categories

| Category | Meaning |
|---|---|
| Quick Start | AWS-provided commonly used AMIs |
| My AMIs | AMIs created by you or shared with you |
| AWS Marketplace | AMIs published by software vendors |
| Community AMIs | Publicly shared AMIs from other users/organizations |

### Quick Start

Examples:

```text
Amazon Linux
Ubuntu
Windows Server
RHEL
```

### My AMIs

```text
EC2
 ↓
Customize Server
 ↓
Create AMI
 ↓
My AMIs
```

### AWS Marketplace

Used for pre-configured commercial or specialized software.

Examples:

```text
Security Appliances
Monitoring Software
Databases
Commercial Applications
```

### Community AMIs

Publicly shared AMIs.

> ⚠️ **Production caution:** Do not blindly use an unknown Community AMI. Verify the publisher, OS version, security posture, patches, and software source.

---

# 4. Create EC2 Server

Create an EC2 instance and configure the application.

Example Ubuntu User Data:

```bash
#!/bin/bash

apt update -y
apt install nginx git -y

cd /tmp
git clone https://github.com/Ironhack-Archive/online-clone-amazon.git

cp -r online-clone-amazon/* /var/www/html/
```

### Important

EC2 User Data generally runs with **root privileges**, so `sudo -i` is usually unnecessary.

Prefer:

```bash
#!/bin/bash
```

over:

```bash
#! /bin/bash
```

---

# 5. Create an AMI

After configuring the EC2 instance:

```text
EC2
 ↓
Select Server
 ↓
Actions
 ↓
Image and templates
 ↓
Create image
```

Example:

```text
Image Name  : SWIGGY-IMAGE
Description : Optional
```

Then:

```text
Create Image
```

Wait until the AMI becomes available before using it.

---

# 6. What Does an AMI Contain?

For EBS-backed EC2 instances, an AMI contains launch information and references to the EBS snapshots required to recreate the instance's storage.

Conceptually:

```text
AMI
├── Operating System
├── Installed Software
├── Configuration
├── Application
├── Security Configuration
└── EBS Snapshot References
```

> **AMI is not simply a single full-disk copy.**

---

# 7. Launch a Server from AMI

```text
EC2
 ↓
AMIs
 ↓
Select AMI
 ↓
Launch instance from AMI
 ↓
Configure Instance
 ↓
Create Instance
```

The new EC2 instance starts from the AMI's predefined configuration.

---

# 8. Server Migration Between AWS Regions

### Example

```text
Source Region
      ↓
Create AMI
      ↓
Copy AMI
      ↓
Destination Region
      ↓
Launch EC2
```

### Console Flow

```text
EC2
 ↓
AMIs
 ↓
Select AMI
 ↓
Actions
 ↓
Copy AMI
 ↓
Select Destination Region
 ↓
Copy
```

Then switch to the destination Region:

```text
Destination Region
 ↓
AMIs
 ↓
Select Copied AMI
 ↓
Launch Instance
```

### 🎯 Interview Answer

> **Create an AMI from the source EC2 instance, copy the AMI to the destination Region, and launch a new EC2 instance from the copied AMI.**

---

# 9. Important Migration Considerations

AMI migration does **not automatically migrate every AWS dependency**.

Review:

```text
Security Groups
IAM Roles
Key Pairs
Elastic IP
DNS
Load Balancer
Databases
S3 Data
Secrets
KMS Keys
VPC/Subnets
Route Tables
Application Dependencies
```

### Production Thinking

```text
AMI Migration
      +
Network Migration
      +
Data Migration
      +
DNS Migration
      +
Security Configuration
      =
Complete Application Migration
```

---

# 10. AMI as Server Recovery

An AMI can be used to recreate an EC2 instance with a known configuration.

```text
Production EC2
      ↓
Create AMI
      ↓
AMI
      ↓
Launch New EC2
      ↓
Recovered Server
```

### AMI vs Backup

An AMI can help with EC2 recovery, but it should not automatically be considered a complete enterprise backup strategy.

For production backup requirements, consider:

- AWS Backup
- EBS Snapshots
- Snapshot Lifecycle Policies
- Cross-Region Backup/Copy
- Cross-Account Backup
- Recovery Testing

---

# 11. AMI vs EBS Snapshot

| AMI | EBS Snapshot |
|---|---|
| Used to launch EC2 instances | Backup of an EBS volume |
| Contains launch metadata | Contains volume data |
| Can contain multiple volume mappings | Represents a volume backup |
| Useful as a server template | Useful for volume backup/recovery |
| References EBS snapshots | Stores point-in-time volume data |

### 🧠 Memory Trick

> **AMI = Server Template**

> **Snapshot = Disk Backup**

---

# 12. Recover a Deleted AMI Using Recycle Bin

AWS Recycle Bin can retain supported resources after deletion/deregistration when an applicable retention rule exists.

## Step 1 — Create Retention Rule

```text
EC2
 ↓
Recycle Bin
 ↓
Retention Rules
 ↓
Create Retention Rule
```

Example:

```text
Rule Name      : rule01
Resource Type  : AMI
Retention      : 7 days
```

## Step 2 — Deregister AMI

```text
AMI
 ↓
Select AMI
 ↓
Actions
 ↓
Deregister AMI
```

## Step 3 — Recover

If the AMI is covered by the retention rule:

```text
Recycle Bin
 ↓
Resources
 ↓
Select AMI
 ↓
Recover
```

### 🧠 Memory Trick

> **Deregister + Retention Rule → Recycle Bin → Recover**

---

# 13. Golden AMI ⭐

## Definition

> **Golden AMI = A standardized, security-hardened, tested AMI containing the organization's approved OS, patches, agents, configurations, and software baseline.**

Typical Golden AMI:

```text
Base OS
   ↓
Security Patches
   ↓
Monitoring Agent
   ↓
Logging Agent
   ↓
Security Agent
   ↓
Approved Packages
   ↓
Application Dependencies
   ↓
Security Hardening
   ↓
Testing
   ↓
Golden AMI
```

Then:

```text
Golden AMI
     ↓
Launch Template
     ↓
Auto Scaling Group
     ↓
EC2 Instances
```

### Production Principle

> **Build Once → Test Once → Deploy Consistently**

---

# 14. AMI + Launch Template + ASG

A common production architecture:

```text
              Golden AMI
                   │
                   ▼
            Launch Template
                   │
                   ▼
                  ASG
             ┌─────┼─────┐
             ▼     ▼     ▼
            EC2   EC2   EC2
```

### Remember

> **AMI = What is installed**

> **Launch Template = How to launch**

> **ASG = How many instances should run**

---

# 15. AMI Versioning

Avoid continuously modifying a single production image.

Prefer immutable, versioned AMIs:

```text
web-ami-v1
web-ami-v2
web-ami-v3
```

Typical process:

```text
Build
 ↓
Test
 ↓
Create AMI
 ↓
Version
 ↓
Deploy
 ↓
Monitor
 ↓
Rollback if required
```

This makes deployments and rollbacks more predictable.

---

# 16. Common Production Considerations

Before using an AMI in production, verify:

- OS security patches
- Application version
- Monitoring agents
- Logging agents
- Security agents
- IAM permissions
- Network configuration
- Secrets handling
- Disk configuration
- Startup scripts
- Cloud-init/User Data
- Vulnerability scanning
- AMI versioning
- Rollback strategy

### ⚠️ Security

Do not bake sensitive credentials, passwords, API keys, or long-lived secrets into an AMI.

Use appropriate secret-management solutions instead.

---

# 17. Common Interview Questions

## Q1. What is an AMI?

> AMI is a reusable template used to launch EC2 instances with a predefined operating system, software, configuration, and storage mapping.

## Q2. How do you migrate an EC2 server between Regions?

> Create an AMI → Copy the AMI to the destination Region → Launch EC2 from the copied AMI.

## Q3. What is a Golden AMI?

> A standardized, security-hardened, tested AMI containing an organization's approved OS, patches, agents, configurations, and software baseline.

## Q4. What is the difference between AMI and Snapshot?

> AMI is used to launch EC2 instances, while an EBS snapshot is a point-in-time backup of an EBS volume.

## Q5. Can an AMI be copied to another Region?

> Yes. An AMI can be copied to another AWS Region and used to launch EC2 instances there.

## Q6. What happens when an AMI is deregistered?

> It can no longer be used to launch new EC2 instances. If covered by an applicable Recycle Bin retention rule, it may be recoverable during the retention period.

## Q7. Why use a Golden AMI?

> To provide a consistent, secure, tested baseline for production EC2 instances.

## Q8. Why use AMIs with Auto Scaling?

> AMIs provide the standardized server image from which ASG instances can be launched.

---

# 18. One-Minute Revision

```text
                         AMI
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
   Quick Start          My AMIs        Marketplace
       │                  │                  │
      AWS                You              Vendors
                          │
                    Community AMIs
                          │
                     Public Users
```

### Server Creation

```text
EC2
 ↓
Configure Server
 ↓
Create AMI
 ↓
Launch New EC2
```

### Migration

```text
Source Region
 ↓
AMI
 ↓
Copy AMI
 ↓
Destination Region
 ↓
Launch EC2
```

### Production

```text
Golden AMI
 ↓
Launch Template
 ↓
ASG
 ↓
EC2 × N
```

---

# ⭐ Final Memory Line

> **AMI = A reusable, versioned server template used to consistently launch EC2 instances.**

## 🔥 Three Things to Never Forget

```text
AMI       → Server Template
Snapshot  → Disk Backup
Golden AMI → Standardized Production Template
```
