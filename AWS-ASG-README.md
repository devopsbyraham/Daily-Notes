# AWS Session Notes — Auto Scaling Group (ASG)

## 1. What is ASG?

**ASG = Auto Scaling Group**

> **ASG automatically adds or removes EC2 instances based on demand and maintains the desired number of healthy instances.**

### Simple Flow

```text
High Load
   ↓
Add EC2 Instances
   ↓
Handle More Traffic

Low Load
   ↓
Remove EC2 Instances
   ↓
Reduce Cost
```

### ⭐ Most Important Point

If an EC2 instance managed by an ASG becomes unhealthy or is terminated, the ASG can launch a replacement to maintain the group's desired capacity.

---

# 2. Why Do We Use ASG?

Use ASG when application demand changes over time.

```text
Low Traffic
    ↓
2 Servers

High Traffic
    ↓
5 Servers

Very High Traffic
    ↓
10 Servers
```

### Main Benefits

- Automatic scaling
- High availability
- Fault recovery
- Cost optimization
- Maintains desired capacity
- Integrates with Load Balancers
- Works with CloudWatch metrics

### 🧠 Memory Trick

> **ASG = Automatically Add + Remove + Replace EC2**

---

# 3. ASG Architecture

```text
                         Users
                           │
                           ▼
                    ┌─────────────┐
                    │     ALB     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │     ASG      │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
            EC2-1        EC2-2        EC2-3
              │            │            │
              └────────────┼────────────┘
                           │
                       CloudWatch
                           │
                           ▼
                    Scaling Policy
```

---

# 4. Horizontal vs Vertical Scaling

## Horizontal Scaling

> **Add or remove servers.**

```text
2 EC2
 ↓
4 EC2
 ↓
8 EC2
```

Also called:

> **Scale Out / Scale In**

Commonly used for:

- Web applications
- APIs
- Stateless application servers
- Microservices

---

## Vertical Scaling

> **Increase or decrease the capacity of an existing server.**

Example:

```text
t3.small
   ↓
t3.medium
   ↓
t3.large
```

This means increasing resources such as:

```text
CPU
Memory
```

Also called:

> **Scale Up / Scale Down**

### Your practical process

```text
Stop Instance
   ↓
Actions
   ↓
Instance Settings
   ↓
Change Instance Type
   ↓
Select New Type
   ↓
Start Instance
```

### Important Production Point

Vertical scaling normally requires changing the instance type, and depending on the resource/workload, this can involve downtime or application considerations.

---

# 5. Web/App vs Database Scaling

Your rule is a good beginner guideline:

```text
Web / App
   ↓
Usually Horizontal Scaling

Database
   ↓
Often Vertical Scaling
```

### ⚠️ Architect-Level Improvement

Don't treat this as an absolute rule.

Modern databases can also scale horizontally using:

- Read replicas
- Sharding
- Partitioning
- Distributed databases
- Managed database scaling features

So the better statement is:

> **Stateless application tiers commonly scale horizontally, while databases may use vertical and/or horizontal scaling depending on architecture.**

---

# 6. Launch Template ⭐⭐⭐

A **Launch Template** defines how EC2 instances created by the ASG should be launched.

It can include:

```text
AMI
Instance Type
Key Pair
Security Group
IAM Role
Storage
Network Settings
User Data
Metadata Options
```

### Architecture

```text
Launch Template
       │
       ▼
     ASG
       │
 ┌─────┼─────┐
 ▼     ▼     ▼
EC2   EC2   EC2
```

### Why?

Without a common template:

```text
EC2-1 → Ubuntu + t3.medium
EC2-2 → Amazon Linux + t3.large
EC2-3 → Different configuration
```

With a Launch Template:

```text
             Launch Template
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
     EC2-1        EC2-2        EC2-3
      Same         Same         Same
   baseline      baseline      baseline
```

> **Launch Template = Blueprint for EC2 instances.**

---

# 7. AMI + Launch Template + ASG

These three concepts connect together:

```text
AMI
 ↓
What is installed?
 ↓
Launch Template
 ↓
How should EC2 launch?
 ↓
ASG
 ↓
How many EC2 instances should run?
```

### ⭐ Interview Memory

> **AMI = Server Image**

> **Launch Template = Server Configuration**

> **ASG = Server Quantity & Lifecycle**

---

# 8. Create an AMI for ASG

Your practical approach:

```text
Create EC2
   ↓
Install Application
   ↓
Configure Server
   ↓
Create AMI
   ↓
Use AMI in Launch Template
```

Example:

```bash
git clone https://github.com/Ironhack-Archive/online-clone-amazon.git
```

Then create the AMI.

---

# 9. Create Auto Scaling Group

Go to:

```text
EC2
 ↓
Auto Scaling Groups
 ↓
Create Auto Scaling Group
```

