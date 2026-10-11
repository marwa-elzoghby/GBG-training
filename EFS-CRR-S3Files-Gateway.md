# Task 4 Guided Lab: Shared, Hybrid and Replicated Storage on AWS

By: (your name)

**How to use this guide.** Work top to bottom. Sections 1 to 5 are background and preparation. Section 6 is the hands-on lab (four Parts, each with small steps). Sections 7 to 12 cover testing, the demo, the presentation talk, troubleshooting, cleanup and the final checklist. Take the screenshots listed in the task as you go; each Part ends with a screenshot list.

**Assumptions I made (change them if your instructor says otherwise)**

- **Primary Region: `us-east-1` (Ireland).** Used for Parts 1, 2, 4 and as the CRR source. You can use any Region your instructor names, but it must support Amazon S3 Files (Part 2) and Storage Gateway.
- **DR Region for Part 3: `us-west-2` (Frankfurt).** Any Region different from the source works.
- **Unique suffix:** your lowercase name plus 4 digits, for example `sara4821`. Wherever you see `SUFFIX`, replace it with yours. Bucket names must be globally unique, lowercase, no spaces.
- **VPC:** the Default VPC with its default public subnets (simplest, no networking to build).
- **Instructor-provided IAM:** you cannot create or edit IAM policies. You will be given (or must ask for) the four items in the table in Section 4.
- **Amazon S3 Files** is a new service (generally available since April 2026). Its console screens may be labelled slightly differently from what I describe; the concepts and CLI commands are stable. The task also allows you to use an instructor-prepared file system if the console blocks you.
- **Operating system:** Amazon Linux 2023 (AL2023) for every EC2 instance. Default user is `ec2-user`.

---


## Marwa's deployed resource record (use these values in the steps below)

The following values record resources already created for this lab. Use the exact ID and Region when a step asks you to select an existing resource. This record documents the deployment; it does not mean every later test or configuration step has already been completed.

| Part | Resource | Value |
| --- | --- | --- |
| 1 | Region | `us-east-1` (N. Virginia) |
| 1 | Availability Zones | AZ1: `us-east-1a`; AZ2: `us-east-1b` |
| 1 | EC2 instance 1 | `efs-marwa-1` — `i-077490010a76d2d44` |
| 1 | EC2 instance 2 | `efs-marwa-2` — `i-00b303555dfbdb19e` |
| 1 | EFS | `marwa-efs` — `fs-0236f6823ae084f99` |
| 1 | Security group | `marwa-sg-4` — `sg-0557fb692d48d17dc` |
| 1 | VPC | `marwa-vpc-default` — `vpc-0af78ffcfe768a5be` |
| 2 | S3 bucket | `marwa-s3-files-2026` |
| 2 | Security group | `s3files-marwa-sg` — `sg-0e0d7b127f5890e84` |
| 2 | S3 Files file system | `fs-0090781e61cc01486` |
| 2 | S3 Files access role used to create the file system | `arn:aws:iam::897192298286:role/S3FilesAccessRole` |
| 2 | EC2 instance role | `EC2-S3Files-Role-marwan` |
| 2 | Previous EC2 client tried | `efs-marwa-1` (not confirmed as the current S3 Files client) |
| 3 | Source Region / bucket | `us-east-1` (N. Virginia) / `marwa-source-2026` |
| 3 | Destination Region / bucket | `us-west-2` (Oregon) / `marwa-destination-2026` |
| 3 | Replication role | `Marwa-S3-Replication-Role` |
| 4 | Region / bucket | `us-east-1` / `file-gateway-marwa` |
| 4 | File share ID | `share-302ECF5A` |
| 4 | Gateway EC2 instance | `gateway-marwa` |

### Part 2: CLI-only S3 Files creation note

The S3 Files file system was created with the CLI because the required IAM role was not authorized through the console workflow. It already exists, so do not rerun the create command unless rebuilding it is intended. Keep the role and command explicit in the record:

```bash
aws s3files create-file-system --bucket arn:aws:s3:::marwa-s3-files-2026 --role arn:aws:iam::897192298286:role/S3FilesAccessRole --region us-east-1
```

Use the existing file system ID `fs-0090781e61cc01486` for subsequent mount-target and client steps. The EC2 client uses `EC2-S3Files-Role-marwan`. Do not replace this with the generic example role names in the guide. Where a task step requires an IAM role that you cannot pass or select in the console, perform that specific operation using the CLI with the authorized role, or use the instructor-prepared resource; do not create or alter IAM roles or policies.

### Parts 1, 3 and 4: names and IDs

Part 1 uses the existing VPC and security group listed above, with the EFS mount targets in `us-east-1a` and `us-east-1b`. Part 3 uses Oregon (`us-west-2`) for the destination bucket. Part 4 uses the existing file share ID `share-302ECF5A`; select gateway `gateway-marwa` and bucket `file-gateway-marwa` when the console requests them.

# 1. Task Understanding

**In simple words:** AWS has several ways to store files. This task makes you build four of them and prove each one works.

| Part | What we build | The idea in one sentence | What the final demo looks like |
| --- | --- | --- | --- |
| 1 | Amazon EFS shared file system | One network drive that many Linux servers mount at the same time | Create a file on Server 1, immediately read it on Server 2 (and the reverse) |
| 2 | Amazon S3 Files | Mount an S3 bucket like a normal folder, no S3 API code | Write a file in `/mnt/s3files`, see it as an S3 object; upload to S3, see it in the folder |
| 3 | S3 Cross-Region Replication (CRR) | Automatic copy of every new object to a bucket in another Region for disaster recovery | Upload to the source bucket, watch it appear in the DR bucket with status COMPLETED |
| 4 | S3 File Gateway | An on-premises style file share (NFS) whose data lives in S3 | Write to `/mnt/cloudmart-share`, see the object in S3; upload to S3, Refresh Cache, see it in the share |

**Services used and why**

- **EC2:** the Linux servers that mount the storage (and, in Part 4, the gateway appliance that pretends to be an on-premises server).
- **EFS:** managed NFS storage that is shared across Availability Zones.
- **S3:** durable object storage; the base of Parts 2, 3 and 4.
- **S3 Files:** gives S3 buckets a file-system (NFS) front door.
- **Storage Gateway (S3 File Gateway):** bridges on-premises file protocols (NFS/SMB) to S3 with a local cache.
- **VPC security groups:** the firewall that restricts NFS (port 2049) to authorised clients only.
- **IAM (provided roles only):** lets S3 Files, replication and the gateway act on your buckets.
- **CloudShell (optional but recommended):** a free browser terminal with the AWS CLI already logged in; makes creating and uploading test files easy.

**Checkpoint:** you can state in one sentence what each of the four Parts does. If yes, continue.

---

# 2. Final Architecture

**Part 1: EFS (shared storage)**

```
          VPC (default)  Region us-east-1
  +----------------------------------------------+
  |  AZ-a                         AZ-b           |
  |  EC2 'efs-marwa-1'              EC2 'efs-marwa-2'|
  |   /mnt/efs                     /mnt/efs      |
  |      |  (TLS, TCP 2049)           |          |
  |  Mount target a  <--- EFS ---> Mount target b|
  |   (SG: marwa-sg-4 allows 2049 only from marwa-sg-4)
  +----------------------------------------------+
        One Regional, encrypted EFS file system
```

Flow: (1) Instance 1 writes a file to /mnt/efs. (2) The mount helper wraps NFS traffic in TLS and sends it to the mount target in its own AZ. (3) EFS stores the data redundantly across AZs. (4) Instance 2 reads the same file through its own mount target. The security group only lets traffic from `marwa-sg-4` reach port 2049.

**Part 2: S3 Files**

```
 EC2 (AL2023, instructor client role)
   /mnt/s3files  --NFS 2049-->  S3 Files mount target (same AZ)
                                   |  (sync, asynchronous, via instructor access role)
                                   v
                       S3 bucket marwa-s3-files-2026
                       (versioning ON, SSE-S3, Block Public Access ON)
```

Flow: write a file in the folder, S3 Files syncs it to the bucket as an object (about a minute). Upload an object to S3, S3 Files imports the change and the file appears in the folder.

**Part 3: Cross-Region Replication**

```
 Upload --> marwa-source-2026 (us-east-1, versioning ON)
                  |  replication rule (enabled, instructor role, asynchronous)
                  v
           marwa-destination-2026 (us-west-2, versioning ON)
```

Flow: a new object version lands in the source bucket with status PENDING; S3 uses the replication role to copy it to the DR bucket; status becomes COMPLETED and the copy shows as REPLICA.

**Part 4: S3 File Gateway**

```
 Client EC2 (simulated on-prem app)
   /mnt/cloudmart-share --NFS--> File Gateway EC2 (m5.xlarge, 80 GiB root + 150 GiB cache disk)
                                       |  HTTPS (async upload, instructor file share role)
                                       v
                         S3 bucket file-gateway-marwa (private)
   Your browser --port 80 (activation only, from your IP)--> Gateway
```

Flow: the client writes to the share; data lands in the gateway's local cache; the gateway uploads it to S3 asynchronously. For files added directly to S3, you run Refresh Cache so the gateway re-reads the bucket listing.

