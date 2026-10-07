
# AWS Session Notes — High Availability & Elastic Load Balancer

## 1. What is HA?

### High Availability

> **High Availability = Designing an application so it remains available even when individual servers, components, or Availability Zones fail.**

Your definition:

> “Maintaining more than one server”

is a good beginner explanation, but for production:

```text
HA ≠ Just Multiple Servers

HA =
Redundancy
+
Multiple AZs
+
Load Balancing
+
Health Checks
+
Automatic Recovery
```

### Example

```text
                 Users
                   │
                   ▼
              Load Balancer
              /            \
             ▼              ▼
          EC2-1           EC2-2
          AZ-1             AZ-2
```

If EC2-1 fails:

```text
Users
  │
  ▼
Load Balancer
  │
  ▼
EC2-2
```

The application can continue serving traffic.

---

# 2. Deploy Application Using User Data

Example:

```bash
#!/bin/bash

apt update -y
apt install nginx git -y

cd /tmp

git clone https://github.com/karishma1521success/swiggy-clone.git

cp -r swiggy-clone/* /var/www/html/
```



---

# 3. Why Do We Need a Load Balancer?

Without a Load Balancer:

```text
Users
  │
  ▼
EC2-1
```

If EC2-1 becomes overloaded or fails:

```text
Users
  │
  ▼
❌ EC2-1
```

The application becomes unavailable.

With a Load Balancer:

```text
                   Users
                     │
                     ▼
              ┌─────────────┐
              │ Load Balancer│
              └──────┬──────┘
                     │
             ┌───────┴───────┐
             ▼               ▼
           EC2-1           EC2-2
```

ELB distributes incoming traffic across registered targets and uses health checks to route traffic to healthy targets. [AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html?utm_source=chatgpt.com)

---

# 4. What is Traffic?

### Traffic

> **Traffic = Requests coming into the application and responses going back to clients.**

```text
User
 │
 │ Request
 ▼
Load Balancer
 │
 ▼
Server
 │
 │ Response
 ▼
User
```

---

# 5. Why Use a Load Balancer?

Main purposes:

### 1. Load Distribution

Distribute incoming requests across multiple targets.

### 2. High Availability

Avoid depending on a single application server.

### 3. Fault Tolerance

If a target becomes unhealthy, the load balancer can stop sending traffic to it.

### 4. Scalability

You can add/remove targets as application demand changes.

### 5. Health Monitoring

ELB continuously checks registered targets and routes traffic to healthy targets. [AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html?utm_source=chatgpt.com)

---

# 6. Types of AWS Load Balancers

| Load Balancer | Layer | Main Use |
|---|---:|---|
| **Application Load Balancer (ALB)** | L7 | HTTP/HTTPS applications |
| **Network Load Balancer (NLB)** | L4 | TCP/TLS/UDP/QUIC, high-performance networking |
| **Gateway Load Balancer (GWLB)** | L3 | Security/network appliances |
| **Classic Load Balancer (CLB)** | Legacy | Previous-generation load balancing |

AWS identifies ALB as Layer 7, NLB as Layer 4, and GWLB as Layer 3; Classic Load Balancer is the previous generation. [AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/?utm_source=chatgpt.com)

### ⭐ Production Recommendation

For a modern web application:

```text
HTTP/HTTPS Application
        ↓
       ALB
```

---

# 7. HTTP & HTTPS

### HTTP

```text
Port: 80
```

**HTTP = Hypertext Transfer Protocol**

### HTTPS

```text
Port: 443
```

**HTTPS = HTTP over TLS**

> HTTPS provides encrypted communication between the client and the load balancer/server.

---

# 8. Target vs Target Group

### Target

> **One backend resource receiving traffic.**

Example:

```text
EC2-1
```

### Target Group

> **A logical collection of targets serving an application/service.**

Example:

```text
Target Group
├── EC2-1
├── EC2-2
└── EC2-3
```

### Memory Trick

> **Target = One**

> **Target Group = Many**

---

# 9. ALB Architecture

```text
                    Internet
                       │
                       ▼
                ┌─────────────┐
                │     ALB     │
                └──────┬──────┘
                       │
                  Listener
                  HTTP : 80
                  HTTPS : 443
                       │
                       ▼
                Target Group
               /      |      \
              ▼       ▼       ▼
            EC2-1   EC2-2   EC2-3
```

The ALB uses **listeners** to accept connections and **listener rules** to determine where traffic should go. Target groups contain the registered backend targets. [AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/Welcome.html?utm_source=chatgpt.com)

---

# 10. Practical — Create ALB

### Step 1 — Create Servers

Create:

```text
EC2-1
EC2-2
```

Deploy the Swiggy application on both.

---

### Step 2 — Create Load Balancer

```text
EC2
 ↓
Load Balancers
 ↓
Create Load Balancer
 ↓
Application Load Balancer
```

Configuration:

```text
Name:
swiggy

Scheme:
Internet-facing

IP Address Type:
IPv4
```

---

### Step 3 — Network Mapping

Select multiple Availability Zones.

Example:

```text
AZ-1
AZ-2
```

### ⭐ Important

