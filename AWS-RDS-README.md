# AWS RDS — Relational Database Service

## 1. What Is Amazon RDS?

**Amazon RDS (Relational Database Service)** is a managed AWS service for creating, operating, backing up, and scaling relational databases.

> **RDS = AWS manages much of the database infrastructure so you can focus on your data and application.**

RDS manages many routine tasks, such as provisioning, backup options, patching options, monitoring, and recovery features. You still manage database users, permissions, schema design, queries, and application access.

---

## 2. Create an RDS MySQL Database

AWS Console flow:

```text
RDS
 ↓
Databases
 ↓
Create database
 ↓
Standard create
 ↓
Engine: MySQL
 ↓
Template: Free tier (if eligible)
 ↓
Configure credentials
 ↓
Configure instance and storage
 ↓
Configure connectivity
 ↓
Create database
```

Example lab settings from the session:

```text
Engine          : MySQL
Template        : Free tier (if eligible)
Instance class  : db.t4g.micro
Storage         : 20 GiB
Credentials     : Manage in AWS Secrets Manager
```

**Important:** Instance-class availability, free-tier eligibility, and pricing depend on Region, account eligibility, and current AWS terms. Review the estimated cost before creating resources.

### Credentials

If you choose AWS Secrets Manager for master credentials:

```text
RDS Database
   ↓
Configuration
   ↓
Master credentials ARN
   ↓
Manage in Secrets Manager
   ↓
Retrieve secret value
```

Avoid putting database passwords directly into scripts, source code, or Git repositories.

---

## 3. Connect RDS to EC2

### Network requirements

```text
EC2
 │
 │ MySQL TCP 3306
 ▼
RDS MySQL
```

Configure the **RDS Security Group** to allow inbound MySQL traffic on TCP port `3306` **from the EC2 instance's Security Group**.

Production best practices:

- Keep RDS private unless there is a specific, reviewed reason to expose it.
- Do not allow MySQL from `0.0.0.0/0`.
- Ensure EC2 and RDS have suitable VPC, subnet, routing, and DNS connectivity.
- A security-group rule alone does not fix missing routing or subnet connectivity.

### Find the RDS endpoint

```text
RDS
 ↓
Databases
 ↓
Select database
 ↓
Connectivity & security
 ↓
Endpoint and port
```

Example endpoint:

```text
database-1-instance-1.us-east-1.rds.amazonaws.com
```

Use your actual endpoint; this example is a placeholder.

### Connect from EC2

Install a compatible MySQL client on EC2, then run:

```bash
mysql -h <RDS_ENDPOINT> -P 3306 -u admin -p
```

Replace `<RDS_ENDPOINT>` with the endpoint shown in the RDS console. Enter the password when prompted.

**Client installation note:** Install only the client if you just need to connect to RDS. You generally do not need a MySQL server on EC2 when the database server is RDS-managed.

---

## 4. Install a MySQL Client

### Amazon Linux

Package names and repository setup vary by Amazon Linux release. Check available packages:

```bash
sudo dnf search mysql
```

Install a compatible client package for your release. For example, where available:

```bash
sudo dnf install mariadb105 -y
```

Verify:

```bash
mysql --version
```

A MariaDB client can connect to many MySQL servers. If you specifically require Oracle MySQL client software, follow the official MySQL repository instructions for your exact operating-system release.

> **Correction:** Installing `mysql-community-server` is unnecessary when EC2 only needs to connect to RDS.

### Ubuntu

Install a client:

```bash
sudo apt update
sudo apt install mysql-client -y
mysql --version
```

You normally do not need `mysql-server` on EC2 when the database server is RDS-managed.

---

## 5. Basic MySQL Hands-On

Connect:

```bash
mysql -h <RDS_ENDPOINT> -P 3306 -u admin -p
```

Create a database:

```sql
CREATE DATABASE raham;
SHOW DATABASES;
USE raham;
```

Create a table:

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255),
    password_hash VARCHAR(255),
    city VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Security improvement:** Store a securely generated password hash in an application, not a user's plaintext password. This demo schema uses `password_hash` to reinforce that practice.

Insert sample rows:

