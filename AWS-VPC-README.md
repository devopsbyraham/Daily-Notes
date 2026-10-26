# AWS VPC Networking — Complete Student Notes and Practical Labs

Beginner-friendly AWS networking notes with console steps, diagrams, commands, endpoint and Transit Gateway theory, VPC peering, Flow Logs, troubleshooting, and interview questions.

---

## 1. What is a VPC?

**Amazon VPC (Virtual Private Cloud)** is a logically isolated virtual network in AWS where you launch resources such as EC2 instances, databases, and application servers.

| Real-world example | AWS concept |
|---|---|
| Campus boundary wall | VPC |
| Rooms inside the campus | Subnets |
| Main gate to the internet | Internet Gateway (IGW) |
| Direction board | Route table |
| Security guard for a resource | Security Group |
| Network-level access control list | Network ACL (NACL) |
| Outbound gateway for private IPv4 resources | NAT Gateway |

**Memory line:** VPC is your AWS network; subnets divide it; route tables decide where traffic goes.

## 2. CIDR basics

CIDR stands for **Classless Inter-Domain Routing**. It represents an IP address range using an address and prefix length.

- `10.0.0.0/16` contains 65,536 IPv4 addresses in total.
- `10.0.1.0/24` contains 256 IPv4 addresses in total.
- AWS reserves five IPv4 addresses in every subnet, so not all subnet addresses can be assigned to resources.

Lab ranges:

| Resource | CIDR |
|---|---|
| Swiggy VPC | `10.0.0.0/16` |
| Web subnet | `10.0.1.0/24` |
| App subnet | `10.0.2.0/24` |

Subnets in the same VPC must have non-overlapping CIDRs. When connecting VPCs, avoid overlapping CIDR ranges because overlapping routes make communication difficult or impossible.

## 3. Main VPC components

- **VPC:** isolated network boundary.
- **Subnet:** a range of IP addresses in one Availability Zone.
- **Route table:** decides the next hop for traffic matching a destination CIDR.
- **Internet Gateway (IGW):** enables internet routing for resources with suitable public addressing and security rules.
- **NAT Gateway:** allows resources in private IPv4 subnets to initiate connections to external IPv4 destinations without accepting unsolicited inbound internet connections.
- **Security Group:** stateful virtual firewall attached to resources/network interfaces.
- **Network ACL (NACL):** stateless subnet-level filter; inbound and outbound rules are evaluated separately.
- **Elastic IP:** static public IPv4 address that can be associated with supported AWS resources.
- **VPC Flow Logs:** records metadata about IP traffic to/from supported network interfaces, subnets, or VPCs.
- **VPC Endpoint:** private connectivity to supported AWS services or endpoint services.
- **VPC Peering:** private point-to-point connectivity between two VPCs.
- **Transit Gateway:** central routing hub for connecting multiple VPCs and supported network attachments.

## 4. Public vs. private subnet

A subnet is considered **public** when its associated route table includes a route to an Internet Gateway, commonly `0.0.0.0/0 → igw-...` for IPv4. An instance also needs suitable public IPv4 addressing and security rules to communicate directly over the internet.

A **private subnet** has no direct route to an Internet Gateway. It may use `0.0.0.0/0 → NAT Gateway` for outbound IPv4 internet access, or use VPC endpoints for private access to supported AWS services.

A subnet is not public simply because its name says “public”; routing and resource configuration determine actual connectivity.

## 5. Practical Lab 1 — Create the Swiggy VPC and public web subnet

### Step 1: Create the VPC
1. AWS Console → **VPC** → **Your VPCs** → **Create VPC**.
2. Name: `SWIGGY`.
3. IPv4 CIDR: `10.0.0.0/16`.
4. Create the VPC.

### Step 2: Create the public web subnet
1. VPC → **Subnets** → **Create subnet**.
2. VPC: `SWIGGY`.
3. Name: `WEB-SUBNET`.
4. CIDR: `10.0.1.0/24`.
5. Choose an Availability Zone.
6. Create the subnet.
7. Select the subnet → **Actions → Edit subnet settings** → enable auto-assign public IPv4 address if desired.