For a highly available ALB, use subnets in **at least two Availability Zones**.

---

### Step 4 — Security Group

Allow:

```text
HTTP   → 80
HTTPS  → 443
```

Example:

```text
Internet
   │
   ▼
ALB Security Group
   │
   ▼
EC2 Security Group
```

### 🔐 Production Best Practice

Don't necessarily expose port 80/443 directly to EC2 from the entire Internet.

A better architecture is:

```text
Internet
   │
   ▼
ALB SG
   │
   ▼
EC2 SG
```

EC2's Security Group should allow application traffic **from the ALB Security Group**.

---

# 11. Create Target Group

```text
Target Groups
 ↓
Create Target Group
```

Example:

```text
Target Group Name:
amazon-target-group
```

Select:

```text
EC2-1
EC2-2
```

Then:

```text
Include as pending below
 ↓
Create Target Group
```

Attach the target group to the ALB listener.

---

# 12. Adding a New Server

Create:

```text
EC2-3
```

Deploy the application.

Then:

```text
Target Group
 ↓
Select Target Group
 ↓
Actions
 ↓
Register Targets
 ↓
Select EC2-3
 ↓
Include as pending
 ↓
Register pending targets
```

Once registered and healthy, the load balancer can route traffic to it.

---

# 13. Health Checks ⭐⭐⭐

This is one of the most important production concepts.

The Load Balancer periodically checks whether targets are healthy.

Example:

```text
ALB
 │
 ├── Health Check → EC2-1 → Healthy ✅
 │
 ├── Health Check → EC2-2 → Healthy ✅
 │
 └── Health Check → EC2-3 → Unhealthy ❌
```

Traffic:

```text
ALB
 ├── EC2-1 ✅
 └── EC2-2 ✅

EC2-3 ❌ ← No new traffic
```

ELB health checks allow traffic to be routed only to healthy registered targets. [AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html?utm_source=chatgpt.com)

### Example ALB Health Check

```text
Protocol: HTTP
Port: 80
Path: /
```

The ALB can check:

```text
http://EC2-IP/
```

If the target repeatedly fails health checks, it becomes unhealthy.

---

# 14. Sticky Sessions ⭐⭐⭐

### Problem

Normally:

```text
Request 1 → EC2-1
Request 2 → EC2-2
Request 3 → EC2-1
```

If an application stores session information locally on EC2-1, the user may have problems when the next request goes to EC2-2.

### Sticky Session

```text
User
 │
 ▼
ALB
 │
 ▼
EC2-1
```

Subsequent requests from that client can continue going to the same target.

For ALB, sticky sessions use cookies and cause subsequent requests from the same client to bypass the normal target-selection algorithm. [AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html?utm_source=chatgpt.com)

### ⚠️ Production Best Practice

Don't use stickiness as a substitute for proper application architecture.

Prefer:

```text
Application
     ↓
Stateless
     ↓
Session stored externally
     ↓
Redis / DynamoDB / Database
```

Then any healthy EC2 can handle the request.

---

# 15. Load Balancing Algorithms ⭐⭐⭐

Your classroom list contains algorithms that should **not all be presented as ALB algorithms**.

For **Application Load Balancer**, the current target-group algorithms are:

### 1. Round Robin — Default

```text
Request 1 → EC2-1
Request 2 → EC2-2
Request 3 → EC2-3
Request 4 → EC2-1
```

Good when targets have similar capacity and requests have similar complexity.

### 2. Least Outstanding Requests

Routes requests toward targets with fewer currently outstanding requests.

Useful when:

```text
Request processing time varies
OR
Target capacity varies
```

### 3. Weighted Random

Selects healthy targets in a randomized weighted manner and supports Automatic Target Weights anomaly mitigation.

AWS currently documents these three ALB target-group routing algorithms, with **Round Robin as the default**. [AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html?utm_source=chatgpt.com)

---

# 16. How to Change ALB Algorithm

```text
Target Groups
 ↓
Select Target Group
 ↓
Attributes
 ↓
Load Balancing Algorithm
 ↓
Select:
   Round Robin
   Least Outstanding Requests
   Weighted Random
```

---

# 17. Cross-Zone Load Balancing ⭐⭐⭐

Very important for HA.

Imagine:

```text
AZ-1                 AZ-2

ALB                  ALB
 │                    │
 ├─ EC2-1             ├─ EC2-3
 └─ EC2-2             └─ EC2-4
```

With cross-zone load balancing, load balancer nodes can distribute requests across healthy targets in enabled Availability Zones.

For **ALB**, cross-zone load balancing is enabled by default. [AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-subnets.html?utm_source=chatgpt.com)

### Simple Definition

> **Cross-zone load balancing = Distribute traffic across healthy targets in multiple Availability Zones.**

---

# 18. Routing Types ⭐⭐⭐

## Host-Based Routing

Route based on the hostname.

Example:

```text
www.paytm.com
        ↓
Website Target Group

api.paytm.com
        ↓
API Target Group

admin.paytm.com
        ↓
Admin Target Group
```

---

## Path-Based Routing

Route based on URL path.

Example:

```text
www.paytm.com/movies
        ↓
Movies Target Group

www.paytm.com/recharge
        ↓
Recharge Target Group

www.paytm.com/payments
        ↓
Payments Target Group
```

### Powerful Architecture

```text
                    ALB
                     │
             Listener HTTPS:443
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       /movies    /recharge   /payments
          │          │          │
          ▼          ▼          ▼
       TG-Movie   TG-Recharge TG-Payment
```

---

# 19. Listener ⭐

A listener waits for incoming connection requests.

Example:

```text
ALB
 │
 ├── HTTP : 80
 │
 └── HTTPS : 443
```

A listener can contain rules such as:

```text
IF host = api.example.com
        ↓
API Target Group

IF path = /movies/*
        ↓
Movies Target Group
```

---

# 20. Provisioning

### Provisioning

> **Provisioning = Creating and configuring infrastructure/resources.**

Example:

```text
Provision EC2
Provision ALB
Provision Target Group
Provision VPC
```

---

# 21. Failure Scenario

### Without Load Balancer

```text
Users
  │
  ▼
EC2-1 ❌
```

Application unavailable.

### With ALB

```text
                ALB
                 │
          ┌──────┴──────┐
          ▼             ▼
       EC2-1 ❌       EC2-2 ✅
                         │
                         ▼
                       Users
```

The unhealthy target is removed from normal routing after health checks identify the failure.

---

# 22. Important Corrections to Your Notes

### ❌ “Load Balancer distributes traffic equally”

Better:

> **A Load Balancer distributes traffic according to its configured routing algorithm and only routes to healthy targets.**

It doesn't always mean equal traffic.

---

### ❌ “At least 2 servers are required to create a Load Balancer”

Better:

> **A target group can technically be created with one target, and an ALB can be created without two backend servers. However, for real HA, use multiple targets across multiple AZs.**

---

### ❌ “Servers automatically add to LB”

Better:

> **Instances must be registered with the target group, or an integration such as Auto Scaling can automatically register/deregister instances as they launch or terminate.**

---

### ❌ “LB needs at least 2 AZs”

For an internet-facing ALB, you should design it across **at least two AZs for high availability**. The important production principle is multi-AZ deployment, not merely checking two boxes during creation.

---

# 23. Real-World Architecture ⭐⭐⭐

```text
                         USERS
                           │
                           ▼
                    Route 53 / DNS
                           │
                           ▼
                  ┌────────────────┐
                  │      ALB       │
                  │   HTTPS :443   │
                  └───────┬────────┘
                          │
                    Listener Rules
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
        /api/*        /movies/*      /payment/*
            │             │             │
            ▼             ▼             ▼
        API TG         Movie TG      Payment TG
            │             │             │
         ┌──┴──┐       ┌──┴──┐       ┌──┴──┐
         ▼     ▼       ▼     ▼       ▼     ▼
        EC2   EC2     EC2   EC2     EC2   EC2
         AZ1   AZ2     AZ1   AZ2     AZ1   AZ2
```

This gives you:

**HA + Multi-AZ + Load Balancing + Health Checks + Routing + Scalability**

---

# 🎯 Interview Questions

### Q1. What is High Availability?

> Designing an application with redundancy across failure domains so it remains available when individual components fail.

### Q2. Why use a Load Balancer?

> To distribute traffic across multiple targets, improve availability, detect unhealthy targets, and support scalable architectures.

### Q3. What happens if one EC2 instance fails?

> The load balancer's health checks detect the unhealthy target and stop routing new traffic to it.

### Q4. What is a Target Group?

> A logical group of registered backend targets that receive traffic from a load balancer.

### Q5. What is Sticky Session?

> A mechanism that keeps subsequent requests from a client routed to the same target using session persistence.

### Q6. What is Cross-Zone Load Balancing?

> Distributing traffic across healthy targets in multiple Availability Zones.

### Q7. What is the default ALB routing algorithm?

> **Round Robin.** [AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html?utm_source=chatgpt.com)

### Q8. What are the ALB routing algorithms?

> Round Robin, Least Outstanding Requests, and Weighted Random. [AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html?utm_source=chatgpt.com)

### Q9. What is Host-Based Routing?

> Routing requests based on the hostname.

### Q10. What is Path-Based Routing?

> Routing requests based on the URL path.

---

# 🧠 One-Minute Revision

```text
HA
 ↓
Multiple Servers + Multiple AZs
 ↓
Load Balancer
 ↓
Listener
 ↓
Target Group
 ↓
Health Checks
 ↓
Healthy Targets
```

### Remember:

```text
ALB        → Layer 7 → HTTP/HTTPS
NLB        → Layer 4 → TCP/TLS/UDP/QUIC
GWLB       → Network/Security Appliances
CLB        → Legacy
```

```text
Target              → One Backend
Target Group        → Group of Backends
Listener            → Accepts Traffic
Health Check        → Finds Healthy Targets
Sticky Session      → Same Client → Same Target
Cross-Zone          → Multiple AZ Targets
Round Robin         → Default ALB Algorithm
```

### 🔥 Final Memory Line

> **ALB = Receive → Inspect → Route → Health Check → Send to Healthy Target.**