```sql
INSERT INTO users (name, email, password_hash, city)
VALUES
('ramu', 'ramu@example.com', 'DEMO_HASH_1', 'rajavaram'),
('remo', 'remo@example.com', 'DEMO_HASH_2', 'vizag'),
('aparichith', 'aparichith@example.com', 'DEMO_HASH_3', 'hell');
```

These are illustrative values, not real password hashes. In an actual application, generate password hashes with an appropriate password-hashing library.

Query:

```sql
SELECT * FROM users;
```

Exit:

```sql
EXIT;
```

---

# 6. Multi-AZ Deployment

## What Is Multi-AZ?

> **Multi-AZ creates standby database capacity in another Availability Zone to improve availability and support failover.**

Concept:

```text
                Application
                     │
                     ▼
              RDS DB Endpoint
                     │
              ┌──────┴──────┐
              ▼             ▼
        Primary DB       Standby DB
          AZ-1              AZ-2
              │             │
              └── Synchronous
                  replication
```

For the standard RDS Multi-AZ DB instance deployment, AWS maintains a standby in another AZ and can fail over to it if the primary has a qualifying issue.

### Key points

- Main purpose: **high availability and failover**.
- The standby is not normally used to serve read traffic.
- The application generally reconnects using the RDS endpoint after failover.
- Failover can interrupt active connections; applications should retry connections safely.
- Multi-AZ improves availability, but it is not a substitute for backups.

> **Memory trick: Multi-AZ = Availability**

---

# 7. Read Replicas

## What Is a Read Replica?

> **A Read Replica is a copy of a database used to serve read queries and reduce load on the primary database.**

Concept:

```text
                Application
                  /     \
             Writes     Reads
                │         │
                ▼         ▼
           Primary DB  Read Replica
                │         ▲
                └─ Replication
```

### Key points

- Main purpose: **read scaling**.
- Writes usually go to the primary database; read replicas are generally used for reads.
- Replication is commonly asynchronous, so replica data can lag behind the primary.
- Applications must tolerate possible stale reads.
- Depending on engine and configuration, replicas may be created in the same or another Region.
- A read replica can sometimes be promoted to an independent writable database, but this is not the same as automatic Multi-AZ failover.

> **Memory trick: Read Replica = Read Scaling**

---

# 8. Multi-AZ vs Read Replica

| Feature | Multi-AZ DB Instance Deployment | Read Replica |
|---|---|---|
| Main purpose | High availability | Read scaling |
| Typical role | Primary + standby | Primary + readable replica |
| Serves application reads | Primary serves normal reads | Replica can serve reads |
| Replication | Synchronous for standard Multi-AZ DB instance standby | Commonly asynchronous |
| Automatic failover | Designed for automatic failover to standby | Not the same as automatic Multi-AZ failover |
| Read scaling | Standby is not normally used for reads | Yes |
| Memory trick | Availability | Read performance |

**Important:** RDS offers multiple deployment options, including Multi-AZ DB instance deployments and Multi-AZ DB clusters. Their architectures and read capabilities differ. Check the deployment type rather than assuming every Multi-AZ option behaves identically.

---

# 9. RDS Proxy

## What Is RDS Proxy?

> **RDS Proxy is a managed database proxy that pools and reuses database connections between applications and RDS.**

Without a proxy:

```text
Many Application Requests
      │  │  │  │  │
      ▼  ▼  ▼  ▼  ▼
       RDS Database
   Many Direct Connections
```

With RDS Proxy:

```text
Application / Lambda / EC2
             │
             ▼
          RDS Proxy
       Connection Pool
             │
             ▼
          RDS MySQL
```

### Why use RDS Proxy?

- Pools and reuses database connections.
- Helps manage connection spikes.
- Can improve database resilience during certain failovers.
- Can reduce connection-management overhead for applications that open many short-lived connections.
- Supports authentication and Secrets Manager integrations for supported configurations.

### Important points

- RDS Proxy does **not** replace RDS.
- It does not automatically make poorly optimized SQL queries faster.
- It adds a proxy endpoint and can add cost.
- Some session behavior can cause connection pinning and reduce pooling benefits.
- Confirm support for your database engine, version, and configuration.

> **Memory trick: RDS Proxy = Connection Management**

---

# 10. Amazon ElastiCache

## What Is ElastiCache?

> **Amazon ElastiCache is a managed in-memory data store/cache that can reduce repeated database queries and improve application response times.**

