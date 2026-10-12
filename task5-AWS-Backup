# Task 5: Centralized Backup and Restore Using AWS Backup

**Date:** October 12, 2026
**AWS Region:** `us-east-1`
**Backup Vault:** `marwa-centralized-backup-vault`
**Backup Plan:** `marwa-centralized-backup-plan`

---

## 1. Project Overview

The goal of this task was to configure centralized backups for four AWS resources and verify that they could be restored successfully.

The resources included:

* **Amazon EC2:** A virtual machine running a web server.
* **Amazon EBS:** A block storage volume containing a test file.
* **Amazon EFS:** A network file system containing a test file.
* **Amazon S3:** An object storage bucket containing a test file.

AWS Backup was used to create and manage recovery points. Each resource was restored, and the recovered application or data was checked.

**Final result:** Backup and restore operations were completed for EC2, EBS, EFS, and S3, with validation performed for each resource.

---

## 2. AWS Backup Configuration

### Backup Vault

Created and used the following backup vault:

`marwa-centralized-backup-vault`

**Explanation:** A backup vault is a logical container that stores and organizes recovery points created by AWS Backup.

### Backup Plan

Created the backup plan:

`marwa-centralized-backup-plan`

The plan included the following configuration:

| Setting                  | Value                     |
| ------------------------ | ------------------------- |
| Backup rule              | `marwa-daily-backup-rule` |
| Schedule                 | Daily at 05:00 UTC        |
| Egypt local time         | 07:00 AM                  |
| Recovery point retention | 7 days                    |
| Start window             | 60 minutes                |
| Completion window        | 180 minutes               |
| Resources                | EC2, EBS, EFS, S3         |

A resource selection named `marwa-task5-four-resources` was configured to include the four original resources.

The instructor-provided IAM service role was used without modifying IAM policies or roles.

### On-Demand Backups

On-demand backups were started to create recovery points immediately instead of waiting for the scheduled backup.

All four backup jobs completed successfully.

| Resource | Backup Job ID                          | Recovery Point                                                              |
| -------- | -------------------------------------- | --------------------------------------------------------------------------- |
| EC2      | `43a05ee5-d90d-4466-a580-22565397ec49` | AMI: `ami-0f68777deac075710`                                                |
| EBS      | `82dd5ff5-9364-4c00-8246-48af227a3c9a` | Snapshot: `snap-05acfe6ebc6cc3dc3`                                          |
| EFS      | `99398462-261b-4a75-b555-20a813023eb9` | Recovery point: `98688e4e-5345-4495-a8e0-f719f8295d6c`                      |
| S3       | `7203ddcb-15e3-4595-952f-8a6dec27d7d3` | Recovery point: `marwa-centralized-backup-20261012-20261012125424-38b55717` |

**Important:** A completed backup job confirms that a recovery point was created. A separate restore and validation test is needed to confirm that the recovered resource or data is usable.

---

## 3. EC2 Backup and Restore

### Original Resource

* Instance name: `marwa-backup-1`
* Instance ID: `i-018a94510ce680ac9`
* Instance type: `t3.micro`
* Availability Zone: `us-east-1b`

### Restored Resource

* Restored instance ID: `i-065cf2c9419e35361`

### What We Did

1. Created an EC2 recovery point using AWS Backup.
2. Restored the recovery point as a new EC2 instance.
3. Started the restored instance.
4. Opened the web page served by the restored instance.

### Validation

The restored web page was opened successfully and displayed:

`Marwa AWS Backup Lab`

### Explanation

An EC2 recovery point can be used to recover an instance and its system and application configuration. Restoring to a new instance allows the recovered environment to be tested without replacing the original instance.

**Result:** Successful restoration and web application validation.

**Screenshots:**

* [Insert screenshot: EC2 restore job showing Completed]
* [Insert screenshot: Restored EC2 instance in the console]
* [Insert screenshot: Restored web page]

---

## 4. EBS Backup and Restore

### Original Resource

* Volume ID: `vol-079bf4f30843ac5e5`
* Volume size: 2 GiB
* Volume type: gp3
* Availability Zone: `us-east-1b`
* Original mount point: `/data`

The original volume contained:

`/data/ebs-backup-test.txt`

Expected content:

```text
Marwa AWS Backup Lab - EBS recovery test - October 2026
```

### Restored Resource

* Volume ID: `vol-014a4e6e49a7a742d`
* Volume size: 2 GiB
* Volume type: gp2
* Availability Zone: `us-east-1b`

### What We Did