**Checkpoint:** you understand that all four are file-style access to storage, but with different back ends (EFS, S3 via S3 Files, S3 via replication, S3 via gateway).

---

# 3. AWS Services

| Service | Purpose | Why do we need it? |
| --- | --- | --- |
| Amazon EC2 | Linux clients and the gateway appliance | Something has to mount the storage |
| Amazon EFS | Shared NFS file system | Task Part 1 |
| Amazon S3 | Buckets for Parts 2, 3, 4 | Durable object storage |
| Amazon S3 Files | NFS access to an S3 bucket | Task Part 2 |
| S3 Replication (CRR) | Automatic cross-Region copy | Task Part 3 |
| AWS Storage Gateway (S3 File Gateway) | Hybrid file share backed by S3 | Task Part 4 |
| VPC security groups | Restrict NFS to approved clients | Security requirement in Parts 1, 2, 4 |
| IAM (instructor roles) | Permissions for S3 Files, replication, gateway share, EC2 client | You must not create IAM, so you use theirs |
| AWS CloudShell (optional) | Browser terminal with AWS CLI | Easy file creation and CLI verification |

Not needed: Application Load Balancer, Route 53, CloudFront, Lambda, KMS (we use SSE-S3), NAT gateway.

---

# 4. Prerequisites

**Required**

- An AWS account you can sign in to with the console (instructor-provided account or sandbox).
- Permissions (from your instructor) to create EC2, EFS, S3 buckets, security groups, Storage Gateway, and to **pass** the instructor roles. You do not need to create IAM policies.
- A Region selected in the top-right of the console: `us-east-1`.
- An EC2 key pair in `us-east-1` (EC2 > Key pairs > Create key pair, name `lab-key`, RSA, .pem) **OR** use EC2 Instance Connect from the browser (no key needed). Browser connect is easier; this guide uses it. It needs SSH (port 22) allowed in the security group.
- A modern browser. No software to install locally.

**Ask your instructor for these (write them down)**

| Item | Used in | Example format |
| --- | --- | --- |
| S3 Files **access role** (lets S3 Files sync your bucket) | Part 2 step 2.4 | `S3FilesAccessRole` |
| EC2 **client role / instance profile** for S3 Files | Part 2 step 2.6 | `EC2-S3Files-Role-marwan` |
| S3 **replication role** (must trust s3.amazonaws.com and cover your bucket names; ask whether bucket names must follow a pattern) | Part 3 step 3.4 | `Marwa-S3-Replication-Role` |
| Storage Gateway **file share role** (allows the gateway to write to its bucket) | Part 4 step 4.7 | Check the existing share details; not provided in the deployment notes |
| Approved EC2 instance type for the gateway | Part 4 step 4.3 | `m5.xlarge` |

**OPTIONAL:** a small PNG image on your computer for Part 3. If you do not have one, Step 3.5 creates one in CloudShell.

**Checkpoint:** you are signed in, region is `us-east-1`, you know your SUFFIX and have the role names.

---

# 5. Cost and Free Tier

Prices are approximate (us/EU list prices in 2026) and change; check the AWS pricing pages. Do not rely on Free Tier unless your account type confirms it.

| Resource | Cost risk | Likely Free Tier? | Approx. cost for this lab |
| --- | --- | --- | --- |
| 3 x t3.micro EC2 (Parts 1 and 2) | Hourly | Often covered by the Free Tier or account credits (depends on account age/plan) | about $0.01 per hour each if not covered |
| EFS (a few KB of data) | Per GB-month stored | Free Tier may exist for new accounts; otherwise cents | under $0.01 |
| S3 Files (a few KB) | Storage plus sync and request charges | Not sure; assume not | cents |
| S3 buckets and CRR (a few KB) | Storage, requests, inter-Region transfer | S3 Free Tier may apply | under $0.01 |
| **Gateway EC2 m5.xlarge** | **Biggest cost** | **No** | about $0.20 to $0.25 per hour |
| 150 GiB gp3 cache volume plus 80 GiB root | Per GB-month while it exists | No | about $0.03 per hour combined |
| Public IPv4 addresses | Hourly per address | No | about $0.005 per hour each |

**Rough total** if you finish in one sitting and delete things the same day: about **$1 to $3**, almost all from the gateway instance. Leaving the gateway running for a month would cost roughly $150 or more.

**Ways to cut cost**

- Do Part 4 last and delete it as soon as you finish screenshots (after the instructor approves, per the task).
- Stop (not terminate) instances between sessions only if you need to keep them; stopped instances still pay for EBS volumes.
- Use t3.micro everywhere except the gateway.
- Use tiny test files.

**Must delete afterwards:** gateway and its EC2 instance and volumes, all EC2 instances, EFS file system (and its mount targets), S3 Files file system and mount targets, all three buckets (empty them first, including versions), security groups, and any unattached volumes or Elastic IPs. Details are in Section 11.

**Checkpoint:** you know the gateway is the expensive piece and that you must clean up.

---

# 6. Guided Lab: Step by Step

General habit for every EC2 launch: always double check **Region**, **Subnet/AZ**, **Security group**, and **Auto-assign public IP** before clicking Launch.

The four Parts are independent. You may do them in the order given; Part 1 creates `marwa-sg-4`, which Part 2 and Part 4 reuse as the client security group.

## PART 1: Shared File Storage with Amazon EFS

**Concept:** EFS is a managed NFS file system. It is *Regional*: data is stored across multiple AZs. To reach it, each AZ you use needs a *mount target* (a network interface with an IP address in your subnet). Clients connect to it on TCP port 2049.

### Step 1.1: Select the Region and note your Default VPC

**Goal:** Make sure everything is built in one Region and one VPC.

**Where:** AWS Console, top-right Region selector. Then VPC > Your VPCs.

**What to enter:** Region: `us-east-1` (Europe, Ireland).

**Action:** Choose the Region. In VPC > Your VPCs, find the VPC where *Default VPC* = Yes and write down its VPC ID.

**Expected Result:** You see one Default VPC (for example `vpc-0abc...`).

**Why?** EFS mount targets and EC2 instances must be in the same VPC to talk to each other.

**Common Problems:** No default VPC exists (it was deleted): ask your instructor, or create one via VPC > Your VPCs > Actions > Create default VPC.

### Step 1.2: Create the EC2 client security group

**Goal:** Create the firewall group that both EC2 instances will use.

**Where:** EC2 > Network & Security > Security Groups > Create security group.

**What to enter:**

- Name: `marwa-sg-4`
- Description: `Lab EC2 clients for Task 4`
- VPC: the Default VPC
- Inbound rule: Type `SSH`, Port 22, Source `My IP`
- Outbound: leave the default (all traffic)

**Action:** Click *Add rule*, set the values, then *Create security group*.

**Expected Result:** `marwa-sg-4` appears in the list with one inbound SSH rule.

**Why?** A security group is a stateful firewall. Later, the EFS group will trust *this group* instead of IP addresses, which is what makes NFS access restricted to your instances only.

**Common Problems:** Browser connect (EC2 Instance Connect) fails later: it needs port 22 reachable. If your instructor's network blocks it, use Session Manager if offered, or open SSH to the EC2 Instance Connect IP range for `us-east-1`. Your public IP changes if you switch networks; edit the rule if SSH stops working.

### Step 1.3: Create the EFS security group

**Goal:** Allow NFS only from the EC2 clients.

**Where:** EC2 > Security Groups > Create security group.

**What to enter:**

- Name: `marwa-sg-4`
- Description: `Allow NFS 2049 from marwa-sg-4`
- VPC: Default VPC
- Inbound rule: Type `NFS` (or Custom TCP), Port `2049`, Source: **Custom**, then select `marwa-sg-4` (start typing `sg-` or the name and pick it from the list)

**Action:** Add the rule, remove any other inbound rules, click *Create security group*.

**Expected Result:** `marwa-sg-4` has exactly one inbound rule: TCP 2049 from `marwa-sg-4`.

**Why?** This satisfies the task requirement that NFS access be restricted to the authorized EC2 instances. No IP ranges, no `0.0.0.0/0`.

**Common Problems:** Using `0.0.0.0/0` as the source (do not). The source box will not show the group: make sure the VPC is the same as `marwa-sg-4`.

### Step 1.4: Create the EFS file system

**Goal:** Create an encrypted, Regional file system with mount targets in two AZs.

**Where:** AWS Console > EFS > File systems > Create file system > **Customize**.

**What to enter:**

- *Step 1 (File system settings):* Name `marwa-efs`; File system type **Regional**; Automatic backups: leave default (or turn off to save cents); Lifecycle management: leave defaults; Performance mode: General Purpose; Throughput mode: Elastic (default); **Enable encryption of data at rest: checked** (AWS-managed key `aws/elasticfilesystem`). Click Next.
- *Step 2 (Network access):* VPC = Default VPC. For **two** Availability Zones, for example `us-east-1a` and `us-east-1b`, keep or choose the subnet, and for each one remove the default security group and select **`marwa-sg-4`**. If the console lists three AZs, you may leave the third, but only use two for the lab; to keep it clean, delete the third mount target.
- *Step 3 (File system policy):* skip. Next, then *Create*.

**Action:** Click through the wizard as above.