### Step 3: Create and attach an Internet Gateway
1. VPC → **Internet gateways** → **Create internet gateway**.
2. Name: `SWIGGY-IGW`.
3. Create it.
4. Select it → **Actions → Attach to VPC** → choose `SWIGGY`.

### Step 4: Create and configure the public route table
1. VPC → **Route tables** → **Create route table**.
2. Name: `SWIGGY-WEB-RTB`; VPC: `SWIGGY`.
3. Create it.
4. Open **Routes → Edit routes → Add route**.
5. Destination: `0.0.0.0/0`; target: `SWIGGY-IGW`.
6. Save changes.
7. Open **Subnet associations → Edit subnet associations**.
8. Select `WEB-SUBNET` and save.

### Step 5: Launch a web EC2 instance
1. EC2 → **Launch instance**; choose a supported Ubuntu AMI and an appropriate lab instance type.
2. Under Network settings, choose VPC `SWIGGY` and subnet `WEB-SUBNET`.
3. Enable auto-assign public IP, if needed.
4. Create a security group allowing inbound TCP 80 from your intended test sources. For a temporary public lab, `0.0.0.0/0` on port 80 is common; do not expose SSH broadly. Restrict SSH to your own IP or use Systems Manager.
5. Add the User Data script below.
6. Launch and wait for instance status checks to pass.

### User Data for Nginx and sample website

```bash
#!/bin/bash
set -euxo pipefail
apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y nginx git
rm -rf /tmp/swiggy-clone
git clone --depth 1 https://github.com/karishma1521success/swiggy-clone.git /tmp/swiggy-clone
cp -a /tmp/swiggy-clone/. /var/www/html/
chown -R www-data:www-data /var/www/html
systemctl enable nginx
systemctl restart nginx
```

Notes:
- EC2 User Data normally runs as root; `sudo -i` is unnecessary.
- The sample repository must be available and contain the expected files. Check `/var/log/cloud-init-output.log` if deployment fails.
- For production, deploy a trusted, pinned revision and avoid exposing `.git` metadata from the web root.
- Do not rely on a sample public IP remaining the same after stop/start unless you use a suitable static address.

## 6. Practical Lab 2 — Create the private app subnet and NAT Gateway

### Step 1: Create the private app subnet
1. VPC → **Subnets → Create subnet**.
2. VPC: `SWIGGY`.
3. Name: `APP-SUBNET`.
4. CIDR: `10.0.2.0/24`.
5. Choose an Availability Zone and create it.
6. Do not enable auto-assign public IPv4 for this subnet.

### Step 2: Create a NAT Gateway
1. VPC → **NAT gateways → Create NAT gateway**.
2. Name: `SWIGGY-NAT`.
3. Subnet: `WEB-SUBNET` (a public subnet with a route to the IGW).
4. Connectivity type: **Public**.
5. Allocate/associate an Elastic IP.
6. Create the NAT Gateway and wait until its state is **Available**.

A public NAT Gateway must be placed in a public subnet and needs an Elastic IP. It allows private IPv4 resources to initiate outbound internet connections; it does not allow unsolicited inbound connections from the internet to the private instance. NAT Gateway hourly and data-processing charges apply.

### Step 3: Create the private route table
1. VPC → **Route tables → Create route table**.
2. Name: `SWIGGY-APP-RTB`; VPC: `SWIGGY`.
3. Add route `0.0.0.0/0 → SWIGGY-NAT`.
4. Associate the route table with `APP-SUBNET`.

### Step 4: Launch the private app EC2 instance
1. Launch an EC2 instance in VPC `SWIGGY`, subnet `APP-SUBNET`.
2. Disable auto-assign public IPv4.
3. Security group: allow only the necessary application port from the web server or (preferably in production) the ALB security group. Do not allow application ports from the whole internet.
4. Launch and verify the instance.

### Important correction: NAT is not a private inbound access method
NAT Gateway provides outbound connectivity from private IPv4 resources. It does not let you SSH from the internet into the private instance. Use Systems Manager Session Manager, a bastion host with carefully restricted access, or another approved private administration method.