Example:

```text
ASG Name:
SWIGGY
```

---

# 10. Create Launch Template

```text
Create Launch Template
```

Example:

```text
Name:
SWIGGY-TEMPLATE
```

Configure it similar to an EC2 launch:

```text
AMI
Instance Type
Security Group
IAM Role
Storage
User Data
```

---

# 11. Availability Zones & Subnets

Example:

```text
us-east-1a
us-east-1b
us-east-1c
```

Select subnets across multiple Availability Zones.

### Why?

```text
AZ-1 → EC2
AZ-2 → EC2
AZ-3 → EC2
```

If one AZ has a failure:

```text
AZ-1 ❌
   ↓
AZ-2 + AZ-3
   ↓
Application continues
```

### ⭐ Production Principle

> **Deploy ASG instances across multiple Availability Zones for higher availability.**

---

# 12. ASG + Load Balancer

Attach the ASG to a target group/load balancer.

```text
                    ALB
                     │
                     ▼
                    ASG
              ┌──────┼──────┐
              ▼      ▼      ▼
             EC2    EC2    EC2
```

When ASG launches a new instance, the instance can be registered with the target group.

When an instance is terminated, it is removed from the target group.

This creates:

> **Load Balancing + Auto Scaling + High Availability**

---

# 13. Desired vs Minimum vs Maximum ⭐⭐⭐

This is extremely important.

Suppose:

```text
Desired = 2
Minimum = 2
Maximum = 10
```

### Desired Capacity

> **How many instances the ASG wants running right now.**

```text
Desired = 2

EC2-1
EC2-2
```

### Minimum Capacity

> **The minimum number of instances the ASG is allowed to maintain.**

```text
Minimum = 2

ASG should not normally scale below 2
```

### Maximum Capacity

> **The maximum number of instances the ASG can scale to.**

```text
Maximum = 10

ASG cannot scale beyond 10
```

### Visual

```text
Minimum                    Maximum
  2                           10
  │                            │
  ▼                            ▼
  ┌────────────────────────────┐
  │  2  3  4  5  6  7  8  9 10│
  └────────────────────────────┘
             ▲
             │
          Desired
            2
```

### Example

```text
Desired = 2
Min     = 2
Max     = 10
```

High load:

```text
2 → 4 → 6 → 8 → 10
```

Low load:

```text
10 → 8 → 6 → 4 → 2
```

The ASG respects the configured minimum and maximum boundaries.

---

# 14. Automatic Scaling Policy ⭐⭐⭐

Scaling policies tell the ASG:

> **When should I add or remove instances?**

Example:

```text
CPU > Target
     ↓
Scale Out

CPU < Target
     ↓
Scale In
```

---

# 15. Target Tracking Scaling Policy

Example:

```text
Policy:
Target Tracking

Metric:
Average CPU Utilization

Target:
50%
```

Conceptually:

```text
CPU > 50%
   ↓
Add instances

CPU ≈ 50%
   ↓
Maintain capacity

CPU < 50%
   ↓
Remove instances
```

### Important

The ASG works with **CloudWatch metrics** to evaluate scaling conditions.

---

# 16. Common ASG Scaling Policy Types ⭐⭐⭐

There isn't just one scaling policy.

## 1. Target Tracking

> Maintain a metric around a target value.

Example:

```text
Average CPU = 50%
```

Good for simple automatic scaling.

---

## 2. Step Scaling

> Scale by different amounts depending on how far the metric moves beyond a threshold.

Example:

```text
CPU 50–70% → +1 instance

CPU 70–85% → +2 instances

CPU >85%   → +3 instances
```

---

## 3. Simple Scaling

> Scale based on a single CloudWatch alarm and use a cooldown period before another scaling action.

For modern architectures, **target tracking or step scaling is generally preferred** for many use cases.

---

## 4. Scheduled Scaling

> Scale based on a predictable schedule.

Example:

```text
9:00 AM
 ↓
Increase capacity

10:00 PM
 ↓
Reduce capacity
```

Useful for predictable traffic patterns.

---

## 5. Predictive Scaling

> Uses historical patterns to forecast future demand and proactively scale capacity.

Useful when traffic has predictable patterns.

---

# 17. CloudWatch + ASG ⭐⭐⭐

Question:

> How does ASG know CPU utilization?

Answer:

```text
EC2
 ↓
CloudWatch Metrics
 ↓
Scaling Policy
 ↓
ASG
 ↓
Add / Remove Instances
```

Example:

```text
Average CPU = 80%
       ↓
CloudWatch
       ↓
Target Tracking Policy
       ↓
ASG
       ↓
Launch EC2
```

---

# 18. Scaling Metrics

Common metrics can include:

### CPU Utilization

