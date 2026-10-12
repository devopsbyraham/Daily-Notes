# AWS EBS — Theory, Hands-on & Interview Notes

## 1. EBS introduction
Amazon Elastic Block Store (EBS) is persistent **block storage** for EC2. Think of EC2 as the computer and EBS as its hard disk. EBS data usually persists after an instance stops; deletion behavior depends on volume lifecycle and the instance's DeleteOnTermination setting.

- **Root/boot volume:** usually contains the operating system and boot files.
- **Data/non-root volume:** extra disk for application data, logs, uploads, or databases.
- **Volume:** live disk attached to an instance.
- **Snapshot:** point-in-time backup of a volume.
- **AMI:** image/template used to launch an EC2 instance, including boot configuration and references to backing snapshots for EBS volumes.
- EBS is not EFS: EBS is block storage; EFS is a shared network file system.

Volumes are created in a specific Availability Zone (AZ) and normally attach to instances in that same AZ. Snapshots are Region-specific and can be copied to another Region.

## 2. EBS volume types

| Type | Media | Best fit | Memory line |
|---|---|---|---|
| gp3 | SSD, general purpose | Most boot, web and app servers | Balanced default |
| gp2 | SSD, general purpose | Existing workloads; consider gp3 for new ones | Older general-purpose SSD |
| io2 / io2 Block Express | SSD, provisioned IOPS | Critical, high-IOPS databases | High-performance database |
| io1 | SSD, provisioned IOPS | Existing high-performance workloads | Older provisioned-IOPS |
| st1 | HDD, throughput optimized | Large sequential I/O, logs, data processing | High-throughput HDD |
| sc1 | HDD, cold | Infrequently accessed throughput-oriented data | Low-cost cold HDD |

HDD volumes (st1/sc1) cannot be boot/root volumes. Choose based on IOPS, throughput, latency, capacity, and price—not just size. Size limits vary by type and AWS quotas; many common types support up to 64 TiB, so do not memorize “8 GB–16 TB” as a universal limit. Check the current AWS volume-type documentation.

## 3. Modify an existing volume
Console:
1. EC2 → Elastic Block Store → Volumes.
2. Select the volume → Actions → Modify volume.
3. Change size, type, or supported performance settings → Modify.
4. Wait for the modification to complete.
5. If size increased, expand the partition/filesystem inside the OS.

Linux checks:
```bash
lsblk
df -hT
```
Device names below are examples only. Identify the correct disk/partition before making changes. For example, after confirming the actual partition:
```bash
sudo growpart /dev/nvme0n1 1
```
For ext4, a typical command is `sudo resize2fs /dev/nvme0n1p1`. For XFS, grow the mounted filesystem, e.g. `sudo xfs_growfs /`. Use only the command matching your actual filesystem and mount point. Shrinking an EBS volume in place is not supported; create a smaller volume and migrate data if needed.

## 4. Create and attach a data volume
1. EC2 → Elastic Block Store → Volumes → Create volume.
2. Choose type (gp3 is a common default), size, and the **same AZ** as the target instance.
3. Add tags, e.g. `Name = app-data`, then create.
4. Select the volume → Actions → Attach volume → choose instance → Attach.

`/dev/xvdb` is a traditional example, not a guaranteed Linux device name. Nitro instances commonly show EBS disks as `/dev/nvme...`. Identify the correct device using `lsblk`.

For a brand-new, empty disk only (formatting erases data):
```bash
lsblk
sudo file -s /dev/nvme1n1
sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir -p /data
sudo mount /dev/nvme1n1 /data
df -hT /data
sudo blkid /dev/nvme1n1
```
Use the filesystem UUID in `/etc/fstab` for persistent mounting, and test with `sudo mount -a`. Never run `mkfs` on a volume that might contain data.

## 5. EBS snapshots
A snapshot is a point-in-time backup of an EBS volume. Snapshots are incremental at the storage level: after the first snapshot, AWS generally stores changed blocks, while each snapshot is still a complete logical restore point. Actual billed storage depends on changed blocks and AWS billing rules.

Create: EC2 → Volumes → select volume → Actions → Create snapshot → add description/tags → Create snapshot.

Restore: EC2 → Snapshots → select snapshot → Actions → Create volume from snapshot → choose AZ → create → attach volume to an instance in that AZ → mount and validate.

A snapshot cannot be attached directly to EC2. Create a volume from it first. For databases, coordinate with the application/database when application consistency is required; a default snapshot is not automatically equivalent to a database-native backup.

### Snapshot archive
Archive is a lower-cost tier for snapshots that are rarely accessed. An archived snapshot must be restored before use; recovery can take many hours (often cited around 24–72 hours, depending on circumstances). Do not archive backups requiring fast recovery. Check current archive pricing, minimum retention, and restore rules.

