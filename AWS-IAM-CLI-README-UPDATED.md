# AWS IAM, AWS CLI, MFA & OIDC — Beginner-to-Architect Notes

## 1. IAM introduction
**AWS Identity and Access Management (IAM)** controls who can access AWS and what actions they can perform.

- **Authentication:** proves who you are.
- **Authorization:** decides what you can do.
- **Policy:** JSON document describing permissions.
- **Least privilege:** grant only permissions needed for a task.

Memory line: Authentication = who are you? Authorization = what can you do?

## 2. Root user vs IAM
Do not use the AWS account root user for routine work. Protect it with MFA and do not create root access keys for everyday automation. For workforce access, prefer IAM Identity Center/federation where practical. Use IAM roles and temporary credentials for AWS workloads; IAM users may still be needed for specific legacy/programmatic use cases.

## 3. IAM users and groups

### Create a test user
1. IAM → Users → Create user.
2. Enter a name, e.g. `s3-lab-user`.
3. Enable console access only if the person needs it.
4. Attach a narrowly scoped policy or add the user to a group.
5. Create the user and test with the sign-in URL in a separate private browser window.

For teams, prefer federated access/IAM Identity Center over long-lived IAM users where possible.

### Groups
Groups simplify permission management for multiple users.
- Groups can have policies attached; users can belong to multiple groups.
- Groups cannot be nested.
- The default quota is up to 10 groups per user; verify current account quota.
- Effective permissions may come from user policies and all group policies.
- An explicit **Deny** overrides an Allow.

Create: IAM → User groups → Create group → name it (e.g. `DevOps-ReadOnly`) → add users → attach appropriate policies → create.

If a user forgets a console password, an authorized administrator can reset console access through the user's settings. Never ask users to share passwords.

## 4. IAM roles
A role is an identity that a trusted principal can **assume**. A role does not have a permanent password or long-lived access key of its own; assuming it provides temporary credentials.

Use cases:
- EC2 → S3
- Lambda → DynamoDB
- GitHub Actions → AWS using OIDC
- A user or another AWS account assuming a role

Roles are not only for service-to-service access. They can be assumed by AWS services, users, and federated identities if the trust policy permits it.

### Hands-on: EC2 role to S3
1. IAM → Roles → Create role.
2. Trusted entity: AWS service → EC2.
3. Attach least-privilege permissions for the required bucket/actions. Avoid `AmazonS3FullAccess` in production unless justified.
4. Name the role, e.g. `EC2-S3-App-Role`.
5. EC2 → Instances → select instance → Actions → Security → Modify IAM role → attach role.
6. On the EC2 instance, install AWS CLI if needed and run:
```bash
aws sts get-caller-identity
aws s3 ls s3://YOUR-BUCKET/
```
Do not place long-lived IAM user keys on EC2 when an instance role can be used.