Supported engine choices include Valkey, Redis OSS, and Memcached, subject to current AWS service availability and configuration.

Concept:

```text
User Request
     │
     ▼
Application
     │
     ▼
Check Cache
   /     \
 HIT     MISS
  │        │
  ▼        ▼
Return   Query RDS
Data        │
            ▼
        Save Result
         in Cache
```

### Example

A product page is requested repeatedly.

Without a cache:

```text
Every Request → RDS Query
```

With a cache:

```text
First Request → RDS → Cache Result
Next Requests → Cache
```

### Benefits

- Lower latency for cacheable data.
- Fewer repeated database queries.
- Reduced database load.
- Can support sessions, counters, rankings, and other low-latency use cases depending on engine and design.

### Important points

- A cache is not automatically the source of truth.
- Cached data can become stale; design expiration and invalidation.
- Plan for cache misses and cache failure.
- Do not cache sensitive data without appropriate security controls.
- ElastiCache is different from RDS: RDS is the relational database; ElastiCache is a caching/data-store layer.

> **Memory trick: ElastiCache = Fast, Reusable Data**

---

# 11. RDS Proxy vs ElastiCache

| Feature | RDS Proxy | ElastiCache |
|---|---|---|
| Main purpose | Manage/pool database connections | Cache data and reduce repeated reads |
| Sits between app and database? | Yes | Usually accessed by the application as a separate cache |
| Stores query results for reuse? | Not its primary purpose | Yes, when the application caches them |
| Replaces RDS? | No | No |
| Main benefit | Connection management and resilience | Lower latency and reduced database query load |
| Memory trick | Connections | Cached data |

### Typical combined architecture

```text
                 Application
                  /       \
                 ▼         ▼
            ElastiCache  RDS Proxy
           Cached Data      │
                             ▼
                           RDS
```

The application decides when to use the cache and when to query the database.

---

# 12. Manual RDS Snapshots

## What Is a Manual Snapshot?

> **A manual snapshot is a point-in-time backup that you create and retain until you delete it.**

Console flow:

```text
RDS
 ↓
Databases
 ↓
Select Database
 ↓
Actions
 ↓
Take Snapshot
 ↓
Enter Snapshot Name
 ↓
Take Snapshot
```

### Why use manual snapshots?

- Before a risky database change
- Before a major migration
- Before a schema/application release
- To retain a backup for longer
- To create a recovery point you control

### Important points

- Snapshot creation can take time to become fully available.
- Snapshots are stored in AWS-managed storage.
- Manual snapshots remain until you delete them, subject to service and account policies.
- Snapshots can be copied to another Region, subject to supported configurations and encryption/KMS requirements.
- A snapshot is not a substitute for testing restoration.

> **Memory trick: Manual Snapshot = User-Created Backup**

---

# 13. Point-in-Time Recovery (PITR)

## What Is PITR?

> **Point-in-Time Recovery restores a supported RDS database to a selected time within its available backup retention window.**

Example:

```text
10:00 AM → Database healthy
10:15 AM → Accidental data deletion
10:30 AM → Issue discovered
```

With PITR, restore the database to a time before the accidental deletion, such as 10:14 AM, if that time is within the available recovery window.

### Typical flow

```text
Automated Backups + Transaction Logs
                │
                ▼
          Choose Recovery Time
                │
                ▼
       Restore to a NEW DB Instance
                │
                ▼
          Validate the Data
                │
                ▼
       Update Application Endpoint
```

### Important points

- PITR typically creates a **new DB instance**; it does not rewind the existing instance in place.
- The restore time must be within the available backup retention window.
- PITR depends on supported automated backup configuration.
- Test the restored database before directing production traffic to it.
- A manual snapshot is a selected backup point; PITR lets you select a time within the supported recovery window.

> **Memory trick: PITR = Restore to a Specific Time**

---

# 14. Manual Snapshot vs PITR

| Feature | Manual Snapshot | Point-in-Time Recovery |
|---|---|---|
| Recovery point | Snapshot creation point | Selected time within recovery window |
| Created by | User | Restore operation uses automated backups/logs |
| Best for | Before changes, retained restore point | Recover from accidental changes/deletion |
| Restore behavior | Creates a new DB instance | Creates a new DB instance |
| Needs automated backup retention? | Not for the manual snapshot itself | Yes, requires supported automated backup/PITR setup |
| Memory trick | Fixed point | Selected time |