```text
High CPU
 ↓
Scale Out
```

### Network In

```text
High Incoming Traffic
 ↓
Scale Out
```

### Network Out

```text
High Outgoing Traffic
 ↓
Scale Out
```

### Application Load Balancer Request Count Per Target

Useful when application demand is better represented by requests rather than CPU.

> **Request Count Per Target** is particularly useful for web applications behind an ALB.

### ⚠️ Improvement

Don't automatically choose CPU for every application.

For example:

```text
CPU-based application
     → CPU may be useful

Web application
     → Request Count Per Target may be better

Network-heavy application
     → Network metrics may be useful
```

Choose the metric that best represents **actual application demand**.

---

# 19. Instance Warmup ⭐⭐⭐

### Warmup

When a new instance launches:

```text
Launch EC2
   ↓
Boot OS
   ↓
Install/Start Application
   ↓
Application becomes ready
```

During instance warmup, the ASG can avoid treating the new instance's metrics as representative of a fully operational instance.

### Simple Definition

> **Instance warmup = Time allowed for a newly launched instance to initialize before its metrics are considered for scaling decisions.**

---

# 20. Cooldown Period ⭐⭐⭐

Your definition needs correction.

You wrote:

> Amount of time taken by server before delete when load is decreased.

Better:

> **Cooldown = A period intended to prevent repeated scaling actions from happening too quickly after a scaling activity.**

It is **not specifically the time before deleting a server**.

### Example

```text
Scale Out
   ↓
Cooldown
   ↓
Allow system to stabilize
   ↓
Evaluate again
```

### ⚠️ Important Modern AWS Point

For **target tracking and step scaling**, the scaling behavior uses instance warmup and policy-specific mechanisms; the old generic cooldown concept should not be treated as the universal waiting period for every ASG policy.

---

# 21. Warm Pool ⭐⭐⭐

A **Warm Pool** keeps pre-initialized EC2 instances available so they can enter service faster when ASG needs additional capacity.

### Normal Scaling

```text
Need EC2
   ↓
Launch Instance
   ↓
Boot
   ↓
Initialize Application
   ↓
Ready
```

Potentially slower.

### Warm Pool

```text
                 Warm Pool
              ┌─────────────┐
              │ Pre-initialized│
              │   Instances   │
              └──────┬──────┘
                     │
                Scale Out
                     ↓
              Move to ASG
                     ↓
                  Serve
```

### Simple Definition

> **Warm Pool = Pre-initialized EC2 capacity kept ready for faster scale-out.**

Useful when:

- Instances take a long time to boot
- Applications have lengthy initialization
- Fast scale-out is important

---

# 22. ASG Health Checks ⭐⭐⭐

ASG can use health checks to determine whether an instance is healthy.

Common types include:

```text
EC2 Health Check
ELB Health Check
```

If configured to use ELB health checks:

```text
ALB
 ↓
EC2 unhealthy
 ↓
ASG detects unhealthy instance
 ↓
Terminate
 ↓
Launch replacement
```

### Important Architecture

```text
                 ALB
                  │
             Health Check
                  │
                  ▼
                 ASG
                  │
          ┌───────┴───────┐
          ▼               ▼
       Healthy          Unhealthy
          │               │
        Keep           Replace
```

This is one of the most important **self-healing** capabilities of ASG.

---

# 23. What Happens If You Manually Delete an ASG Instance?

Suppose:

```text
Desired = 2
```

Current:

```text
EC2-1
EC2-2
```

You manually terminate EC2-1.

Then:

```text
Current = 1
Desired = 2
```

ASG sees the capacity deficit and can launch:

```text
EC2-3
```

Final:

```text
EC2-2
EC2-3
```

### ⭐ Important

> **ASG tries to maintain the desired capacity.**

---

# 24. How to Remove ASG Instances Permanently

Don't simply terminate instances and expect them to stay deleted.

If you want the ASG to run fewer instances:

```text
Change Desired Capacity
```

Example:

```text
Desired = 5

Change to:

Desired = 3
```

Then ASG can scale in from 5 → 3.

Or modify the minimum/maximum values as required.

### To Delete the Entire ASG

Delete the Auto Scaling Group when the group itself is no longer required.

---

# 25. Notification

You can configure notifications for ASG events.

Example:

```text
ASG
 ↓
SNS
 ↓
Email
```

Useful events can include:

```text
Instance Launch
Instance Termination
Scaling Events
```

---

# 26. Complete ASG Creation Flow

```text
Create AMI
   ↓
Create Launch Template
   ↓
Create Auto Scaling Group
   ↓
Select VPC/Subnets
   ↓
Select Multiple AZs
   ↓
Attach Load Balancer
   ↓
Select/Create Target Group
   ↓
Set Desired Capacity
   ↓
Set Minimum Capacity
   ↓
Set Maximum Capacity
   ↓
Configure Scaling Policy
   ↓
Configure Health Checks
   ↓
Configure Notifications
   ↓
Create ASG
```