### Connect to a private app instance for the classroom demo
If using a temporary bastion/web instance, keep the private key secure. Do not paste a production private key into shell history, public repositories, or logs. If copying a lab key to a bastion is unavoidable, restrict file permissions (`chmod 400`) and delete it after the lab. Prefer Session Manager or SSH agent forwarding with appropriate controls.

From the web/bastion server, a typical SSH command is:

```bash
ssh -i /path/to/lab-key.pem ubuntu@10.0.2.X
```

Use the private IP of the app instance and a valid key/username. The app instance security group must allow TCP 22 from the bastion/web server's security group or private IP as designed.

## 7. Route tables, Security Groups, and NACLs

### Route table
Routes traffic based on destination CIDR and target. Example:
- `10.0.0.0/16 → local` (created automatically for the VPC).
- `0.0.0.0/0 → igw-...` (public IPv4 internet route).
- `0.0.0.0/0 → nat-...` (private subnet outbound IPv4 route).
- `10.20.0.0/16 → pcx-...` (VPC peering route).
- `10.30.0.0/16 → tgw-...` (Transit Gateway route).

### Security Group vs. NACL

| Feature | Security Group | Network ACL |
|---|---|---|
| Level | Resource/network interface | Subnet |
| Stateful? | Yes | No, stateless |
| Rules | Allow rules | Allow and deny rules |
| Return traffic | Automatically allowed if stateful connection is permitted | Must be permitted by the appropriate inbound/outbound rules |
| Common use | Control traffic to/from instances | Additional subnet-level network filter |

## 8. VPC Peering

**VPC Peering creates private point-to-point connectivity between two VPCs** using AWS's network. It is not transitive: if VPC A peers with B and B peers with C, A does not automatically reach C through B.

### Example
- VPC A: `10.10.0.0/16`
- VPC B: `10.20.0.0/16`

The CIDRs must not overlap for normal routed communication.

### Practical steps
1. Create two VPCs with non-overlapping CIDRs.
2. Launch test instances in each VPC. Security groups should allow only the test protocol/source CIDR or security group as appropriate.
3. VPC console → **Peering connections → Create peering connection**.
4. Select requester VPC and accepter VPC. For cross-account or cross-Region peering, provide the required account/Region details.
5. Create the connection.
6. Select it → **Actions → Accept request** from the accepter account/Region when required.
7. Wait until status is **Active**.
8. In VPC A's route table, add destination VPC B CIDR → peering connection (`pcx-...`).
9. In VPC B's route table, add destination VPC A CIDR → the same peering connection.
10. Update both sides' security groups/NACLs to allow the intended traffic.
11. Test using private IPs and a permitted port. ICMP ping works only if security rules permit ICMP; a failed ping does not by itself prove routing is broken.

### Peering use cases
- Connect a development VPC to a shared-services VPC.
- Private communication between two VPCs.
- Cross-account connectivity for a small number of VPCs.

### Peering limitations
- Not transitive.
- CIDR overlap is a problem.
- Each VPC needs appropriate routes in both directions.
- Security groups/NACLs must permit the traffic.
- For many VPCs, a Transit Gateway may be easier to manage.

## 9. Transit Gateway (TGW)

**AWS Transit Gateway is a central network hub that connects multiple VPCs and supported on-premises networks through attachments and route tables.** It simplifies routing at scale compared with maintaining many individual peerings.

### Simple analogy
VPC peering = a direct road between two campuses. Transit Gateway = a central interchange connecting many campuses.

### Use cases
- Connect many application VPCs to a shared-services VPC.
- Centralize connectivity between VPCs and on-premises networks through VPN or Direct Connect attachments.
- Hub-and-spoke network architecture.
- Centralize network inspection or routing patterns where designed.
- Reduce the operational complexity of a large VPC estate.

### High-level architecture

```text
VPC-A ----\
VPC-B -----+---- Transit Gateway ---- VPN / Direct Connect ---- On-premises
VPC-C ----/             |
                   TGW route tables
```

