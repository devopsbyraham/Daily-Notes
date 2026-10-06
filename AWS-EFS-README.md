# Amazon EFS — Elastic File System

## 1. What is EFS?

**Amazon EFS (Elastic File System)** is a fully managed, scalable file system that allows multiple servers and AWS services to access the **same shared files**.

> **EFS = Shared, scalable Linux file storage**

---

## 2. Key Features

| Feature | Details |
|---|---|
| Purpose | Share files/data between multiple servers |
| Storage | Automatically scales as data grows |
| Management | Fully managed / serverless |
| Protocol | NFSv4.0 and NFSv4.1 |
| Performance Modes | General Purpose, Elastic |
| Integrations | EC2, ECS, EKS, Lambda, Fargate |
| Backup | Can be protected using AWS Backup |
| Access | Controlled using network connectivity and Security Groups |

---

## 3. EFS Architecture

```text
                 ┌──────────────────┐
                 │       EFS        │
                 │  Shared Storage  │
                 └────────┬─────────┘
                          │
                    NFS 4.1 / 2049
                          │
             ┌────────────┼────────────┐
             │            │            │
          EC2-1         EC2-2        EKS/ECS
             │            │            │
             └────────────┴────────────┘
                    Same Files
```

### Simple Understanding

If multiple servers mount the same EFS filesystem:

```text
EC2-1 ──┐
        │
EC2-2 ──┼──> EFS
        │
EC2-3 ──┘
```

All servers can access the same shared data.

---

# 4. Create EFS

### AWS Console

```text
AWS Console
   ↓
EFS
   ↓
Create File System
   ↓
Customize
```

Recommended session configuration:

```text
Name        : ONE
File System : Regional
Throughput  : Elastic
Security    : Security Group
```

Then:

```text
Next
 ↓
Create
```

---

# 5. Security Group Configuration

EFS uses **NFS** for communication.

### Required Rule

```text
Protocol : TCP
Port     : 2049
Source   : Application/EC2 Security Group
```

### Important

> **If NFS (TCP 2049) is not allowed between the client and EFS, the EC2 server cannot properly mount/access the EFS filesystem.**

### Memory Trick

> **EFS → NFS → TCP 2049**

---

# 6. Prepare EC2 Server

For Ubuntu:

```bash
sudo apt update -y
sudo apt install nfs-common -y
```

Check the existing web directory:

```bash
ls /var/www/html/
```

---

# 7. Mount EFS on EC2

Go to:

```text
EFS
 ↓
Select File System
 ↓
Attach
```

AWS provides the mount commands.

Example:

```bash
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-02ea223ceb1c70423.efs.us-east-1.amazonaws.com:/ /var/www/html
```

After mounting:

```bash
cd /var/www/html
```

---

# 8. Verify the Mount

Check the mounted filesystem:

```bash
df -h
```

You can also verify the mount:

```bash
mount | grep efs
```

Check the directory:

```bash
ls -la /var/www/html/
```

---

# 9. Practical Test

Clone an application into the EFS-mounted directory:

```bash
cd /var/www/html

git clone https://github.com/Ironhack-Archive/online-clone-amazon.git

mv online-clone-amazon/* .
```

Now the application files are stored on EFS.

If another EC2 server mounts the **same EFS filesystem**, it can access the same files.

```text
EC2-1
  │
  ├── /var/www/html
  │
  ▼
 EFS
  ▲
  │
  ├── /var/www/html
  │
EC2-2
```

---

# 10. EFS with Multiple Servers

### Without EFS

Each server has its own local files:

```text
EC2-1 → Local Storage
EC2-2 → Local Storage
EC2-3 → Local Storage
```

Files are different.

### With EFS

```text
EC2-1 ──┐
EC2-2 ──┼──→ EFS
EC2-3 ──┘
```

All servers can access the same shared files.

---

# 11. EFS and Auto Scaling

An Auto Scaling Group does not automatically mount EFS just because EFS exists.

The EC2 instances launched by the ASG need to be configured to mount EFS.

Typical approach:

```text
Create EC2
    ↓
Configure Application
    ↓
Configure EFS Mount
    ↓
Create AMI
    ↓
Create Launch Template
    ↓
Create Auto Scaling Group
```

For production environments, EFS mounting can be automated through **User Data/cloud-init** or other instance bootstrapping mechanisms.

### Production Architecture

```text
                       ┌─────────────┐
                       │     ALB     │
                       └──────┬──────┘
                              │
                       ┌──────▼──────┐
                       │     ASG      │
                       └───┬──────┬───┘
                           │      │
                         EC2    EC2
                           │      │
                           └──┬───┘
                              │
                         ┌────▼────┐
                         │   EFS   │
                         │ Shared  │
                         │ Storage │
                         └─────────┘
```

---

# 12. EFS vs EBS — Quick Understanding

| EFS | EBS |
|---|---|
| File storage | Block storage |
| Shared across multiple instances | Typically attached to one EC2 instance at a time |
| Uses NFS | Uses block device |
| Automatically scalable | Provisioned volume capacity |
| Great for shared application files | Great for OS/application/database disks |

### Easy Memory Trick

> **EBS = Disk for EC2**

> **EFS = Shared File System**

---

# 13. Important Production Points

### 1. Network Connectivity

The EC2 instance must be able to reach the EFS mount target.

### 2. Security Group

Allow:

```text
TCP 2049
```

from the appropriate client Security Group.

### 3. Mount Targets

EFS should have appropriate mount targets in the Availability Zones where clients need access.

### 4. Regional EFS

A Regional EFS filesystem is designed for highly available access across Availability Zones within a Region.

### 5. Performance

Choose the appropriate EFS throughput/performance configuration based on workload requirements.

### 6. Backup

Use **AWS Backup** when backup and recovery requirements exist.

---

# 14. Common Troubleshooting

### Problem: Mount fails

Check:

```text
1. Security Group
2. TCP 2049
3. EFS Mount Target
4. VPC/Subnet
5. Route/Network Connectivity
6. DNS Resolution
7. NFS Client Installation
```

Check NFS package:

```bash
dpkg -l | grep nfs-common
```

Check DNS:

```bash
nslookup fs-xxxxxxxx.efs.<region>.amazonaws.com
```

Check connectivity:

```bash
nc -zv fs-xxxxxxxx.efs.<region>.amazonaws.com 2049
```

---

# 15. Interview Questions

### Q1. What is EFS?

**Answer:**  
EFS is a fully managed, scalable NFS-based file system that allows multiple compute resources to access shared files.

### Q2. Which protocol does EFS use?

**Answer:**  
NFS, primarily NFSv4.0 and NFSv4.1.

### Q3. What port does EFS use?

**Answer:**  
TCP **2049**.

### Q4. Can multiple EC2 instances mount the same EFS?

**Answer:**  
Yes. This is one of the primary use cases of EFS.

### Q5. Why would you use EFS with an ASG?

**Answer:**  
To provide shared persistent files to dynamically created EC2 instances.

### Q6. What happens if NFS is blocked?

**Answer:**  
The EC2 instance cannot establish the required NFS connection to mount/access EFS.

### Q7. EFS vs EBS?

**Answer:**  
EFS provides shared file storage over NFS, while EBS provides block storage for EC2.

---

# 16. One-Minute Revision

```text
EFS
 ↓
Shared File Storage
 ↓
NFS
 ↓
TCP 2049
 ↓
Multiple Servers
 ↓
EC2 / ECS / EKS / Fargate / Lambda
 ↓
Scales Automatically
```

## ⭐ Final Memory Line

> **EFS = Shared + Scalable + NFS + Multiple Servers**