1. Created an EBS recovery point using AWS Backup.
2. Restored the recovery point as a new EBS volume.
3. Attached the restored volume to the restored EC2 instance.
4. Identified the Linux device as `/dev/nvme2n1`.
5. Mounted the existing file system at `/mnt/ebs-restore`.
6. Checked the recovered test file.

The volume was identified as an existing ext4 file system.

**Important:** We did not format the restored volume because formatting could overwrite the recovered file system and its data.

### Validation

The recovered file was checked at:

`/mnt/ebs-restore/ebs-backup-test.txt`

Expected content:

```text
Marwa AWS Backup Lab - EBS recovery test - October 2026
```

### Explanation

Amazon EBS provides block storage for EC2 instances. AWS Backup can restore an EBS recovery point into a new volume.

The restored volume must be attached to an EC2 instance in the same Availability Zone. After attachment, the existing file system must be identified and mounted carefully.

**Result:** Successful volume restoration, attachment, mounting, and test-file verification.

**Screenshots:**

* [Insert screenshot: Restored EBS volume in the console]
* [Insert screenshot: Volume attachment to the restored EC2 instance]
* [Insert screenshot: Mounted volume and recovered test file]

---

## 5. EFS Backup and Restore

### Original Resource

* File system name: `marwa-efs`
* File system ID: `fs-0236f6823ae084f99`

The original EFS contained:

`efs-backup-test.txt`

Expected content:

```text
Marwa AWS Backup Lab - EFS recovery test - October 2026
```

### Restored Resource

* Restored file system ID: `fs-0d82f07197eee9142`
* Mount target IP: `172.31.38.65`
* Region: `us-east-1`
* Availability Zone used for the mount target: `us-east-1b`

### What We Did

1. Created an EFS recovery point using AWS Backup.
2. Restored the recovery point to a new EFS file system.
3. Created a mount target in the VPC.
4. Used the lab security group to provide the required network access.
5. Mounted the restored EFS from the restored EC2 instance.
6. Located and checked the recovered test file.

The restored file system was mounted at:

`/mnt/efs-restore`

Mount command:

```bash
sudo mkdir -p /mnt/efs-restore
sudo mount -t nfs4 -o nfsvers=4.1 172.31.38.65:/ /mnt/efs-restore
```

AWS Backup placed the restored data inside a timestamped directory:

`aws-backup-restore_2026-10-12T13-57-39-084270622Z`

The test file was located within that directory.

### Validation

The restored file system was mounted successfully from EC2, and the recovered test file was checked.

Expected content:

```text
Marwa AWS Backup Lab - EFS recovery test - October 2026
```

### Explanation

Amazon EFS is a managed network file system that can be mounted by multiple clients.

Unlike EBS, which provides block storage to an instance, EFS is accessed over the network using a file protocol such as NFS.

To mount EFS successfully, the client must have network connectivity to an EFS mount target, and the security group must allow NFS traffic over TCP port `2049`.

The restored files may be placed in a timestamped directory, so the directory structure should be inspected after restoration.

**Result:** Successful EFS restoration, mount-target configuration, mounting, and test-file verification.

**Screenshots:**

* [Insert screenshot: Restored EFS file system]
* [Insert screenshot: EFS mount target and security group]
* [Insert screenshot: Successful EFS mount and recovered test file]

---

## 6. S3 Backup and Restore

### Original Resource

* Bucket name: `marwa-centralized-backup-20261012`
* Region: `us-east-1`
* Versioning: Enabled
* Block Public Access: Enabled
* Encryption: SSE-S3

The original bucket contained:

`s3-backup-test.txt`

Expected content:

```text
Marwa AWS Backup Lab - S3 recovery test - October 2026
```

### Restored Destination Bucket

`marwa-s3-20261012-restored`

### What We Did

1. Created an S3 recovery point using AWS Backup.
2. Restored the recovery point to a separate destination bucket.
3. Opened the restored bucket in the S3 console.
4. Checked that `s3-backup-test.txt` was present.
5. Verified the restored object's content.
6. Kept the destination bucket private.

### Validation

The restore job completed, and the restored test object was checked in:

`marwa-s3-20261012-restored`

Expected content:

```text
Marwa AWS Backup Lab - S3 recovery test - October 2026
```

The original bucket was retained.

### Explanation

Amazon S3 is object storage. Objects are stored in buckets and identified by object keys.

AWS Backup can restore backed-up S3 data to a destination bucket. Using a separate bucket lets us validate the recovered object without intentionally overwriting the original data.