**Expected Result:** The file system `marwa-efs` shows Encrypted = Yes and, after about a minute, its Network tab shows mount targets in state *Available*.

**Why?** Mount targets are what instances actually connect to. Two AZs means the file system stays reachable if one AZ fails.

**Common Problems:** If you used the simple *Create* button, EFS attaches the VPC default security group to mount targets, and mounting will time out. Fix: EFS > your file system > Network > Manage > replace the group with `marwa-sg-4`. Remember which two AZs you chose: you must launch the EC2 instances in those same AZs.

### Step 1.5: Verify the file system and mount targets

**Goal:** Capture evidence for the first deliverable.

**Where:** EFS > File systems > `marwa-efs`.

**Action:** On the General tab check *Regional*, *Encrypted*, and copy the **File system ID** (looks like `fs-0123456789abcdef0`). Open the **Network** tab.

**Expected Result:** Two mount targets, each Available, each in a different AZ, each using `marwa-sg-4`.

**Why?** The task asks you to prove the file system is Regional and has mount targets in both AZs.

**Common Problems:** State is *Creating*: wait one minute and refresh.

**Screenshot 1:** EFS General tab plus Network tab with mount targets.

### Step 1.6: Launch EC2 instance 1 (AZ-a)

**Goal:** Create the first client server.

**Where:** EC2 > Instances > Launch instances.

**What to enter:**

- Name: `efs-marwa-1`
- AMI: Amazon Linux 2023 (free tier eligible)
- Instance type: `t3.micro`
- Key pair: your key pair, or *Proceed without a key pair* if you will use browser connect
- Network settings > Edit: VPC = Default VPC; **Subnet = the subnet in `us-east-1a`**; Auto-assign public IP = Enable; Select existing security group = `marwa-sg-4`
- Storage: default 8 GiB

**Action:** Click *Launch instance*.

**Expected Result:** Instance state *Running* and 2/2 checks passed within a couple of minutes.

**Why?** This is the first of two clients in different AZs.

**Common Problems:** Wrong subnet/AZ: check the instance's Networking tab for *Availability Zone*. `t3.micro` unavailable: use `t2.micro` or `t3.small`.

### Step 1.7: Launch EC2 instance 2 (AZ-b)

**Goal:** Create the second client in a different AZ.

**Where:** EC2 > Instances > Launch instances.

**What to enter:** Same as Step 1.6 except Name `efs-marwa-2` and **Subnet = the subnet in `us-east-1b`** (the second AZ where you created a mount target).

**Action:** Launch.

**Expected Result:** Two running instances, one in each AZ. In the instance list, add the *Availability zone* column (gear icon) to show this.

**Why?** Task requirement: instances in different AZs in the same VPC.

**Common Problems:** Both instances end up in the same AZ: re-launch with the other subnet.

**Screenshot 2:** EC2 instance list with both instances and the Availability zone column visible.

### Step 1.8: Connect to instance 1 and install the EFS mount helper

**Goal:** Install `amazon-efs-utils`, which provides `mount -t efs` and TLS support.

**Where:** EC2 > Instances > select `efs-marwa-1` > **Connect** > *EC2 Instance Connect* tab > Connect (user `ec2-user`).

**What to enter (in the terminal):**

```
sudo dnf install -y amazon-efs-utils
mount.efs --version
```

**Expected Result:** The package installs and a version number prints.

**Why?** The mount helper sets up encryption in transit (stunnel/TLS) and picks the correct mount options for you.

**Common Problems:** `Connection timed out` when connecting: security group SSH rule or your public IP changed. `dnf` fails: the instance needs internet access, so confirm it has a public IP and the subnet is a public subnet.

### Step 1.9: Connect to instance 2 and install the EFS mount helper

**Goal:** Same as Step 1.8 on the second server.

**Where:** EC2 > Instances > `efs-marwa-2` > Connect > EC2 Instance Connect.

**What to enter:**

```
sudo dnf install -y amazon-efs-utils
mount.efs --version
```

**Expected Result:** Version prints. Keep both browser terminals open in two tabs.

**Why?** Both clients need the helper.

**Common Problems:** Same as Step 1.8.

### Step 1.10: Mount EFS on instance 1 with TLS and set permissions

**Goal:** Mount the shared file system at `/mnt/efs` with encryption in transit and make it writable.

**Where:** Terminal of `s3files-client` (the dedicated Part 2 client; `efs-marwa-1` is only noted as a previous machine tried).

**What to enter:** Use EFS ID `fs-0236f6823ae084f99`.

```
sudo mkdir -p /mnt/efs
sudo mount -t efs -o tls fs-0236f6823ae084f99:/ /mnt/efs
df -hT /mnt/efs
mount | grep /mnt/efs
sudo chown ec2-user:ec2-user /mnt/efs
sudo chmod 775 /mnt/efs
ls -ld /mnt/efs
```

**Expected Result:** `df -hT` shows type `nfs4` with a huge size (EFS reports 8 EiB). The `mount` line shows `127.0.0.1:/` because TLS traffic goes through a local stunnel proxy. `ls -ld` shows `drwxrwxr-x ... ec2-user ec2-user`.

**Why?** `-o tls` = encryption in transit. The root of a new EFS file system is owned by root, so we hand ownership to `ec2-user`. Both instances run `ec2-user` with the same UID (1000), so permissions behave identically on both. In real systems, matching UIDs/GIDs or EFS access points are how you keep permissions consistent.

**Common Problems:** `mount.nfs4: Connection timed out`: mount target uses the wrong security group (Step 1.4 fix) or the instance is in an AZ with no mount target. `Failed to resolve fs-...`: the Region is wrong, or DNS hostnames are disabled in the VPC (Default VPC has them enabled). `mount: unknown filesystem type 'efs'`: the helper is not installed.

**Screenshot 3a:** Terminal output of `df -hT /mnt/efs` and `mount | grep efs` on instance 1.

### Step 1.11: Mount EFS on instance 2

**Goal:** Mount the same file system on the second server.

**Where:** Terminal of `efs-marwa-2`.

**What to enter:**

```
sudo mkdir -p /mnt/efs
sudo mount -t efs -o tls fs-0236f6823ae084f99:/ /mnt/efs
df -hT /mnt/efs
mount | grep /mnt/efs
ls -ld /mnt/efs
```

**Expected Result:** Same output as instance 1. Ownership already shows `ec2-user` because the permission change was stored in EFS itself, not on the instance.

**Why?** Proof that both instances see the same file system.

**Common Problems:** Same as Step 1.10. No need to run `chown` again.

**Screenshot 3b:** Same outputs on instance 2.

### Step 1.12: Create a file on instance 1 and read it on instance 2

**Goal:** Prove sharing in direction 1 to 2.

**Where:** Instance 1 terminal, then instance 2 terminal.

**What to enter:**

On `efs-marwa-1`:

```
echo "Hello from efs-marwa-1 at $(date)" > /mnt/efs/from-instance1.txt
cat /mnt/efs/from-instance1.txt
hostname
```

On `efs-marwa-2`:

```
ls -l /mnt/efs
cat /mnt/efs/from-instance1.txt
hostname
```

**Expected Result:** The file appears on instance 2 with identical content and a different hostname printed.

**Why?** This is the core use case: user uploads written by one web server are visible to all of them.

**Common Problems:** File not visible: confirm both are mounted to the *same* `fs-` ID (`df -hT`), and that you did not write to a plain folder because the mount failed (`mount | grep efs` must show output).

**Screenshot 4a:** Both terminals showing the file and content.

### Step 1.13: Create a file on instance 2 and read it on instance 1

**Goal:** Prove sharing in the other direction.

**Where:** Instance 2 terminal, then instance 1 terminal.

**What to enter:**

On `efs-marwa-2`:

```
echo "Hello from efs-marwa-2 at $(date)" > /mnt/efs/from-instance2.txt
cat /mnt/efs/from-instance2.txt
md5sum /mnt/efs/*.txt
```

On `efs-marwa-1`:

```
ls -l /mnt/efs
cat /mnt/efs/from-instance2.txt
md5sum /mnt/efs/*.txt
```

**Expected Result:** Both files are visible on both instances, and the `md5sum` hashes match across the two terminals (this proves identical contents).

**Why?** Verifies two-way access and content integrity.

**Common Problems:** `Permission denied`: re-run the `chown`/`chmod` from Step 1.10 on either instance.

**Screenshot 4b:** Both terminals showing the second file and matching checksums.

**Part 1 Checkpoint:** EFS is Regional and encrypted, two mount targets are Available in two AZs, both instances mount `/mnt/efs` over TLS, and files created on one instance appear on the other. Leave everything running for now; Part 2 reuses `marwa-sg-4`.

---

## PART 2: Mounting S3 Data with Amazon S3 Files

**Concept:** S3 Files presents an S3 general purpose bucket as an NFS file system. You mount it with the same `amazon-efs-utils` helper, but with type `s3files`. Changes sync in both directions *asynchronously*. Two IAM roles are involved, both provided by the instructor: an **access role** (lets S3 Files read and write your bucket) and a **client role** (lets the EC2 instance mount the file system).

### Step 2.1: Create the S3 bucket

**Goal:** Create a private, versioned, encrypted bucket for S3 Files.

**Where:** AWS Console > S3 > Buckets > Create bucket.