### Practical TGW steps (lab outline)
1. Open VPC console → **Transit Gateways → Create transit gateway**.
2. Give it a name and review ASN, DNS support, default route-table association/propagation, and other options for the lab.
3. Wait for the Transit Gateway to become available.
4. Create a **Transit Gateway VPC attachment** for each VPC. Select the VPC and subnets in the required Availability Zones.
5. Wait until attachments are available.
6. In each VPC's relevant route table, add the destination remote VPC CIDR → Transit Gateway (`tgw-...`).
7. Configure Transit Gateway route-table associations and route propagation, or add static routes as your design requires.
8. Confirm the TGW route table has routes to the intended destination attachments.
9. Update security groups and NACLs for the required flows.
10. Test between private IPs and inspect VPC Flow Logs if it fails.

**Important:** Creating a Transit Gateway alone does not establish connectivity. You need attachments, VPC routes, Transit Gateway routes/associations, and security rules. TGW and attachment/traffic usage may incur charges; delete lab attachments and TGW when finished.

## 10. VPC Endpoints

**A VPC Endpoint provides private connectivity from a VPC to supported AWS services or endpoint services without requiring that traffic to use a public internet gateway or NAT Gateway.** The exact behavior depends on the endpoint type and service.

### Endpoint types

| Type | What it does | Example/use case |
|---|---|---|
| **Gateway endpoint** | Adds routes for supported services to selected route tables | Private access to S3 or DynamoDB from private subnets without NAT for that service traffic |
| **Interface endpoint** | Creates endpoint network interfaces (ENIs) with private IPs, powered by AWS PrivateLink | Private access to services such as Systems Manager, CloudWatch Logs, ECR APIs, Secrets Manager, and many others, subject to service support |
| **Gateway Load Balancer endpoint** | Connects VPC traffic to supported virtual network appliances via Gateway Load Balancer | Centralized inspection/firewall architectures |

### Why use VPC endpoints?
- Keep supported service traffic on private AWS connectivity rather than routing through a public internet path.
- Reduce NAT Gateway dependency and possibly NAT processing costs for traffic that can use an endpoint.
- Improve security posture by limiting public egress paths.
- Allow private-subnet workloads to reach supported AWS APIs even when they have no public IP or internet route, provided DNS, endpoint, routing, and policy configuration is correct.

### Gateway endpoint vs. Interface endpoint

| Gateway endpoint | Interface endpoint |
|---|---|
| Supported for S3 and DynamoDB | Supports many AWS services through PrivateLink |
| Uses route-table entries | Uses endpoint ENIs and private DNS options |
| No hourly endpoint charge for gateway endpoint itself under standard pricing, but data-transfer/service charges may apply | Usually has hourly and data-processing charges; check current pricing |
| Select route tables | Select VPC/subnets and configure security group/DNS as needed |

### Practical Lab A — Create an S3 Gateway Endpoint

Prerequisites: VPC exists, private subnet and route table exist, and the EC2 instance has suitable IAM permissions for the S3 operation. The endpoint provides network connectivity; IAM and bucket policies still control authorization.

1. VPC console → **Endpoints → Create endpoint**.
2. Name: `SWIGGY-S3-ENDPOINT`.
3. Service category/type: AWS services; find the S3 service for the current Region and select its **Gateway** endpoint option.
4. VPC: `SWIGGY`.
5. Route tables: select `SWIGGY-APP-RTB` (and any other route tables that need private S3 access).
6. Review endpoint policy. Use full access only for a controlled lab; production should restrict access to required buckets/actions where feasible.
7. Create endpoint and wait until it is available.
8. From an app instance with S3 permissions, test:

```bash
aws sts get-caller-identity
aws s3 ls s3://YOUR-BUCKET/
```

9. Verify the route table has the S3 prefix-list route associated with the endpoint. Confirm IAM role and bucket policy permit the operation.

The gateway endpoint does not grant S3 permissions by itself; it supplies a private network path. A NAT route can remain for other destinations, but S3 traffic matching the endpoint route uses the endpoint.

### Practical Lab B — Create an interface endpoint (example: Systems Manager)