## 6. Fast Snapshot Restore (FSR)
FSR initializes snapshot-backed volumes in advance in selected AZs, reducing first-read latency and making performance more predictable immediately after volume creation.

Use it for time-critical recovery or fleets launched from the same snapshot. Console outline: EC2 → Snapshots → select snapshot → Fast Snapshot Restore → enable for required AZ(s) → wait until enabled → test a restored volume. FSR costs extra per snapshot per enabled AZ; enable it only where the benefit justifies the cost. It does not replace backups.

## 7. Backups and retention
Define:
- **Frequency:** how often backups are made.
- **Retention:** how long they are kept.
- **RPO:** acceptable data loss.
- **RTO:** acceptable time to restore service.
- **Isolation:** another Region/account for disaster recovery.
- **Restore testing:** prove backups can be restored.

| Option | Use |
|---|---|
| Manual snapshot | One-off pre-change backup |
| Amazon Data Lifecycle Manager (DLM) | Automate EBS snapshot/AMI creation, retention, and cleanup |
| AWS Backup | Centralized backup governance for supported AWS services |

A backup is not a complete DR plan unless access, encryption keys, Region/account recovery, RTO, and restore testing are addressed.

## 8. Cross-Region snapshot copy
Use for disaster recovery or migration.
1. In source Region, create snapshot.
2. EC2 → Snapshots → select snapshot → Actions → Copy snapshot.
3. Select destination Region and encryption/KMS options → Copy.
4. Switch console to destination Region.
5. Create volume from the copied snapshot in the AZ where the destination instance runs.
6. Attach, mount, and validate.

**Correction to common lab errors:** A volume created in `ap-south-1` cannot attach to an instance in `us-east-1`. The snapshot must be copied to the destination Region first. A volume must be created in the same AZ as the target instance. AZ names like `us-east-1a` may map to different physical AZs across accounts; use AZ IDs if physical alignment matters.

## 9. Cross-account sharing
In the source account: EC2 → Snapshots → select snapshot → Actions → Modify permissions → add the destination account ID as a private permission.

In the destination account, find the shared snapshot in the same Region and create a volume or copy the snapshot into the destination account.

Security:
- A snapshot may contain all disk data, credentials, logs, or customer information. Share only with trusted accounts.
- Snapshots encrypted with the default AWS-managed EBS KMS key cannot be shared directly across accounts. Use a customer-managed KMS key and grant the required key permissions.
- Encrypted snapshot sharing requires appropriate KMS permissions; the receiving account may need to copy it using a key it controls.
- Never share snapshots publicly unless intentionally public and thoroughly reviewed.

## 10. Amazon Data Lifecycle Manager (DLM)
DLM automates creation, retention, and deletion of EBS snapshots and EBS-backed AMIs.

Hands-on daily backup policy:
1. Tag target volumes, e.g. `Backup = Daily`.
2. EC2 → Elastic Block Store → Lifecycle Manager → Create lifecycle policy.
3. Choose an EBS snapshot policy and target resources by tags.
4. Configure schedule (e.g. daily), retention (e.g. keep the last 7 daily backups), and optional cross-Region copy/advanced options.
5. Check execution role and KMS permissions.
6. Create/enable the policy.
7. Verify the first snapshots and confirm retention behavior.

Use separate tags/policies for production and test resources. Monitor policy failures and test restores. DLM manages resources targeted by its policies; it does not automatically manage every manually created snapshot or AMI.

## 11. AMI vs snapshot vs volume

| Item | Meaning | Purpose |
|---|---|---|
| EBS volume | Live block disk | Storage attached to EC2 |
| EBS snapshot | Point-in-time volume backup | Backup, restore, copy, migration |
| AMI | EC2 launch image/template | Launch servers with prepared OS/configuration |

Memory line: **AMI to launch servers; snapshots to back up volumes.** This is a simplification: EBS-backed AMIs rely on snapshots for their backing volumes, and DLM can automate both snapshot and AMI management. “1–10 files = AMI; 10–100 files = snapshot” is not a valid AWS rule. Choose based on whether you need to launch an instance or restore/copy a disk.

## 12. Server migration lab — corrected
### Move only a data volume from `us-east-1` to `ap-south-1`
1. Stop writes or quiesce the app if consistency requires it.
2. Create a snapshot of the source data volume.
3. Copy the snapshot to `ap-south-1`.
4. Create a volume from the copied snapshot in the destination instance's AZ.
5. Launch/select an EC2 instance in that same destination AZ.
6. Attach the volume; inspect with `lsblk`, mount, and validate data.
7. Update app configuration and test before traffic cutover.