**What to enter:**

- Bucket type: General purpose
- Bucket name: `marwa-s3-files-2026` (example `cloudmart-s3files-sara4821`)
- AWS Region: `us-east-1`
- Object Ownership: ACLs disabled (default)
- Block all public access: **checked** (default)
- Bucket Versioning: **Enable**
- Default encryption: Server-side encryption with Amazon S3 managed keys (**SSE-S3**); Bucket Key: default

**Action:** Scroll down and click *Create bucket*.

**Expected Result:** The bucket appears in your list in `us-east-1`.

**Why?** S3 Files requires versioning on the bucket it links to. SSE-S3 and Block Public Access satisfy the task's security requirements.

**Common Problems:** `Bucket name already exists`: add more digits to the suffix. Versioning left disabled: open the bucket > Properties > Bucket Versioning > Edit > Enable.

### Step 2.2: Verify versioning and encryption

**Goal:** Capture evidence for the first deliverable.

**Where:** S3 > Buckets > your bucket > **Properties** tab.

**Action:** Look at *Bucket Versioning* and *Default encryption*; also open the *Permissions* tab and confirm *Block all public access* is On.

**Expected Result:** Versioning: Enabled. Default encryption: SSE-S3. Block public access: On.

**Why?** Task testing requirement.

**Common Problems:** None expected.

**Screenshot 5:** Properties tab showing versioning and encryption.

### Step 2.3: Create the mount target security group

**Goal:** Allow NFS from the EC2 client to the S3 Files mount target.

**Where:** EC2 > Security Groups > Create security group.

**What to enter:**

- Name: `s3files-marwa-sg`
- Description: `Allow NFS 2049 from marwa-sg-4 to S3 Files`
- VPC: Default VPC
- Inbound: Type NFS (TCP 2049), Source: Custom, select `marwa-sg-4`

**Action:** Create security group.

**Expected Result:** `s3files-marwa-sg` with one inbound rule.

**Why?** Same pattern as EFS: only the client group may reach port 2049. Note that the client's *outbound* rule (default: all) must also allow it.

**Common Problems:** `marwa-sg-4` no longer exists (you cleaned up Part 1): recreate it as in Step 1.2.

### Step 2.4: Create the S3 Files file system

**Goal:** Link the bucket to a new S3 Files file system and create a mount target.

**Where:** AWS Console > S3 > look in the left menu for **File systems** (the S3 Files section) > Create file system. If you cannot find it, search the console for `S3 Files`.

**What to enter:**

- Bucket: `marwa-s3-files-2026`
- Prefix: leave empty (whole bucket)
- IAM role: the **instructor-provided S3 Files access role**
- VPC: Default VPC; mount target subnet: the subnet in the AZ where you will launch the Part 2 instance (use `us-east-1a`)
- Security group: `s3files-marwa-sg`

**Action:** Create. Wait until status is *Available*.

**Expected Result:** A file system with ID like `fs-0abc...` linked to your bucket, and a mount target in `us-east-1a`.

**Why?** The file system is the translator between S3 objects and file-system semantics; the mount target is its network endpoint.