---

# 15. Hands-On: Test a Backup Strategy

For a lab database:

1. Create the RDS database.
2. Insert a few sample records.
3. Take a manual snapshot.
4. Confirm automated backups and retention are configured.
5. Insert or change additional sample records.
6. Practice restoring from the manual snapshot into a new database.
7. Practice PITR to a time before a test change, if available.
8. Validate the restored data.
9. Delete test resources when no longer needed.

**Do not test destructive recovery procedures on production data without an approved plan.**

---

# 16. Production Security Checklist

- Keep RDS private whenever possible.
- Allow TCP 3306 only from approved application Security Groups.
- Store credentials in Secrets Manager or another approved secret store.
- Use least-privilege database accounts.
- Do not store plaintext user passwords; use secure password hashing in the application.
- Enable encryption at rest and in transit where required.
- Configure backups and an appropriate retention period.
- Consider Multi-AZ for availability requirements.
- Consider Read Replicas for read-heavy workloads.
- Consider RDS Proxy for high connection churn or connection spikes.
- Consider ElastiCache for frequently accessed, cacheable data.
- Monitor CPU, storage, connections, latency, and replication lag.
- Test restore and failover procedures.

---

# 17. Troubleshooting RDS Connections

If EC2 cannot connect to RDS, check:

```text
1. Correct RDS endpoint
2. Correct username and password
3. RDS status is Available
4. RDS Security Group allows TCP 3306 from EC2 Security Group
5. EC2 and RDS have network connectivity
6. VPC/subnet/routes/NACLs
7. DNS resolution
8. MySQL client is installed
9. Database user privileges
```

Test port connectivity from EC2 if `nc` is installed:

```bash
nc -zv <RDS_ENDPOINT> 3306
```

Then connect:

```bash
mysql -h <RDS_ENDPOINT> -P 3306 -u admin -p
```

---

# 18. Interview Questions

### Q1. What is Amazon RDS?

**Answer:** A managed AWS service for relational databases that helps with provisioning, operations, backups, and recovery.

### Q2. What is Multi-AZ?

**Answer:** A high-availability deployment that maintains standby capacity in another Availability Zone and supports failover.

### Q3. What is a Read Replica?

**Answer:** A database replica used mainly to scale read traffic.

### Q4. Multi-AZ vs Read Replica?

**Answer:** Multi-AZ primarily improves availability; Read Replicas primarily improve read scalability.

### Q5. What is RDS Proxy?

**Answer:** A managed proxy that pools and reuses database connections.

### Q6. What is ElastiCache?

**Answer:** A managed in-memory cache/data store used to reduce repeated database queries and improve response times.

### Q7. RDS Proxy vs ElastiCache?

**Answer:** RDS Proxy manages database connections; ElastiCache stores reusable data in memory.

### Q8. What is a manual snapshot?

**Answer:** A user-created point-in-time database backup retained until deleted.

### Q9. What is PITR?

**Answer:** Point-in-Time Recovery restores a database to a selected time within the supported recovery window.

### Q10. Does PITR normally overwrite the existing DB instance?

**Answer:** No. It normally creates a new DB instance that must be validated and then used by the application.

### Q11. Which port does MySQL use by default?

**Answer:** TCP 3306.

### Q12. Why keep RDS private?

**Answer:** To reduce exposure and allow database access only from approved application resources.

---

# 19. One-Minute Revision

```text
RDS
│
├── Multi-AZ
│   └── High Availability / Failover
│
├── Read Replica
│   └── Read Scaling
│
├── RDS Proxy
│   └── Connection Pooling
│
├── ElastiCache
│   └── Cache / Reduce Repeated Queries
│
├── Manual Snapshot
│   └── User-Created Backup Point
│
└── PITR
    └── Restore to a Selected Time
```

## Final Memory Lines

- **RDS:** Managed relational database
- **Multi-AZ:** Availability
- **Read Replica:** Read performance
- **RDS Proxy:** Connection management
- **ElastiCache:** Faster access to reusable data
- **Manual Snapshot:** Backup at a chosen point
- **PITR:** Restore to a chosen time