### Re-create a full server
1. Create an AMI from the source EC2 instance, or snapshot all required volumes.
2. Copy the AMI to the destination Region (or copy the required snapshots).
3. Launch an instance from the copied AMI in the destination Region.
4. Validate/recreate networking, security groups, IAM role, secrets, DNS, monitoring, and app configuration.
5. Test, cut over traffic, and retain the source until rollback is no longer needed.

Do not detach a root volume from a running instance. Do not assume a copied data volume is bootable. Full server migration is usually safer through AMI copy/launch or a planned image-and-data migration.

## 13. AWS CLI examples
Replace sample IDs and use your actual Regions/AZs and permissions.

```bash
aws ec2 describe-volumes --region ap-south-1

aws ec2 create-snapshot \
  --region us-east-1 \
  --volume-id vol-0123456789abcdef0 \
  --description "Before application upgrade"

aws ec2 copy-snapshot \
  --region ap-south-1 \
  --source-region us-east-1 \
  --source-snapshot-id snap-0123456789abcdef0 \
  --description "DR copy in ap-south-1"

aws ec2 create-volume \
  --region ap-south-1 \
  --availability-zone ap-south-1a \
  --snapshot-id snap-0123456789abcdef0 \
  --volume-type gp3
```

The copied snapshot ID returned by `copy-snapshot` is the ID to use when creating the destination volume.

## 14. Troubleshooting

| Issue | Check |
|---|---|
| Volume cannot attach | Same AZ? Instance/volume limits? Correct instance selected? |
| Disk not visible in Linux | Run `lsblk`; NVMe name may differ from console device name |
| Disk size unchanged inside OS | Extend partition and filesystem |
| Mount fails | Confirm correct device, filesystem, UUID, and mount point |
| Snapshot copy fails | Region, IAM, KMS, quota, snapshot state |
| Encrypted share inaccessible | Customer-managed KMS key permissions and account sharing |
| First reads slow after restore | Consider volume initialization and FSR if justified |
| DLM creates no backups | Policy enabled? Tags match? Role/KMS permissions? Correct Region? |

## 15. Interview questions
1. **How is persistent data stored on EC2?** Usually EBS; instance store is another option with different persistence behavior.
2. **How do you back up EBS?** Snapshots, DLM, or AWS Backup.
3. **Are volumes and snapshots AZ/Region specific?** Volumes are AZ-specific; snapshots are Region-specific.
4. **What is the EBS min/max size?** Depends on volume type and current AWS quotas; consult current docs.
5. **Can one EC2 instance have multiple EBS volumes?** Yes, within supported limits.
6. **How do you reduce snapshot cost?** Delete unnecessary backups with lifecycle policies; archive suitable long-term snapshots.
7. **Can a snapshot attach directly to EC2?** No—create a volume first.
8. **What are the volume families?** SSD-backed general-purpose/provisioned-IOPS and HDD-backed throughput-optimized/cold.
9. **What is incremental snapshotting?** Changed blocks are generally newly stored; each snapshot remains a logical restore point.
10. **What is FSR?** Pre-initializes snapshot-backed volumes in selected AZs for predictable first-read performance, for an additional charge.
11. **What is DLM?** Automates snapshot/AMI schedules, retention, and cleanup.
12. **How do you migrate a volume across Regions?** Snapshot → copy snapshot → create volume in destination AZ → attach and validate.
13. **What matters for encrypted cross-account snapshots?** Customer-managed KMS key and permissions.
14. **Does an AMI equal a snapshot?** No; an AMI is a launch image and may reference snapshots for its EBS-backed disks.

## 16. Quick memory lines
- EBS volume = live disk.
- Snapshot = point-in-time volume backup.
- Incremental = changed blocks stored where applicable.
- AMI = template to launch EC2.
- FSR = faster, predictable first reads; extra cost.
- DLM = automate backup schedule and retention.
- Volume = same AZ as instance.
- Snapshot = same Region unless copied.
- Cross-Region = snapshot → copy → create volume/AMI → validate.
- Backup is not proven until restore is tested.

## Official AWS references
- Volume types: https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html
- Snapshots: https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html
- Snapshot archive: https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-archive.html
- Fast Snapshot Restore: https://docs.aws.amazon.com/ebs/latest/userguide/ebs-fast-snapshot-restore.html
- Snapshot sharing: https://docs.aws.amazon.com/ebs/latest/userguide/ebs-modifying-snapshot-permissions.html
- DLM: https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-lifecycle.html
- EBS encryption: https://docs.aws.amazon.com/ebs/latest/userguide/ebs-encryption.html