---

# 27. Practical Configuration

Example:

```text
ASG Name:
SWIGGY
```

### Launch Template

```text
Name:
SWIGGY-TEMPLATE

AMI:
SWIGGY-AMI

Instance Type:
t3.micro / appropriate type

Security Group:
Application SG
```

### Network

```text
us-east-1a
us-east-1b
us-east-1c
```

### Capacity

```text
Desired = 2
Minimum = 2
Maximum = 10
```

### Scaling

```text
Policy:
Target Tracking

Metric:
Average CPU Utilization

Target:
50%
```

---

# 28. Generate Load

Your practical command:

```bash
sudo apt update -y
sudo apt install stress -y

stress -c 10
```

### Better Production/Lab Approach

Monitor CPU first:

```bash
top
```

or:

```bash
htop
```

Then observe:

```text
EC2
 ↓
CloudWatch
 ↓
ASG
 ↓
Scaling Activity
```

You should see the ASG launch additional instances when the configured scaling conditions are met.

---

# 29. ASG + ALB + CloudWatch

This is the architecture you should remember:

```text
                         USERS
                           │
                           ▼
                    ┌─────────────┐
                    │     ALB     │
                    └──────┬──────┘
                           │
                     Target Group
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
             EC2          EC2          EC2
              │            │            │
              └────────────┼────────────┘
                           │
                      ASG Capacity
                           │
                           ▼
                     CloudWatch
                           │
                           ▼
                   Scaling Policy
                           │
                    ┌──────┴──────┐
                    ▼             ▼
                 Scale Out     Scale In
```

This gives:

> **Load Balancing + Auto Scaling + Health Checks + Self-Healing + High Availability**

---

# 30. Important Corrections to Your Notes

### ❌ “ASG is used only when load changes frequently”

Better:

> **ASG is useful whenever you need EC2 capacity to automatically adjust, maintain availability, or replace unhealthy instances.**

It can also be useful when load is predictable through scheduled scaling.

---

### ❌ “Vertical scaling = ASG”

Not exactly.

> **ASG primarily manages horizontal scaling of EC2 capacity.**

Changing an individual EC2 instance type is vertical scaling.

---

### ❌ “Template contains configuration of a server which is created by ASG”

Better:

> **Launch Template defines how instances launched by the ASG should be configured.**

---

### ❌ “Cooldown = time before deleting server”

Correct:

> **Cooldown helps prevent rapid repeated scaling actions; instance warmup is the period used for newly launched instances to initialize.**

---

# 31. Interview Questions

### Q1. Why do we need ASG?

> To automatically adjust EC2 capacity, maintain desired capacity, replace unhealthy instances, and support high availability.

### Q2. What is horizontal scaling?

> Adding or removing instances.

### Q3. What is vertical scaling?

> Increasing or decreasing the resources of an existing instance.

### Q4. What is Desired Capacity?

> The number of instances the ASG currently aims to maintain.

### Q5. What is Minimum Capacity?

> The lowest number of instances the ASG should normally maintain.

### Q6. What is Maximum Capacity?

> The maximum number of instances the ASG can launch.

### Q7. How does ASG know when to scale?

> Scaling policies evaluate metrics such as CloudWatch metrics and determine whether to scale out or scale in.

### Q8. How do all ASG instances get the same configuration?

> The ASG launches instances using a Launch Template or Launch Configuration where applicable.

### Q9. What happens if an ASG instance fails?

> ASG can detect the unhealthy instance and launch a replacement to maintain desired capacity.

### Q10. What is instance warmup?

> The time allowed for a newly launched instance to initialize before it is fully considered in scaling decisions.

### Q11. What is a Warm Pool?

> A pool of pre-initialized EC2 instances maintained by an ASG to reduce scale-out latency.

### Q12. What is Target Tracking?

> A scaling policy that automatically adjusts capacity to maintain a selected metric near a target value.

---

# 🔥 ASG One-Minute Revision

```text
ASG
 │
 ├── Launch Template → HOW to create EC2
 │
 ├── Desired          → HOW MANY now
 │
 ├── Minimum          → LOWEST capacity
 │
 ├── Maximum          → HIGHEST capacity
 │
 ├── Scaling Policy   → WHEN to scale
 │
 ├── Health Check     → IS instance healthy?
 │
 ├── Warmup           → NEW instance initialization
 │
 ├── Warm Pool        → PRE-INITIALIZED capacity
 │
 └── CloudWatch       → METRICS
```

### ⭐ Final Memory Line

> **ASG = Automatically maintain the right number of healthy EC2 servers based on application demand.**