Block Public Access should remain enabled because public access is unnecessary for this recovery test.

**Result:** Successful S3 restoration and object verification.

**Screenshots:**

* [Insert screenshot: S3 restore job showing Completed]
* [Insert screenshot: Object list in the restored bucket]
* [Insert screenshot: Restored object's content]
* [Insert screenshot: Destination bucket Block Public Access settings]

---

## 7. Summary of Validation Results

| Resource | Recovery action               | Validation performed                                              | Result     |
| -------- | ----------------------------- | ----------------------------------------------------------------- | ---------- |
| EC2      | Restored as a new instance    | Opened the restored web page                                      | Successful |
| EBS      | Restored as a new volume      | Attached, mounted, and checked the test file                      | Successful |
| EFS      | Restored as a new file system | Created mount target, mounted from EC2, and checked the test file | Successful |
| S3       | Restored to a separate bucket | Checked the object and its contents                               | Successful |

---

## 8. Key Concepts Learned

### AWS Backup

AWS Backup centralizes backup management for supported AWS services. It provides backup plans, resource assignments, backup vaults, recovery points, and restore jobs.

### Backup Plan vs. Backup Vault

* **Backup plan:** Defines when backups run and how long recovery points are retained.
* **Backup vault:** Stores and organizes recovery points.

### Recovery Point

A recovery point is a saved backup state that can be used to restore a resource or its data. The underlying recovery mechanism depends on the service—for example, an EC2 recovery point can involve an AMI, while an EBS recovery point uses a snapshot.

### On-Demand vs. Scheduled Backup

* **On-demand backup:** Started manually when an immediate backup is needed.
* **Scheduled backup:** Runs automatically according to the backup plan.

On-demand backups are useful for testing without waiting for the next scheduled execution.

### Why Restore to New Resources?

Restoring to new resources makes it possible to test recovery independently and reduces the risk of overwriting the original environment.

### Why Validate After Restoration?

A restore job showing `Completed` is important, but it does not by itself prove that the application or recovered data works as expected.

Validation should check the actual result—for example, opening the restored web page, mounting the restored storage, and verifying recovered files.

### EBS vs. EFS vs. S3

| Service | Storage type                 | How it is accessed                                              |
| ------- | ---------------------------- | --------------------------------------------------------------- |
| EBS     | Block storage                | Attached to an EC2 instance                                     |
| EFS     | Managed network file storage | Mounted over the network, typically using NFS                   |
| S3      | Object storage               | Accessed through the S3 API, console, or supported integrations |

These services serve different purposes, so their restore and validation procedures are different.

---

## 9. Security and Cost Considerations

* The original EC2 instance, EBS volume, EFS file system, and S3 bucket were retained.
* Restores were performed separately where applicable.
* The instructor-provided AWS Backup IAM role was used; no IAM roles or policies were modified.
* The S3 destination bucket should remain private, with all Block Public Access settings enabled.
* AWS resources, backup recovery points, snapshots, EFS mount targets, and stored data may incur charges.
* Before ending the lab, review which temporary resources are still needed for evidence.
* Do not delete original resources or recovery points before the report and required screenshots are complete.
* After collecting evidence, clean up only resources that are no longer needed and that you can safely identify as temporary. Check dependencies before deleting instances, volumes, mount targets, or recovery points.

---

## 10. Final Checklist

* [x] Created the AWS Backup vault.
* [x] Configured a daily backup plan with 7-day retention.
* [x] Selected EC2, EBS, EFS, and S3.
* [x] Ran on-demand backups.
* [x] Confirmed completed recovery points.
* [x] Restored EC2 and tested the web page.
* [x] Restored EBS, attached and mounted it without formatting, and checked the test file.
* [x] Restored EFS, configured a mount target, mounted it from EC2, and checked the test file.
* [x] Restored S3 to `marwa-s3-20261012-restored` and checked the test object.
* [ ] Insert screenshots into the report.
* [ ] Review and clean up temporary resources after collecting evidence.

---

## 11. Conclusion

This task demonstrated a centralized backup and recovery workflow for four AWS resource types: EC2, EBS, EFS, and S3.

AWS Backup was used to organize the backup configuration and create recovery points. The restore tests then confirmed that the web application could run on a restored EC2 instance and that test data could be recovered from EBS, EFS, and S3.

The main lesson is that **a backup is only useful if the recovery process works**. A reliable backup strategy should therefore include regular restore tests, data or application validation, appropriate access controls, and cost-aware cleanup.