**Common Problems:** Access denied or no *Create* permission: the task allows you to use the instructor-prepared file system instead; in that case ask for its file system ID and make sure your bucket is the one it is linked to (or use the instructor's bucket for this Part). Role not selectable: ask the instructor for the exact role and that it trusts the S3 Files service.

**CLI alternative (CloudShell, only if the console is blocked):**

```
aws s3files create-file-system --bucket arn:aws:s3:::marwa-s3-files-2026 --role arn:aws:iam::897192298286:role/S3FilesAccessRole --region us-east-1
aws s3files create-mount-target --file-system-id fs-0090781e61cc01486 --subnet-id subnet-XXXXXXXX --security-groups sg-XXXXXXXX --region us-east-1
```

Available subcommands may differ slightly by CLI version; run `aws s3files help` to check. Use `aws s3files list-mount-targets --file-system-id fs-0090781e61cc01486 --region us-east-1` to see its state.

### Step 2.5: Verify the file system and mount target

**Goal:** Confirm everything is Available and in the correct VPC and AZ.

**Where:** S3 > File systems > your file system > details and mount targets.

**Action:** Check status = Available, the linked bucket name, the mount target's VPC, subnet/AZ and security group.

**Expected Result:** One mount target *Available* in `us-east-1a`, same VPC as your instance will be in.

**Why?** Task step 5. The instance must be in the same VPC (and, for best results, the same AZ) as the mount target.

**Common Problems:** Mount target still *Creating*: wait a minute.

**Screenshot 6:** File system details with mount target.

### Step 2.6: Launch the S3 Files client EC2 instance

**Goal:** Create a Linux client with the instructor's client IAM role.

**Where:** EC2 > Instances > Launch instances.

**What to enter:**

- Name: `efs-marwa-1`
- AMI: Amazon Linux 2023
- Type: `t3.micro`
- Network: Default VPC; **Subnet in `us-east-1a`**; Auto-assign public IP: Enable; Security group: `marwa-sg-4`
- Advanced details > **IAM instance profile:** the instructor-provided client role

**Action:** Launch instance. Wait for 2/2 checks.

**Expected Result:** Instance running with the IAM role shown under Security tab > IAM Role.

**Why?** The role gives the instance permission to mount and use the file system without you storing any keys.

**Common Problems:** Role not in the dropdown: it must exist as an *instance profile*; ask the instructor. Wrong AZ: relaunch in the mount target's AZ.

### Step 2.7: Install amazon-efs-utils 3.0.0 or newer

**Goal:** Install the S3 Files mount helper.

**Where:** EC2 > `s3files-client` > Connect > EC2 Instance Connect.

**What to enter:**

```
sudo dnf install -y amazon-efs-utils
rpm -q amazon-efs-utils
mount.s3files --version
```

**Expected Result:** Version 3.0.0 or higher, and `mount.s3files` exists.

**Why?** Older versions do not know the `s3files` file system type (error: `mount.s3files: command not found`).

If the version is lower than 3.0.0, use the official installer script, then re-check:

```
curl https://amazon-efs-utils.aws.com/efs-utils-installer.sh | sudo sh -s -- --install
mount.s3files --version
```

**Common Problems:** The installer needs internet access (public IP, public subnet). Read the output of the script if it asks for extra prerequisites; the AWS documentation lists any client prerequisites (for example required packages).

### Step 2.8: Mount the S3 file system

**Goal:** Mount at `/mnt/s3files`.

**Where:** Terminal of `s3files-client` (the dedicated Part 2 client; `efs-marwa-1` is only noted as a previous machine tried).

**What to enter:** Replace the ID with your S3 Files file system ID.

```
sudo mkdir -p /mnt/s3files
sudo mount -t s3files fs-0090781e61cc01486:/ /mnt/s3files
df -hT /mnt/s3files
ls -la /mnt/s3files
```

**Expected Result:** The mount appears in `df`; the directory is empty (new bucket).

**Why?** The `s3files` mount type handles the protocol and uses the instance role for authorization.

**Common Problems:** Timeout: check port 2049 rules (`s3files-marwa-sg` inbound from `marwa-sg-4`, instance outbound open) and that the instance and mount target share a VPC. Access denied: client role is missing S3 Files permissions, tell your instructor.

**Screenshot 7:** `df -hT /mnt/s3files` output.

### Step 2.9: Write a file through the mount and find it in S3

**Goal:** Prove file-to-S3 sync.

**Where:** Terminal, then S3 console.

**What to enter:**

```
echo "Created via S3 Files mount on $(date)" > /mnt/s3files/from-ec2.txt
cat /mnt/s3files/from-ec2.txt
sync
```

The `>` redirect closes the file when the command finishes, as the task requires. Wait about 1 to 3 minutes. Then open S3 > your bucket > Objects and click refresh.

**Expected Result:** Object `from-ec2.txt` appears. Open it (Open or Download) or view its content; it matches.

**Why?** Writes are batched and synchronized to S3 asynchronously.

**Common Problems:** Object not there yet: wait a few more minutes. Persistent absence: check the file system status and that the instructor's access role is correct.

**Screenshot 8a:** Terminal file content plus S3 object list.

### Step 2.10: Upload a different file directly to S3 and read it on EC2

**Goal:** Prove S3-to-file sync.

**Where:** S3 console (or CloudShell) and the EC2 terminal.

**What to enter:** On your computer create `from-s3.txt` containing `Uploaded directly to S3`. In S3 > bucket > Upload > Add files > select it > Upload. (Or in CloudShell: `echo "Uploaded directly to S3" > from-s3.txt && aws s3 cp from-s3.txt s3://marwa-s3-files-2026/from-s3.txt`.) Then on the EC2 terminal:

```
ls -l /mnt/s3files
cat /mnt/s3files/from-s3.txt
```

**Expected Result:** `from-s3.txt` appears with the same text. Repeat `ls` after a minute if it is not there yet.

**Why?** S3 Files imports changes made directly in the bucket. The two test files use different names, as required, so there is no conflict.

**Common Problems:** File not visible immediately: synchronization is asynchronous; wait and retry. Do not edit the same file in both places at once.

**Screenshot 8b:** S3 upload plus the file visible and readable in the mounted directory.

**Part 2 Checkpoint:** versioned, SSE-S3 bucket; S3 Files file system and mount target Available; `/mnt/s3files` mounted; files sync in both directions.

---

## PART 3: S3 Cross-Region Replication (CRR)

**Concept:** Replication copies *new object versions* from a source bucket to a destination bucket automatically. It needs versioning on both buckets and an IAM role that S3 assumes to read from the source and write to the destination. It is asynchronous and only applies to objects uploaded **after** the rule exists.

### Step 3.1: Create the source bucket

**Goal:** Create the source in `us-east-1`.

**Where:** S3 > Buckets > Create bucket.

**What to enter:** Name `marwa-source-2026`; Region `us-east-1`; Block all public access: checked; Versioning: **Enable**; Default encryption: SSE-S3.

**Action:** Create bucket.

**Expected Result:** Bucket created in `us-east-1`.

**Why?** This is the primary copy. Versioning is mandatory for replication.

**Common Problems:** Name taken: change the suffix. If the instructor's role only covers certain bucket names, use the required naming.

### Step 3.2: Create the destination bucket in another Region

**Goal:** Create the DR bucket in `us-west-2`.

**Where:** S3 > Buckets > Create bucket (the create page lets you choose the Region).

**What to enter:** Name `marwa-destination-2026`; AWS Region `us-west-2` (Europe, Frankfurt); Block all public access: checked; Versioning: **Enable**; Default encryption: SSE-S3.

**Action:** Create bucket.

**Expected Result:** The S3 bucket list shows both buckets, with different Regions in the *AWS Region* column.

**Why?** Cross-Region means the destination must be in a different Region, so a Region outage does not affect both copies.

**Common Problems:** Accidentally chose the same Region: delete and recreate. Versioning not enabled on the destination: the rule save will fail with a versioning error.

### Step 3.3: Verify both buckets, Regions and versioning

**Goal:** Capture deliverable screenshots.

**Where:** S3 > Buckets (list view), then each bucket > Properties > Bucket Versioning.

**Action:** Screenshot the list showing both names and Regions; screenshot Versioning = Enabled on each bucket.

**Expected Result:** Both buckets show Versioning Enabled; Regions differ.

**Why?** Task testing requirements.

**Common Problems:** None expected.

**Screenshots 9a and 9b:** bucket list with Regions; Versioning on both buckets.

### Step 3.4: Create the replication rule

**Goal:** Replicate every new object from source to destination using the instructor's role.

**Where:** S3 > `marwa-source-2026` > **Management** tab > Replication rules > Create replication rule.

**What to enter:**

- Replication rule name: `replicate-all-to-dr`
- Status: **Enabled**
- Rule scope: *Apply to all objects in the bucket* (tick the acknowledgement)
- Destination: *Choose a bucket in this account* > Browse > `marwa-destination-2026`
- IAM role: *Choose from existing IAM roles* > select the instructor-provided replication role
- Encryption / storage class / other options: leave defaults
- Optional: *Delete marker replication*: leave unchecked for now (see Step 3.8)

**Action:** Click *Save*. When asked **Replicate existing objects?**, choose **No, do not replicate existing objects**, then Submit.

**Expected Result:** The Replication rules table shows `replicate-all-to-dr`, Status Enabled, destination bucket and Region, and the role.

**Why?** Task rule: only objects uploaded after the rule is enabled are replicated live. Existing objects would need S3 Batch Replication, which is out of scope.

**Common Problems:** Role not in the list or `Access denied` when saving: the instructor's role must trust `s3.amazonaws.com` and cover both buckets. Error about versioning: enable it on the destination. Cannot find the destination: it must be in your account and a different Region.

**Screenshot 10:** Replication rule showing Enabled, destination and role.

### Step 3.5: Create the test files

**Goal:** Make two small PNG test files.

**Where:** Open **CloudShell** (the terminal icon in the top bar of the console) with Region `us-east-1` selected.

**What to enter:**

```
echo 'iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mP8z8BQDwAEhQGAhKmMIQAAAABJRU5ErkJggg==' | base64 -d > product-image-01.png
cp product-image-01.png product-image-02.png
ls -l *.png
md5sum *.png
```

(You can use your own image files instead; just keep the file names.)

**Expected Result:** Two small PNG files are listed, with their hashes.

**Why?** We need real image files with a stable checksum so we can compare source and destination.

**Common Problems:** CloudShell not available in your Region: use your computer's own images and the console Upload.

### Step 3.6: Upload product-image-01.png and check the destination

**Goal:** Test replication of a root-level object.

**Where:** CloudShell, then S3 console (source bucket, then destination bucket in `us-west-2`).

**What to enter:**

```
aws s3 cp product-image-01.png s3://marwa-source-2026/product-image-01.png --region us-east-1
```

**Action:** Wait 1 to 2 minutes (up to 15 minutes in rare cases). Open the destination bucket and refresh. Then open the source object > Properties > **Replication status**.

**Expected Result:** `product-image-01.png` shows in the destination bucket. Source status moves PENDING then **COMPLETED**; the destination copy shows *REPLICA*.

**Why?** Shows the rule works with no manual copy.

**Common Problems:** Nothing appears: the rule is disabled, the role lacks permissions, or you uploaded before creating the rule. Status FAILED: role/permission problem; show the instructor. Never upload to the destination yourself.

### Step 3.7: Upload images/product-image-02.png and compare

**Goal:** Test replication of a prefixed key and check the prefix is kept.

**Where:** CloudShell, then S3 console.

**What to enter:**

```
aws s3 cp product-image-02.png s3://marwa-source-2026/images/product-image-02.png --region us-east-1
```

**Action:** After a minute or two, open the destination bucket: you should see the `images/` folder. Open it. Check checksums and status:

```
aws s3api head-object --bucket marwa-source-2026 --key images/product-image-02.png --region us-east-1 --query ReplicationStatus
aws s3api head-object --bucket marwa-destination-2026 --key images/product-image-02.png --region us-west-2 --query ReplicationStatus
aws s3 cp s3://marwa-destination-2026/images/product-image-02.png /tmp/dr-copy.png --region us-west-2
md5sum product-image-02.png /tmp/dr-copy.png
```

**Expected Result:** Source status `COMPLETED`, destination status `REPLICA`, identical md5 hashes, key `images/product-image-02.png` identical on both sides.

**Why?** Keys (including prefixes) are preserved by replication.

**Common Problems:** `images/` is just a key prefix, not a real folder; if it is missing, the object has not replicated yet.

**Screenshots 11 and 12:** source and destination object lists side by side; object Properties showing Replication status COMPLETED.

### Step 3.8 (OPTIONAL): Delete marker replication

**Goal:** Show that deletions can be replicated as delete markers.

**Where:** Source bucket > Management > Replication rules > select the rule > Edit rule.

**What to enter:** Tick **Delete marker replication**, then Save.

**Action:** Upload a disposable file (`echo test > disposable.txt && aws s3 cp disposable.txt s3://marwa-source-2026/ --region us-east-1`), wait for it to replicate, then in the source bucket with *Show versions* off delete it (this adds a delete marker). After a minute or two, check the destination: turn *Show versions* on and look for a delete marker.

**Expected Result:** A delete marker appears at the destination; older versions remain.

**Why?** By default deletes are *not* replicated. Even with it enabled, specific version deletions (permanent deletes) are never replicated, which protects against malicious deletion.

**Common Problems:** Marker missing: wait longer, or confirm the checkbox was saved.

**Part 3 Checkpoint:** two buckets in different Regions, versioning on both, enabled rule with instructor role, both test objects replicated and COMPLETED.

---

## PART 4: Amazon S3 File Gateway (Hybrid Storage)

**Concept:** An S3 File Gateway is a virtual appliance you normally run on-premises (VMware, Hyper-V, KVM). It exposes an NFS or SMB share and keeps recently used data in a **local cache disk**; it uploads new files to S3 asynchronously. Here we simulate on-premises by running the appliance on EC2. Three things live in the lab: the **gateway instance**, a **client instance** that mounts the share, and the **S3 bucket**.

Important: the gateway is the only expensive part of this lab. Do Part 4 last and be ready to delete it quickly.

### Step 4.1: Create the bucket

**Goal:** Create the private bucket behind the file share.

**Where:** S3 > Buckets > Create bucket.

**What to enter:** Name `file-gateway-marwa`; Region `us-east-1`; Block all public access: checked; Versioning: your choice (leave default Disabled; it is not required here); Default encryption: SSE-S3.

**Action:** Create bucket.

**Expected Result:** Bucket exists in `us-east-1`.

**Why?** This is where gateway files end up as objects. The task asks for a separate bucket for Part 4.

**Common Problems:** Name collision: change the suffix. The gateway and the bucket should be in the same Region for the simplest setup.

### Step 4.2: Create the security groups

**Goal:** Allow the client to reach the gateway on NFS ports, and your browser to activate it, without exposing anything publicly.

**Where:** EC2 > Security Groups > Create security group (twice).

**What to enter:**

Group 1: Name `gateway-marwa-client-sg`, Default VPC, inbound: SSH 22 from `My IP`.

Group 2: Name `fgw-gateway-sg`, Default VPC, inbound rules:

| Type | Protocol / Port | Source | Purpose |
| --- | --- | --- | --- |
| HTTP | TCP 80 | My IP | Activation only (you will delete this rule after activation) |
| Custom TCP | 111 | `gateway-marwa-client-sg` | NFS portmapper |
| Custom UDP | 111 | `gateway-marwa-client-sg` | NFS portmapper |
| NFS | TCP 2049 | `gateway-marwa-client-sg` | NFS |
| Custom UDP | 2049 | `gateway-marwa-client-sg` | NFS |
| Custom TCP | 20048 | `gateway-marwa-client-sg` | NFS mount daemon |
| Custom UDP | 20048 | `gateway-marwa-client-sg` | NFS mount daemon |

Outbound: leave the default (all traffic). The gateway needs outbound HTTPS to reach AWS.

**Action:** Create both groups. Create `gateway-marwa-client-sg` first so you can reference it.

**Expected Result:** Both groups exist; the gateway group has seven inbound rules, none with `0.0.0.0/0`.

**Why?** NFS is reachable only from your approved client group, never from the Internet. Port 80 from your IP is needed once, so the console (your browser) can fetch the activation key.

**Common Problems:** Activation later fails: port 80 rule missing or your IP changed. Mounting later fails: a protocol/port in the table is missing.

### Step 4.3: Launch the gateway appliance on EC2

**Goal:** Deploy the File Gateway appliance with a 150 GiB cache disk.

**Where:** AWS Console > Storage Gateway > Create gateway. (Alternatively open EC2 > Launch instances and search the AMI catalog for the AWS Storage Gateway AMI.)

**What to enter:**

- Gateway name: `file-gateway-marwa`
- Gateway type: **Amazon S3 File Gateway**
- Host platform: **Amazon EC2**
- Click **Launch instance** (the button opens the EC2 launch wizard with the correct gateway AMI preselected)
- Instance name: `gateway-marwa`
- Instance type: **the one your instructor approves** (the documented minimum is `m5.xlarge`: 4 vCPU, 16 GiB RAM)
- Key pair: your key pair or none
- Network: Default VPC; a subnet in `us-east-1a`; Auto-assign public IP **Enable**; security group `fgw-gateway-sg`
- Storage: keep the **root volume** (80 GiB is the AMI default) and click *Add new volume*: size **150 GiB**, type gp3, device `/dev/sdb` (this is the cache disk; use 150 GiB, not less)

**Action:** Launch instance. Wait for 2/2 status checks (a few minutes). Copy its **Public IPv4 address** and its **Private IPv4 address**.

**Expected Result:** Instance running with two EBS volumes attached (check the Storage tab).

**Why?** The task requires an appliance with a separate cache disk of at least 150 GiB. The cache holds recently written and read data.

**Common Problems:** `InstanceType not supported` or vCPU/quota error: ask the instructor for the approved size, or request a different type. Forgetting the second disk: stop the instance and attach a new 150 GiB volume in the same AZ (EC2 > Volumes > Create volume > Attach), then start it.

### Step 4.4: Launch the client instance

**Goal:** Create the instance that simulates the on-premises application server.

**Where:** EC2 > Launch instances.

**What to enter:** Name `gateway-marwa-client`; Amazon Linux 2023; `t3.micro`; Default VPC; same AZ/subnet as the gateway (`us-east-1a`); Auto-assign public IP Enable; security group `gateway-marwa-client-sg`.

**Action:** Launch. Write down the client's **Private IPv4 address** (for example `172.31.12.34`).

**Expected Result:** Running instance; private IP noted.

**Why?** The file share will later allow only this IP.

**Common Problems:** Different VPC from the gateway: NFS will time out. Stay in the Default VPC.

### Step 4.5: Activate the gateway

**Goal:** Connect the appliance to your AWS account.

**Where:** Storage Gateway console > Create gateway (continue the wizard from Step 4.3).

**What to enter:**

- *Set up gateway* page: name `file-gateway-marwa`, type Amazon S3 File Gateway, platform Amazon EC2 (already set)
- *Connect to AWS*: Service endpoint **Publicly accessible**
- Gateway connection options: **IP address**, enter the gateway's **public IPv4 address**

**Action:** Click *Next*, then *Activate gateway* (the console talks to the appliance on port 80 to get an activation key).

**Expected Result:** A *Configure gateway* page appears showing the local disks.

**Why?** Activation registers the appliance with your account and Region.

**Common Problems:** `Unable to connect to gateway`: appliance still booting (wait 3 minutes), security group port 80 not open to your *current* public IP, or the wrong IP entered. If your instructor requires a private-only gateway, they will tell you to use a VPC endpoint instead.

**Screenshot 13:** Activated gateway (after Step 4.6).

### Step 4.6: Allocate the cache disk

**Goal:** Configure the 150 GiB disk as cache.

**Where:** Still in the *Configure gateway* page. Later: Storage Gateway > Gateways > your gateway > Storage tab / Cache.

**What to enter:**

- CloudWatch log group: *Deactivate logging* (saves a little cost) or the default
- Cache: for the 150 GiB disk, choose **Cache**; leave the root disk unselected

**Action:** Click *Configure*.

**Expected Result:** Gateway status **Running**; the cache disk shows as *Cache* (allocated).

**Why?** Without allocated cache the gateway cannot create file shares.

**Common Problems:** No cache disk listed: the volume is not attached or the instance was not rebooted after attaching (reboot it and re-open the page). Disk smaller than 150 GiB: replace it.

**Screenshot 14:** Gateway Storage/Cache tab showing the allocated disk.

### Step 4.7: Remove the activation rule (optional hardening)

**Goal:** Close port 80 after activation.

**Where:** EC2 > Security Groups > `fgw-gateway-sg` > Inbound rules > Edit.

**Action:** Delete the HTTP 80 rule from My IP. Save.

**Expected Result:** Six inbound rules remain, all from `gateway-marwa-client-sg`.

**Why?** Least privilege: the gateway only needs inbound traffic from the client now. Do not delete it if you plan to re-activate.

**Common Problems:** None expected.

### Step 4.8: Create the NFS file share

**Goal:** Create a file share linked to your bucket, restricted to your client.

**Where:** Storage Gateway > File shares > Create file share.

**What to enter:**

- Gateway: `file-gateway-marwa`
- File share protocol: **NFS** (SMB only if your instructor approves)
- Amazon S3 bucket name: `file-gateway-marwa`
- S3 prefix: leave empty
- AWS Region: `us-east-1`
- Access to your S3 bucket / IAM role: choose **Use an existing IAM role** and enter the instructor-provided role (do not let the wizard create roles, since you cannot create IAM)
- Storage class: S3 Standard
- Additional settings, **Allowed clients**: replace `0.0.0.0/0` with your client's private IP as `/32`, for example `172.31.12.34/32`
- Squash level / file permissions: defaults

**Action:** Click *Create file share*. Wait for status **Available**.

**Expected Result:** A file share with a mount path like `GATEWAY_PRIVATE_IP:/file-gateway-marwa`, status Available.

**Why?** The IAM role lets the gateway write objects to your bucket; the allowed-client list is what restricts who can mount the share, and nothing is exposed publicly.

**Common Problems:** Stuck *Creating*: the IAM role is wrong (it must trust the Storage Gateway service and allow access to your bucket). Left allowed clients at `0.0.0.0/0`: edit the file share now and restrict it. `Bucket does not exist`: check the Region.

**Screenshot 15:** File share details with the bucket, IAM role and allowed client.

### Step 4.9: Mount the share on the client

**Goal:** Mount at `/mnt/cloudmart-share`.

**Where:** EC2 > `gateway-marwa-client` > Connect > EC2 Instance Connect.

**What to enter:** Replace `GATEWAY_PRIVATE_IP` with the gateway instance's private IP, and use your bucket name.

```
sudo dnf install -y nfs-utils
sudo mkdir -p /mnt/cloudmart-share
sudo mount -t nfs -o nolock,hard GATEWAY_PRIVATE_IP:/file-gateway-marwa /mnt/cloudmart-share
df -hT /mnt/cloudmart-share
mount | grep cloudmart-share
```

**Expected Result:** The share appears as type `nfs4` (or `nfs`) mounted at `/mnt/cloudmart-share`.

**Why?** The client uses a normal NFS client, which is the whole point: applications keep their usual file protocol.

**Common Problems:** `Connection timed out` or `access denied by server`: allowed-client IP is wrong, a security group port is missing, or the client IP changed after a restart (re-check it). `mount: wrong fs type`: `nfs-utils` not installed.

**Screenshot 16:** Mount output on the client.

### Step 4.10: Create test-file.txt on the client and find it in S3

**Goal:** Prove writes through the share reach S3.

**Where:** Client terminal, then S3 console.

**What to enter:**

```
echo "File Gateway test from client on $(date)" | sudo tee /mnt/cloudmart-share/test-file.txt
sudo cat /mnt/cloudmart-share/test-file.txt
```

Wait 1 to 5 minutes. Open S3 > `file-gateway-marwa` > Objects > refresh.

**Expected Result:** Object `test-file.txt` appears with the same text.

**Why?** The gateway holds the file in its local cache first, then uploads asynchronously.

**Common Problems:** Not visible yet: wait a few more minutes. Permission denied writing: use `sudo`, or check share squash and permission settings. Upload errors: re-check the IAM role.

**Screenshot 17a:** Terminal and S3 object.

### Step 4.11: Upload a different file directly to S3

**Goal:** Create an object the gateway does not yet know about.

**Where:** CloudShell or S3 console.

**What to enter:** `echo "Uploaded straight to S3" > from-s3-direct.txt && aws s3 cp from-s3-direct.txt s3://file-gateway-marwa/from-s3-direct.txt --region us-east-1`

**Expected Result:** The object is in the bucket. On the client, `ls /mnt/cloudmart-share` still lists only `test-file.txt` (the gateway has not refreshed).

**Why?** Shows that out-of-band changes do not appear automatically.

**Common Problems:** None expected. If the file already appears, the cache refreshed on its own, which is fine.

### Step 4.12: Refresh the cache and read the file

**Goal:** Make the gateway re-read the bucket listing.

**Where:** Storage Gateway > File shares > select your file share > **Actions > Refresh cache**.

**What to enter:** Defaults (refresh the root `/`), confirm.

**Action:** Wait for the refresh to finish (seconds to minutes). On the client run:

```
ls -l /mnt/cloudmart-share
cat /mnt/cloudmart-share/from-s3-direct.txt
```

**Expected Result:** `from-s3-direct.txt` is listed and its content is `Uploaded straight to S3`.

**Why?** Refresh Cache updates the gateway's view of the bucket. It does not download all data; objects are fetched when you read them.

**Common Problems:** File not shown: run `ls` again; if still missing, confirm the filename and bucket, and that the refresh completed (a notification appears in the console). If a mounted client caches directory data, remount or wait a moment.

**Screenshot 17b:** Refresh cache action and the file visible on the client.

### Step 4.13: Capture healthy gateway status

**Goal:** Evidence that the gateway is healthy.

**Where:** Storage Gateway > Gateways > `file-gateway-marwa`.

**Action:** Look at *Status: Running*, and open the *Storage* tab for the cache disk. Optionally check the CloudWatch metric `CachePercentDirty` is 0 (no pending uploads).

**Expected Result:** Status Running; cache allocated; file share Available.

**Why?** Task deliverable. Before you ever shut the gateway down, confirm nothing is waiting to upload.

**Common Problems:** Status Offline: instance stopped or has no outbound internet.

**Screenshot 18:** Gateway details showing Running.

**Part 4 Checkpoint:** gateway Running with cache disk; NFS share Available and restricted to the client; test-file.txt visible in S3; direct S3 upload visible after Refresh Cache.

---

# 7. Verification / Testing

Run these tests after each Part (or together before the demo).

### Part 1 (EFS)

| # | Test | Expected result | How to verify | If it fails |
| --- | --- | --- | --- | --- |
| 1.1 | EFS is Regional and encrypted | Type Regional, Encryption Enabled | EFS console General tab. CLI: `aws efs describe-file-systems --region us-east-1 --query 'FileSystems[].[Name,Encrypted,AvailabilityZoneName]'` (AZ name is empty/None for Regional) | Recreate with Regional + encryption (encryption cannot be changed later) |
| 1.2 | Mount targets in both AZs | Two targets, state available, different AZs | EFS > Network tab. CLI: `aws efs describe-mount-targets --file-system-id fs-0090781e61cc01486 --region us-east-1` | Add the missing mount target (Network > Manage) |
| 1.3 | Both instances mounted the same FS | Same `fs-` ID, TLS | `df -hT /mnt/efs` and `mount \| grep efs` on both | Re-run the mount; check security groups |
| 1.4 | Two-way sharing | Both files readable on both nodes | `cat` the files, `md5sum /mnt/efs/*.txt` equal on both | Check permissions and that both nodes use the same mount |
| 1.5 | NFS restricted | Only 2049 from `marwa-sg-4` | EC2 > Security Groups > `marwa-sg-4` inbound rules | Remove any other rules |
| 1.6 | OPTIONAL negative test | An instance outside `marwa-sg-4` cannot mount (timeout) | Launch a throwaway instance with the default SG, try the mount | If it works, `marwa-sg-4` is too open |

### Part 2 (S3 Files)

| # | Test | Expected result | How to verify | If it fails |
| --- | --- | --- | --- | --- |
| 2.1 | Versioning and SSE-S3 | Enabled | Console Properties. CLI: `aws s3api get-bucket-versioning --bucket BUCKET` and `aws s3api get-bucket-encryption --bucket BUCKET` | Enable them (never disable versioning later) |
| 2.2 | File system and mount target | Available | S3 Files console; `aws s3files list-mount-targets --file-system-id fs-0090781e61cc01486 --region us-east-1` | Wait or recreate |
| 2.3 | Mount works | `df` shows mount | `df -hT /mnt/s3files` | See Troubleshooting |
| 2.4 | File to S3 | `from-ec2.txt` object with same text | S3 console; `aws s3 cp s3://BUCKET/from-ec2.txt -` | Wait longer; check roles |
| 2.5 | S3 to file | `from-s3.txt` readable | `cat /mnt/s3files/from-s3.txt` | Wait and retry `ls` |

### Part 3 (CRR)

| # | Test | Expected result | How to verify | If it fails |
| --- | --- | --- | --- | --- |
| 3.1 | Different Regions | us-east-1 vs us-west-2 | S3 bucket list | Recreate the destination |
| 3.2 | Versioning both | Enabled | `aws s3api get-bucket-versioning --bucket NAME --region REGION` | Enable |
| 3.3 | Rule enabled with role | Status Enabled, role shown | Management > Replication rules; `aws s3api get-bucket-replication --bucket marwa-source-2026 --region us-east-1` | Edit the rule |
| 3.4 | Both objects at destination | Keys `product-image-01.png` and `images/product-image-02.png` | `aws s3 ls s3://marwa-destination-2026/ --recursive --region us-west-2` | Wait up to 15 min; check status |
| 3.5 | Status COMPLETED | Source `COMPLETED`, destination `REPLICA` | `aws s3api head-object --bucket marwa-source-2026 --key product-image-01.png --region us-east-1 --query ReplicationStatus` | If FAILED, check the role permissions with the instructor |
| 3.6 | No manual upload | You never uploaded to the destination | Destination object shows `REPLICA` status | Delete the manual object and re-upload to source |

### Part 4 (File Gateway)

| # | Test | Expected result | How to verify | If it fails |
| --- | --- | --- | --- | --- |
| 4.1 | Gateway healthy | Running | Storage Gateway > Gateways | Check instance and outbound internet |
| 4.2 | Cache configured | 150 GiB disk allocated as Cache | Gateway > Storage tab | Allocate the disk |
| 4.3 | Share linked correctly | Bucket name matches, Available | File shares details | Recreate the share |
| 4.4 | Authorized client mounts | Mount succeeds | `df -hT /mnt/cloudmart-share` | Check allowed-client IP, ports |
| 4.5 | Client file in S3 | `test-file.txt` with same content | S3 console; `aws s3 cp s3://BUCKET/test-file.txt -` | Wait; check IAM role |
| 4.6 | Direct S3 object after refresh | `from-s3-direct.txt` visible | Refresh Cache then `ls` | Re-run refresh |
| 4.7 | Not public | Gateway SG has no `0.0.0.0/0`; allowed clients is a /32 | Security group and file share settings | Remove open rules |

---

# 8. Demo for Presentation

To fit in 5 minutes, prepare everything beforehand (Parts 1 to 4 built, terminals open, tabs open). During the demo you only *perform the tests*, you do not build. Open these browser tabs in advance: (1) EFS console, (2) two terminals (efs-marwa-1, efs-marwa-2) side by side, (3) S3 source bucket, (4) S3 DR bucket in another window, (5) Storage Gateway console, (6) client terminal.

**Demo A: EFS shared storage (about 1 minute)**

1. Show the EFS console: Regional, Encrypted, two mount targets in different AZs.
2. In terminal 1 run `echo "live demo $(date)" > /mnt/efs/demo.txt`.
3. In terminal 2 run `cat /mnt/efs/demo.txt`. The same text appears.
4. Say: *"Two servers in different data centers, one shared disk. This is how a load-balanced web app shares user uploads."*

**Demo B: S3 Files (about 1 minute)**

1. Show the bucket (versioning on, SSE-S3, public access blocked).
2. On the EC2 client run `echo hi > /mnt/s3files/demo-live.txt`. Refresh the bucket after a minute and open the object. (If time is short, show the earlier pre-synced files, and upload one in the console to show it appear in `ls`.)
3. Say: *"The application only sees a folder. S3 keeps the data and syncs in both directions, a bit asynchronously."*

**Demo C: Cross-Region Replication (about 1.5 minutes)**

1. Show the Replication rule (Enabled, role).
2. In CloudShell upload a new file: `aws s3 cp product-image-01.png s3://marwa-source-2026/images/demo-live.png`. (Pre-uploading a second file before the demo avoids waiting.)
3. Show the source object with Replication status COMPLETED and the same key in the DR bucket.
4. Say: *"I never touched the Frankfurt bucket. S3 copied it itself, so if Ireland is unavailable we still have the data."*

**Demo D: File Gateway (about 1.5 minutes)**

1. Show Storage Gateway: Running, cache disk, NFS share limited to one client.
2. On the client run `echo gateway > /mnt/cloudmart-share/live.txt`; show it later in the S3 bucket. (Pre-create one file so it is already there.)
3. Upload a file in the S3 console, click *Actions > Refresh cache*, run `ls` on the client.
4. Say: *"Old on-premises apps keep using NFS. The data lives in S3 with a local cache for speed."*

**Tips:** Because syncing and replication are asynchronous, do the slow actions right at the start and talk about the architecture while they finish. Keep a backup of your screenshots in case of a delay.

---

# 9. What I Should Explain During the Presentation

**Architecture:** "I built four storage patterns. EFS is a shared network disk for servers. S3 Files lets servers see an S3 bucket as a folder. Cross-Region Replication copies S3 data to another Region. File Gateway lets on-premises-style NFS apps store data in S3."

**Each service**

- **EFS:** managed, elastic NFS that many Linux servers can mount at once across AZs.
- **S3 Files:** a file-system view of an S3 bucket, so apps do not need the S3 API.
- **S3 plus CRR:** durable object storage; replication copies each new object to a second Region.
- **Storage Gateway (File Gateway):** a local file share with a cache that uploads to S3.
- **EC2 and security groups:** the clients, and the firewall that decides who can use the storage.

**Data flow:** "EFS: client > mount target > EFS. S3 Files: client > mount target > sync > S3. CRR: upload > source > replication > destination. Gateway: client > gateway cache > asynchronous upload > S3."

**Security:** "Everything is private. Encryption at rest is on for EFS and S3 (SSE-S3). EFS is mounted with TLS in transit. NFS port 2049 is opened only to the client security group, not the Internet. S3 Block Public Access is on. We used instructor-provided IAM roles with least privilege. The gateway share only allows one client IP."

**Scalability:** "EFS grows and shrinks automatically. S3 is practically unlimited. Replication and S3 Files scale with the number of objects. A real File Gateway can be sized up by choosing a larger appliance and cache."

**Availability:** "EFS stores data across AZs and has mount targets in two. S3 stores across AZs. CRR adds a second Region for disaster recovery. Replication is not a full backup: it also copies mistakes, and failover must be planned separately."

**Cost:** "EFS and S3 Files charge for stored data, so tiny test files cost cents. CRR costs storage twice plus a small transfer fee. The gateway EC2 instance is the main cost, so we delete it right after the demo. Resources are small and temporary."

---

# 10. Troubleshooting

| Problem | Possible cause | Solution |
| --- | --- | --- |
| `mount.nfs4: Connection timed out` (EFS or S3 Files) | Security group does not allow 2049, or the mount target uses the default SG, or no mount target in the instance's AZ | Check inbound on the mount target SG from the client SG; fix the SG on the mount target; launch the instance in an AZ with a mount target |
| `mount: unknown filesystem type 'efs'` | `amazon-efs-utils` not installed | `sudo dnf install -y amazon-efs-utils` |
| `mount.s3files: command not found` / unknown type `s3files` | efs-utils older than 3.0.0 | Run the official installer script (Step 2.7) |
| `Failed to resolve fs-...` | Wrong Region or DNS resolution disabled in the VPC | Use the correct Region; enable DNS hostnames and resolution on the VPC |
| `Permission denied` writing to the mount | Root-owned directory | `sudo chown ec2-user:ec2-user /mnt/efs` (Part 1) or use `sudo` |
| S3 Files access denied or failure creating the file system | Missing IAM permission or role not passed | Use the instructor-prepared file system and role |
| S3 object does not appear after writing through S3 Files | Sync is asynchronous, file still open, or role issue | Close the file, wait several minutes, check roles |
| Replication rule save fails: versioning | Versioning disabled on a bucket | Enable it on both buckets |
| Replication status FAILED or nothing replicates | Role lacks permissions or trust; rule disabled; object uploaded before the rule existed | Ask the instructor to check the role; confirm Enabled; re-upload a new object |
| Destination shows no `images/` folder | Not replicated yet | Wait; check status on the source object |
| Gateway activation fails | Port 80 not open to your IP, wrong IP, instance still booting | Fix SG rule, check the IP, wait a few minutes |
| No cache disk available | Second volume missing or not attached | Attach a 150 GiB volume in the same AZ, reboot, refresh |
| File share stuck in Creating | Wrong IAM role or bucket in a different Region | Verify the role and bucket; recreate the share |
| Client cannot mount the gateway share | Allowed client IP wrong, SG ports missing, `nfs-utils` missing | Fix the /32 IP, add ports 111/2049/20048 from the client SG, install `nfs-utils` |
| New S3 file not visible through the share | Cache not refreshed | Actions > Refresh cache; run `ls` again |
| EC2 Instance Connect cannot connect | Port 22 rule, public IP changed, no public IP | Update the SG rule to My IP; give the instance a public IP |
| Bucket name already exists | Names are global | Add digits to your suffix |

---

# 11. Cleanup

Do this only **after** your instructor approves your work. Before shutting the gateway down, confirm CloudWatch `CachePercentDirty` is 0 so no data is lost.

**Order matters** (delete dependents first):

1. **Part 4 (most expensive first):**
   - Storage Gateway > File shares > select > Actions > Delete file share.
   - Storage Gateway > Gateways > select > Actions > Delete gateway (removes the registration).
   - EC2 > terminate `gateway-marwa` and `gateway-marwa-client`. Check EC2 > Volumes: delete any remaining volumes (state *available*), including the 150 GiB cache disk.
   - S3 > empty `file-gateway-marwa` (Empty bucket), then Delete bucket.
2. **Part 3:**
   - Source bucket > Management > Replication rules > delete the rule.
   - Empty **both** buckets. With versioning on, use *Empty bucket*, which removes all versions and delete markers. Then delete both buckets (`marwa-source-2026` in `us-east-1`, `marwa-destination-2026` in `us-west-2`; switch Region to see the destination if needed).
3. **Part 2:**
   - On the EC2 client used for Part 2: `sudo umount /mnt/s3files`. Terminate that client only if it was created for this lab; `efs-marwa-1` is recorded as a previous machine tried with S3 Files.
   - S3 > File systems > delete the mount target(s) first, then delete the file system (only the one you created; do not delete the instructor's).
   - Empty and delete `marwa-s3-files-2026`.
4. **Part 1:**
   - On both instances: `sudo umount /mnt/efs`. Terminate `efs-marwa-1` and `efs-marwa-2`.
   - EFS > File systems > select `marwa-efs` > Delete (type the ID to confirm); mount targets are removed with it. Wait until it disappears.
5. **Network objects:** EC2 > Security Groups: delete `marwa-sg-4`, `s3files-marwa-sg`, `fgw-gateway-sg`, `gateway-marwa-client-sg`, then `marwa-sg-4` (groups that reference each other cannot be deleted until the referencing group is gone). Delete key pairs you created if you do not need them.
6. **Optional:** delete files in CloudShell storage (`rm *.png *.txt`).

**Verify you are no longer charged**

- EC2 > Instances: all show *Terminated*. EC2 > Volumes: empty. EC2 > Elastic IPs: none.
- EFS, S3 file systems, Storage Gateway: empty lists. S3: no `cloudmart-` buckets.
- Check the **Billing and Cost Management > Cost Explorer / Bills** page the next day (charges can appear with a delay). Optionally use the Billing home page to look for any active services.
- Check other Regions too (`us-west-2` for the DR bucket).

---

# 12. Final Checklist

**Part 1: EFS**

- [ ] `marwa-sg-4` and `marwa-sg-4` created (only 2049 from `marwa-sg-4`)
- [ ] EFS `marwa-efs` created: Regional, encrypted, mount targets in two AZs
- [ ] Two EC2 instances running in different AZs
- [ ] Mounted at `/mnt/efs` with TLS on both
- [ ] Files shared both ways, checksums match
- [ ] Screenshots 1 to 4 captured

**Part 2: S3 Files**

- [ ] Bucket with Versioning, SSE-S3, Block Public Access
- [ ] S3 Files file system and mount target Available
- [ ] EC2 client with instructor role mounted at `/mnt/s3files`
- [ ] File to S3 and S3 to file tested
- [ ] Screenshots 5 to 8 captured

**Part 3: CRR**

- [ ] Source and DR buckets in different Regions, versioning on both
- [ ] Replication rule enabled with instructor role
- [ ] `product-image-01.png` and `images/product-image-02.png` replicated, COMPLETED
- [ ] Optional delete marker test done
- [ ] Screenshots 9 to 12 captured

**Part 4: File Gateway**

- [ ] Gateway activated and Running, 150 GiB cache allocated
- [ ] NFS share linked to the bucket, restricted to one client
- [ ] Mounted at `/mnt/cloudmart-share`; `test-file.txt` in S3
- [ ] Direct S3 upload visible after Refresh Cache
- [ ] Screenshots 13 to 18 captured

**Wrap up**

- [ ] AWS resources created
- [ ] Configuration completed
- [ ] Applications working
- [ ] Testing completed
- [ ] Demo ready
- [ ] Instructor approval received
- [ ] Resources deleted (all four Parts, both Regions)
- [ ] Billing checked the next day