### Example least-privilege S3 policy
Replace `YOUR-BUCKET` with the exact bucket name. Remove any actions the workload does not need.
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListOneBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::YOUR-BUCKET"
    },
    {
      "Sid": "ReadWriteObjects",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::YOUR-BUCKET/*"
    }
  ]
}
```
` s3:ListBucket` uses the bucket ARN; object actions use the object ARN ending in `/*`.

## 5. IAM policy types

| Policy type | Purpose |
|---|---|
| **Identity-based policy** | Attached to a user, group, or role; defines actions that identity may perform |
| **Resource-based policy** | Attached to a resource such as an S3 bucket; defines who can access it |
| **Permissions boundary** | Maximum permissions an IAM user/role can receive from identity policies; does not grant permissions itself |
| **Service control policy (SCP)** | AWS Organizations guardrail limiting permissions available in member accounts; does not grant permissions |
| **Session policy** | Further restricts permissions for an assumed-role/federated session |
| **ACL** | Legacy/resource-level access mechanism for supported services; prefer modern policy controls where possible |
| **Role trust policy** | Defines who can assume a role; separate from the role's permissions policy |
| **VPC endpoint policy** | Controls requests through a VPC endpoint; does not replace identity/resource policies |

### Managed vs inline policies
- **AWS managed policy:** maintained by AWS; convenient but may be broader than needed.
- **Customer managed policy:** created in your account, reusable and easier to govern.
- **Inline policy:** embedded in one user, group, or role; tightly coupled but harder to reuse at scale.

Prefer customer-managed policies for reusable custom permissions. Avoid broad `s3:*` on `*` unless there is a justified administrative requirement.

### Policy evaluation essentials
- Explicit Deny overrides Allow.
- A policy Allow is effective only if no applicable explicit Deny or higher-level restriction blocks it.
- Boundaries and SCPs set maximum permissions; they do not grant access.
- Trust policy controls role assumption; permissions policies control what the role can do after assumption.

## 6. Permissions boundaries
A permissions boundary sets the maximum permissions an IAM user or role can receive from identity-based policies. It does not grant permissions itself.

Example: a developer has an identity policy allowing several services, but a boundary excludes IAM administration. The developer cannot exceed the boundary even if a broader identity policy is attached.

Use boundaries for delegated administration and guardrails. They are not a universal “deny this service” switch; design and test the maximum permission set carefully.

## 7. MFA (multi-factor authentication)
MFA adds an additional identity-verification factor to sign-in. Supported options can include authenticator apps, security keys/passkeys, or hardware devices, depending on the identity setup.

Setup outline:
1. Open IAM → Users → select user → Security credentials.
2. Assign/register an MFA device and complete the verification steps.
3. Test sign-in and protect recovery methods.

Enable MFA for the root user and privileged human identities. Prefer phishing-resistant security keys/passkeys where supported. Never share MFA codes. MFA complements—not replaces—least privilege and temporary credentials.

## 8. Access keys and safe handling
An access key consists of an **Access Key ID** and **Secret Access Key**. These are programmatic credentials, not permissions; IAM policies determine allowed actions.

- An IAM user can have up to **two access keys** at once.
- The secret key is shown only at creation. If lost, create a replacement and rotate.
- If a key may be compromised, deactivate it immediately, investigate usage, rotate dependent applications, then delete it when safe.
- Never commit keys to Git, include them in AMIs, or paste them into public chats.
- Prefer roles and temporary credentials for EC2, Lambda, containers, and CI/CD.
- Never create root access keys for routine work.

Create a CLI key only if truly needed: IAM → Users → user → Security credentials → Create access key → select use case and follow the warnings. Store it securely.

`aws configure` writes credentials to local CLI config files. Do not configure long-lived IAM user keys on EC2 if an instance role can do the job.

## 9. Credential report
IAM → Credential report → Generate/download report. It summarizes IAM users and credential status such as password, access-key, and MFA details. AWS documents the generated report as available for up to four hours. Treat the report as sensitive security data.

## 10. AWS CLI introduction and install
AWS CLI lets you operate AWS services from a terminal.

Ubuntu/Debian example for AWS CLI v2:
```bash
sudo apt update
sudo apt install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```
Official installation guide: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

Configure a profile only when appropriate:
```bash
aws configure --profile lab
aws sts get-caller-identity
aws configure list
```
Use a valid Region for your resources (e.g. `ap-south-1`) and an output format such as `json` or `table`. Prefer IAM Identity Center or temporary role credentials where available. Do not paste real keys into notes.

## 11. Useful S3 CLI commands
Replace example bucket names and paths.
```bash
# List buckets
aws s3 ls

# List objects
aws s3 ls s3://YOUR-BUCKET/

# Create a bucket (globally unique name)
aws s3 mb s3://YOUR-UNIQUE-BUCKET --region ap-south-1

# Upload / download
aws s3 cp ./report.txt s3://YOUR-BUCKET/report.txt
aws s3 cp s3://YOUR-BUCKET/report.txt ./report.txt

# Upload a directory
aws s3 cp ./website/ s3://YOUR-BUCKET/website/ --recursive

# Sync directories or buckets (review direction and --delete carefully)
aws s3 sync ./website/ s3://YOUR-BUCKET/website/
aws s3 sync s3://SOURCE-BUCKET/ s3://DESTINATION-BUCKET/
```
Remove an empty bucket:
```bash
aws s3 rb s3://YOUR-EMPTY-BUCKET
```
Destructive options such as recursive deletion or `--delete` require careful review.

## 12. Useful EC2 CLI commands
AMI IDs, instance types, network settings, and Regions must match your environment. An AMI ID may not exist in another Region.
```bash
aws ec2 describe-instances --region ap-south-1
aws ec2 start-instances --instance-ids i-0123456789abcdef0 --region ap-south-1
aws ec2 stop-instances --instance-ids i-0123456789abcdef0 --region ap-south-1
# Destructive: permanently terminates an instance
aws ec2 terminate-instances --instance-ids i-0123456789abcdef0 --region ap-south-1
```
A launch command usually also needs the correct subnet, security groups, key/access method, IAM instance profile, and other environment-specific settings.

## 13. OIDC introduction
**OpenID Connect (OIDC)** enables federation with a trusted identity provider. AWS can validate the provider's token using an IAM OIDC provider and a role trust policy. A workload can then obtain temporary AWS credentials without storing long-lived AWS access keys.

Example: GitHub Actions → OIDC token → IAM role → temporary credentials → deploy to AWS.

High-level setup:
1. Register/configure the OIDC identity provider in IAM.
2. Create a role with a trust policy for that provider.
3. Restrict the expected audience (`aud`) and subject (`sub`) claims to the intended repository, branch, or environment.
4. Attach only the permissions needed for the deployment.
5. Configure the workflow to request an OIDC token and assume the role.
6. Test both authorized and unauthorized branches/repositories.

Security: avoid broad trust policies, scope the provider/audience/subject claims, use least privilege, and review CloudTrail. OIDC handles federation/authentication; the role policy still determines authorization.

## 14. Revoke sessions and remove access
Existing temporary role sessions may remain valid until expiry unless explicitly revoked or constrained. IAM role session revocation mechanisms can deny sessions issued before a specified time in supported cases; use carefully because workloads may be interrupted.

For compromised access keys: deactivate immediately, investigate logs, rotate dependent systems, then delete when safe. For offboarding, review console access, access keys, roles, active sessions, and resource policies.

## 15. Hands-on checklist
- [ ] Create a test user or use a federated test identity.
- [ ] Create a group with read-only permissions and test access.
- [ ] Attach a least-privilege EC2 role for one S3 bucket.
- [ ] Run `aws sts get-caller-identity` on EC2 and test S3 access.
- [ ] Create/review a customer-managed JSON policy.
- [ ] Demonstrate how a permissions boundary limits permissions.
- [ ] Enable MFA for privileged identities.
- [ ] Generate and review a credential report.
- [ ] Study OIDC trust-policy conditions in a sandbox.

## 16. Interview questions
1. **Authentication vs authorization?** Authentication proves identity; authorization decides allowed actions.
2. **How should EC2 access S3?** Use an instance role/profile with least-privilege permissions.
3. **Can groups be nested?** No.
4. **How many groups can a user join?** Default quota is 10; verify current account quota.
5. **IAM user vs role?** Users can have long-term credentials; roles are assumed and provide temporary credentials.
6. **Can a user use multiple roles?** A user can be authorized to assume multiple roles; roles are not attached to a user in the same way an instance profile is attached to EC2.
7. **How do you cap permissions?** A permissions boundary can set the maximum identity-based permissions.
8. **How many access keys can an IAM user have?** Two at a time.
9. **What if a key is compromised?** Deactivate immediately, investigate, rotate dependent systems, and delete when safe.
10. **How can a customer account access a bucket?** Use a carefully scoped bucket policy and/or cross-account role with the necessary identity and resource permissions; an inline policy alone is not a universal solution.
11. **What format are IAM policies?** JSON.
12. **What does a trust policy do?** Defines who can assume a role.
13. **What is OIDC used for?** Federated access, such as CI/CD obtaining temporary AWS credentials without long-lived keys.
14. **What is MFA?** An extra factor to verify identity.
15. **What is a credential report?** An account-level report of IAM users' credential status for audits.

## 17. Quick memory lines
- Authentication = prove identity; authorization = allowed actions.
- Policy = JSON permission rules.
- Trust policy = who can assume a role.
- Role = temporary credentials for a trusted principal.
- Boundary/SCP = permission guardrail, not an Allow by itself.
- Explicit Deny wins.
- MFA = extra sign-in protection.
- Access key = programmatic credential, not permission.
- EC2 → S3 = role, not embedded long-lived keys.
- OIDC = trusted external identity → role → temporary AWS credentials.
- Least privilege = only what is needed.

## Official AWS references
- IAM best practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- Policy types: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html
- Permissions boundaries: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html
- MFA: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa.html
- Access keys: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html
- Credential report: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_getting-report.html
- EC2 roles: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html
- OIDC federation: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_oidc.html
- AWS CLI v2: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html


---

# 18. Practical Console Walkthroughs from the Original Session

## A. Create an IAM user with console access
1. **IAM → Users → Create user**.
2. Enter a username such as `s3-lab-user`.
3. Select **Provide user access to the AWS Management Console** if console login is needed.
4. Choose the IAM-user option if prompted, configure the password flow, then select **Next**.
5. For a lab, attach a narrow policy such as `AmazonS3ReadOnlyAccess`. Use `AmazonS3FullAccess` only when full access is specifically required in a disposable lab.
6. Select **Next → Create user**; save the sign-in URL and deliver initial credentials securely.
7. Test sign-in in a separate private/incognito browser. Test both allowed and denied actions.

Prefer IAM Identity Center/federation and group-based permissions for human teams where practical.

## B. Create users and add them to a group
1. Create test users without service permissions.
2. Go to **IAM → User groups → Create group**.
3. Enter a name such as `DevOps`.
4. Select users, attach an appropriate policy, and create the group.
5. Sign in as a test user and verify the group permissions.

Groups cannot be nested. The default quota is 10 groups per user; check current account quotas. Direct user and group permissions combine, but explicit **Deny** overrides Allow.

## C. Reset a forgotten console password
1. **IAM → Users → select user**.
2. Open **Security credentials** or the console-access section.
3. Choose **Manage console access** or the available password-reset action.
4. Set a new password and require a change at next sign-in if appropriate.
5. Share it through an approved secure channel, never a public chat/document.

## D. Create and attach an EC2-to-S3 role
1. **IAM → Roles → Create role → AWS service → EC2**.
2. Attach least-privilege S3 permissions scoped to the required bucket/actions. Use `AmazonS3FullAccess` only for a temporary demo that intentionally needs full access.
3. Name it `EC2-S3-App-Role`, review the trust policy (EC2 service), and create it.
4. **EC2 → Instances → select instance → Actions → Security → Modify IAM role**.
5. Select the role and update.
6. On the instance, install AWS CLI if needed and verify:
```bash
aws sts get-caller-identity
aws s3 ls s3://YOUR-BUCKET/
```
7. For `AccessDenied`, check role permissions, bucket policy, KMS permissions, bucket name, and Region.

Do not store long-lived IAM user keys on EC2 when an instance role can provide temporary credentials.

### Upload an Nginx access log
Check the actual log path and permissions first; paths vary by OS/package.
```bash
sudo ls -l /var/log/nginx/access.log
aws s3 cp /var/log/nginx/access.log s3://YOUR-BUCKET/logs/access.log
```
The role needs `s3:PutObject` on that destination. Review logs for secrets/sensitive data before upload.

## E. Create an inline policy
1. **IAM → Users → select user → Add permissions → Create inline policy** (or select a role/group).
2. Choose S3 and only the required actions.
3. Use `arn:aws:s3:::YOUR-BUCKET` for bucket actions such as `s3:ListBucket`.
4. Use `arn:aws:s3:::YOUR-BUCKET/*` for object actions such as `s3:GetObject` and `s3:PutObject`.
5. Review JSON, name/save, then test allowed and denied actions.

Inline policies are tied to one identity. Prefer customer-managed policies for reusable permissions. Avoid `s3:*` on `*` unless broad access is genuinely required.

## F. Install AWS CLI v2

### Ubuntu/Debian Linux
```bash
sudo apt update
sudo apt install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```
This URL is for 64-bit x86 Linux; use the official installer for other architectures/OS:
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

### Windows
Use the official AWS CLI v2 MSI installer and follow the current instructions:
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

Open a new terminal and verify:
```powershell
aws --version
```

## G. Configure AWS CLI
Only configure long-lived IAM user keys when truly needed. Prefer IAM Identity Center or temporary role credentials.
```bash
aws configure --profile lab
aws sts get-caller-identity --profile lab
aws configure list
```
Enter Access Key ID, Secret Access Key, a valid Region (for example `ap-south-1` for Mumbai resources), and output format (`json` or `table`). Never paste real keys into notes/screenshots or configure user keys on EC2 when a role is available.

## H. Create and rotate an access key
1. **IAM → Users → select user → Security credentials → Create access key**.
2. Follow the use-case prompts and store the secret securely; it is displayed only once.
3. For planned rotation, create a replacement if the two-key limit permits, update/test consumers, deactivate the old key, then delete it when safe.
4. If compromise is suspected, deactivate the key immediately, investigate usage, rotate dependent systems, then delete when safe.

An IAM user can have at most two access keys at a time. Do not create root-user access keys for routine work.

## I. S3 CLI practice
Replace example bucket names with buckets you own.
```bash
aws s3 ls
aws s3 ls s3://YOUR-BUCKET/
aws s3 mb s3://YOUR-UNIQUE-BUCKET --region ap-south-1
aws s3 cp ./file.txt s3://YOUR-BUCKET/file.txt
aws s3 cp s3://YOUR-BUCKET/file.txt .
aws s3 cp ./my-folder/ s3://YOUR-BUCKET/my-folder/ --recursive
aws s3 sync s3://SOURCE-BUCKET/ s3://DESTINATION-BUCKET/
aws s3 rb s3://YOUR-EMPTY-BUCKET
```
`aws s3 rb` without recursive deletion requires an empty bucket. Sync is directional; review source/destination and any `--delete` option carefully.

## J. EC2 CLI practice
These commands control real infrastructure. Replace sample IDs/Regions.
```bash
aws ec2 describe-instances --region ap-south-1
aws ec2 stop-instances --instance-ids i-0123456789abcdef0 --region ap-south-1
aws ec2 start-instances --instance-ids i-0123456789abcdef0 --region ap-south-1
# Destructive: terminates the instance
aws ec2 terminate-instances --instance-ids i-0123456789abcdef0 --region ap-south-1
```
AMI IDs are Region-specific. Launching also requires correct networking, security groups, access method/key pair, and instance profile.

## K. Set a custom password policy
1. **IAM → Account settings → Password policy → Edit**.
2. Set requirements according to current console options and your organization's security standard.
3. Save and test with a lab user.

Do not rely on an old screenshot for exact length limits; use the current console options.

## L. Generate an IAM credential report
1. **IAM → Credential report → Generate report**.
2. Download it when ready.
3. Review password/console status, MFA status, and access-key age/status.
4. Treat the report as sensitive. AWS documents it as available for up to four hours after generation.

## M. Permissions boundary practical
1. Create a customer-managed identity policy allowing a small set of actions.
2. Create a boundary policy defining the maximum permissions.
3. Attach the boundary to a lab user/role.
4. Attach an identity policy that allows an action outside the boundary.
5. Test that the action is blocked, then clean up.

A boundary does not grant permissions by itself and is not simply a universal deny-list.

## N. Revoke or restrict role sessions
Temporary credentials may remain valid until expiry unless a supported session-revocation or policy mechanism blocks them. Review current IAM role-session revocation guidance and test in a sandbox because older sessions may be interrupted. For a compromised access key, deactivate it immediately.

## O. OIDC practical outline for CI/CD
1. Configure the CI/CD platform's OIDC identity provider in IAM.
2. Create a role trusting that provider.
3. Restrict trust conditions using expected `aud` and `sub` claims (repository, branch, or environment).
4. Attach only required deployment permissions.
5. Configure the pipeline to request an OIDC token and assume the role.
6. Run `aws sts get-caller-identity` in the pipeline and test that unauthorized repositories/branches cannot assume the role.

## Lab cleanup
Remove disposable users, groups, policies, and roles when finished. Deactivate/delete unused lab keys. Remove lab infrastructure only after checking that its data is no longer needed.