1. VPC console → **Endpoints → Create endpoint**.
2. Choose the Systems Manager service endpoint for your Region (service names vary by service/Region; confirm the console's service list).
3. Select VPC `SWIGGY`.
4. Select private subnets in the required Availability Zones.
5. Enable private DNS if appropriate and supported for the selected service.
6. Attach an endpoint security group allowing inbound TCP 443 from the private instance security group.
7. Review endpoint policy and create it.
8. Ensure the instance has the appropriate Systems Manager instance permissions/role and the required SSM endpoints for your chosen operation, then test Session Manager.

For Systems Manager in private subnets, a working design may require multiple endpoints (for example, `ssm`, `ssmmessages`, and, depending on Region/agent/configuration, `ec2messages`), plus any required S3 or CloudWatch endpoints. Check current Systems Manager private connectivity requirements for your Region.

## 11. VPC Flow Logs

**VPC Flow Logs capture metadata about IP traffic to and from supported network interfaces, subnets, or VPCs.** They do not capture packet payload or the full application conversation.

Think of them as a CCTV camera for network traffic: they show traffic metadata, not message content.

### Where can Flow Logs be enabled?
- VPC level
- Subnet level
- Elastic Network Interface (ENI) level

### Destinations
- **CloudWatch Logs:** monitoring and investigation.
- **Amazon S3:** longer-term storage and analytics.
- **Amazon Data Firehose:** delivery to supported destinations for streaming/analytics workflows.

### Why use Flow Logs?
- Investigate traffic that is accepted or rejected.
- Troubleshoot connectivity between EC2 and RDS.
- Identify unexpected ports or network traffic patterns.
- Support security monitoring, audit evidence, and traffic analysis.
- Investigate potential scanning or repeated rejected connection attempts.
- Analyze traffic patterns that may contribute to unnecessary data-transfer/NAT costs.

Flow Logs provide useful evidence, but they do not by themselves prove an attack or identify the user behind an IP. Combine them with other logs and security tools.

### What Flow Logs do not capture
- Packet payload or application message content.
- Full DNS query content as a packet capture would.
- All traffic in every situation: some AWS platform traffic, including traffic to certain link-local addresses, may not be captured. Check current AWS Flow Logs documentation for exclusions and limitations.

### Practical: Create VPC Flow Logs to CloudWatch Logs

1. VPC console → **Your VPCs** → select `SWIGGY`.
2. Choose **Actions → Create flow log** (console labels may vary).
3. Filter: **All** to capture accepted and rejected traffic.
4. Destination: **Send to CloudWatch Logs**.
5. Select/create a CloudWatch log group.
6. Select or create an IAM role that allows the Flow Logs service to publish to that log group. Use the role wizard where available and review its trust/policy.
7. Choose the default log format initially, or a custom format if your lab requires additional fields.
8. Create the flow log and generate test traffic.
9. Open CloudWatch Logs Insights/log streams and inspect `ACCEPT` and `REJECT` records.

For S3 destination, choose S3 and configure the required bucket permissions and destination settings. Flow Logs are not instantaneous packet captures; allow time for records to be delivered.

## 12. Troubleshooting checklist

If EC2 cannot reach another resource, check in this order:
1. Is the source/destination IP correct?
2. Do the route tables contain forward and return routes?
3. Are VPC peering/TGW attachments active and routes installed?
4. Do Security Groups allow the protocol/port and correct source?
5. Do NACL inbound and outbound rules allow the traffic and return path?
6. Does the target service listen on the expected port?
7. Is DNS configured and resolving to the intended address?
8. For private AWS service access, is the correct endpoint created, associated, and permitted by endpoint/IAM/resource policies?
9. Check VPC Flow Logs for ACCEPT/REJECT evidence, remembering that logs are not a packet capture.

### Common symptoms

| Symptom | Possible cause |
|---|---|
| Public EC2 cannot reach internet | No IGW route, no public IPv4/EIP, wrong subnet route table, SG/NACL restriction |
| Private EC2 cannot download updates | NAT missing/unavailable, private route table incorrect, DNS/egress rule issue |
| Private EC2 cannot access S3 privately | S3 gateway endpoint missing from route table, IAM/bucket/endpoint policy denies access |
| Peered instances cannot communicate | Peering not active, missing routes in one direction, overlapping CIDRs, SG/NACL blocks |
| TGW-connected VPCs cannot communicate | Missing attachments/routes/associations/propagation, SG/NACL blocks, overlapping routes |
| Flow Logs show REJECT | A network ACL or security control rejected traffic; inspect direction, port, and relevant rules |

## 13. Security and production best practices

- Use private subnets for application/database tiers unless public access is explicitly required.
- Do not expose SSH/RDP to `0.0.0.0/0`; use a restricted source or Systems Manager.
- Use least-privilege security group rules and IAM policies.
- Prefer VPC endpoints for supported private AWS service access when they fit the design.
- Use multiple Availability Zones for production designs where appropriate.
- Use managed, appropriately sized NAT Gateways and account for their costs; for high availability, plan NAT per AZ and routing deliberately.
- Avoid overlapping CIDRs across VPCs/on-premises networks.
- For many VPCs, consider Transit Gateway instead of many independent peerings.
- Enable Flow Logs for security investigations and network troubleshooting, with appropriate retention and access controls.
- Review costs for NAT Gateway, Transit Gateway, interface endpoints, Flow Logs, and data transfer.

## 14. Interview questions and answers

**Q1. What is a VPC?**  A logically isolated network in AWS for launching resources.

**Q2. What makes a subnet public?**  Its route table has a route to an Internet Gateway; a resource also needs appropriate public addressing and security rules for direct internet communication.

**Q3. What is CIDR?**  A notation for an IP address range using a prefix length, such as `10.0.0.0/16`.

**Q4. IGW vs. NAT Gateway?**  An IGW provides internet routing for suitable public resources. A NAT Gateway allows private IPv4 resources to initiate outbound connections without accepting unsolicited inbound internet connections.

**Q5. Security Group vs. NACL?**  Security Groups are stateful resource-level controls; NACLs are stateless subnet-level controls.

**Q6. What is VPC peering?**  Private point-to-point routing between two VPCs; it is not transitive.

**Q7. What is Transit Gateway?**  A central hub for routing between multiple VPCs and supported network attachments.

**Q8. What is a VPC endpoint?**  A private connectivity option for supported AWS services or endpoint services, avoiding the need for a public internet path for that service traffic.

**Q9. Gateway vs. interface endpoint?**  Gateway endpoints use route tables for S3/DynamoDB; interface endpoints create private IP network interfaces using PrivateLink for many services.

**Q10. What are VPC Flow Logs?**  Records of network traffic metadata, including accepted/rejected traffic where captured, not packet payload.

**Q11. Why use Flow Logs?**  Troubleshooting, security investigation, traffic analysis, and audit support.

**Q12. Does a VPC endpoint automatically grant access to S3?**  No. IAM, bucket policy, and endpoint policy must permit the action.

**Q13. Does NAT Gateway allow inbound SSH to private EC2?**  No. Use Systems Manager or an approved bastion/private access method.

**Q14. Why are overlapping VPC CIDRs a problem?**  Routing becomes ambiguous and direct routed connectivity may not work as intended.

**Q15. Why use Transit Gateway rather than peering?**  TGW centralizes routing and can simplify large multi-VPC networks; it has its own design and cost considerations.

## 15. One-line revision map

- **VPC:** Isolated AWS network.
- **Subnet:** IP range in one Availability Zone.
- **CIDR:** IP range notation.
- **IGW:** Internet route target for public resources.
- **NAT Gateway:** Outbound internet access for private IPv4 resources.
- **Route table:** Chooses next hop based on destination.
- **Security Group:** Stateful resource-level firewall.
- **NACL:** Stateless subnet-level filter.
- **Peering:** Direct connection between two VPCs; not transitive.
- **Transit Gateway:** Central hub for many network attachments.
- **VPC Endpoint:** Private access to supported services.
- **Flow Logs:** Network traffic metadata, not payload.

## 16. Lab cleanup checklist

- [ ] Terminate EC2 lab instances when finished.
- [ ] Delete NAT Gateway when no longer needed; release its Elastic IP if unused.
- [ ] Delete unused VPC endpoints, especially interface endpoints that can incur hourly charges.
- [ ] Remove peering connections and Transit Gateway attachments; delete a lab Transit Gateway when safe.
- [ ] Delete Flow Logs and log groups if no longer needed, or set appropriate retention.
- [ ] Remove temporary security group rules and IAM permissions.
- [ ] Check for other billable resources before ending the lab.
