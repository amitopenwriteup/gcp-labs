# Google Cloud Storage — Deep Dive Workshop
### A Comprehensive Lab Guide | UI-Based | No gcloud Commands

---

## Table of Contents

1. [Cloud Storage Deep Dive](#1-cloud-storage-deep-dive)
 - 1.1 [Storage Classes & Lifecycle Management](#11-storage-classes--lifecycle-management)
 - 1.2 [Bucket Configuration & Access Control](#12-bucket-configuration--access-control)
 - 1.3 [Transfer Services & Migration Tools](#13-transfer-services--migration-tools)
2. [Persistent Disks & Filestore](#2-persistent-disks--filestore)
 - 2.1 [Disk Types & Performance Characteristics](#21-disk-types--performance-characteristics)
 - 2.2 [Snapshot & Backup Strategies](#22-snapshot--backup-strategies)
 - 2.3 [Filestore for NFS Requirements](#23-filestore-for-nfs-requirements)
3. [Storage Best Practices](#3-storage-best-practices)
 - 3.1 [Data Organization & Naming Conventions](#31-data-organization--naming-conventions)
 - 3.2 [Security & Access Patterns](#32-security--access-patterns)
 - 3.3 [Cost Optimization Strategies](#33-cost-optimization-strategies)
4. [Hands-On Labs](#4-hands-on-labs)
 - Lab 1: [Static Website Hosting with Cloud Storage](#-lab-1-static-website-hosting-with-cloud-storage)
 - Lab 2: [Lifecycle Policy for Log Archive Automation](#-lab-2-lifecycle-policy-for-log-archive-automation)
 - Lab 3: [Persistent Disk Setup for a VM Workload](#-lab-3-persistent-disk-setup-for-a-vm-workload)
 - Lab 4: [Filestore Share for Multi-VM NFS Access](#-lab-4-filestore-share-for-multi-vm-nfs-access)
 - Lab 5: [Cross-Region Data Migration with Transfer Service](#-lab-5-cross-region-data-migration-with-transfer-service)

---

## 1. Cloud Storage Deep Dive

### 1.1 Storage Classes & Lifecycle Management

#### What is Cloud Storage?

Google Cloud Storage (GCS) is a globally unified, scalable, and durable object storage service. Objects are stored in **buckets** — containers tied to a project and a geographic location.

---

#### Storage Classes Explained

| Storage Class | Min Storage Duration | Access Frequency | Use Case |
|---|---|---|---|
| **Standard** | None | Frequent | Active data, websites, mobile apps |
| **Nearline** | 30 days | ~Once/month | Backups, data accessed monthly |
| **Coldline** | 90 days | ~Once/quarter | Disaster recovery archives |
| **Archive** | 365 days | ~Once/year | Long-term compliance archiving |

> **Key Insight:** You pay less per GB stored as you move down the classes, but retrieval costs increase. Choose based on how often you need to read data.

---

#### Object Lifecycle Management

Lifecycle management lets you automatically transition objects between storage classes or delete them based on rules — all configured from the Console UI.

**Rules you can define:**

| Condition | Description |
|---|---|
| `Age` | Number of days since the object was created |
| `CreatedBefore` | Objects created before a specific date |
| `IsLive` | Only applies to versioned objects |
| `NumberOfNewerVersions` | Keeps only N newest versions |
| `MatchesStorageClass` | Targets objects of a specific class |

**Available Actions:**

- `SetStorageClass` — Moves object to a different class (e.g., Standard → Coldline)
- `Delete` — Permanently removes the object
- `AbortIncompleteMultipartUpload` — Cleans up failed uploads

---

#### Lab Preview — Lifecycle UI Walk-Through

**Navigation Path:** 
`Google Cloud Console` → `Cloud Storage` → `Buckets` → *(Select your bucket)* → `Lifecycle` tab → `Add a rule`

**Rule creation wizard steps:**
1. Select **action** (Set Storage Class / Delete)
2. Define **conditions** (age, storage class, etc.)
3. Review and **Save** the rule

---

### 1.2 Bucket Configuration & Access Control

#### Creating a Bucket — UI Steps

**Navigation Path:** 
`Cloud Storage` → `Buckets` → `Create`

| Setting | Options | Recommendation |
|---|---|---|
| **Name** | Globally unique | Use `project-env-purpose` pattern |
| **Location type** | Region / Dual-region / Multi-region | Multi-region for global HA |
| **Default storage class** | Standard / Nearline / Coldline / Archive | Standard for active workloads |
| **Access control** | Uniform / Fine-grained | Uniform (recommended) |
| **Protection tools** | Retention policy, Object versioning | Enable for compliance |
| **Encryption** | Google-managed / Customer-managed (CMEK) | CMEK for regulated data |

---

#### Access Control Models

**Uniform Access Control (Recommended)**
- Applies IAM permissions uniformly to all objects
- Disables per-object ACLs
- Best practice for most workloads

**Fine-Grained Access Control**
- Allows per-object ACLs in addition to IAM
- Useful for legacy migrations
- Harder to audit at scale

---

#### IAM Roles for Cloud Storage

| Role | Description |
|---|---|
| `Storage Admin` | Full control over buckets and objects |
| `Storage Object Admin` | Manage objects (not bucket configuration) |
| `Storage Object Creator` | Upload objects only |
| `Storage Object Viewer` | Read-only access to objects |
| `Storage Legacy Bucket Owner` | Legacy ACL-based full access |

**Navigation Path for IAM:** 
`Cloud Storage` → `Buckets` → *(Select bucket)* → `Permissions` tab → `Grant Access`

---

#### Additional Bucket Security Controls

**Public Access Prevention:** 
`Bucket Settings` → `Permissions` → `Public access prevention` → Set to **Enforced**

**Retention Policies:** 
`Bucket Settings` → `Protection` → `Retention policy` → Set minimum retention period 
> Once set, objects **cannot be deleted** until the retention period expires.

**Object Versioning:** 
`Bucket Settings` → `Protection` → `Object versioning` → **Enable** 
> Keeps previous versions of objects on overwrite/delete. Essential for accidental deletion recovery.

---

### 1.3 Transfer Services & Migration Tools

#### Storage Transfer Service

Used to move data **into GCS** from external sources on a scheduled or one-time basis.

**Supported Sources:**

| Source Type | Description |
|---|---|
| Amazon S3 | Cross-cloud migration from AWS |
| Azure Blob Storage | Cross-cloud migration from Azure |
| HTTP/HTTPS URLs | Batch download from public endpoints |
| Another GCS bucket | Intra-GCS copies or replication |
| POSIX filesystem | On-prem server via agent |

**Navigation Path:** 
`Cloud Console` → `Storage Transfer Service` → `Create Transfer Job`

**Transfer Job Configuration Steps:**

1. **Select source** — Choose source type and provide credentials/URL
2. **Select destination** — Choose target GCS bucket
3. **Configure options:**
 - Overwrite behavior (never / if different / always)
 - File deletion (source or destination)
 - Include/exclude filters by prefix or date
4. **Schedule** — One time, recurring (daily/weekly), or run now
5. **Review and Create**

---

#### Transfer Appliance

For very large datasets (hundreds of TB to petabytes) where network transfer is impractical.

**Process:**
1. Request appliance via Cloud Console → `Storage Transfer Service` → `Transfer Appliance`
2. Google ships the physical device to your datacenter
3. Load your data onto the appliance
4. Ship it back to Google — data is ingested into GCS

---

#### gsutil vs. Storage Transfer Service

| Factor | Storage Transfer Service (UI) | Transfer Appliance |
|---|---|---|
| Data size | Up to hundreds of TB | Hundreds of TB to PB |
| Speed | Network-dependent | Physical shipment |
| Schedule | Automated recurring jobs | One-time |
| Cost | Egress + operation fees | Appliance rental fee |

---

## 2. Persistent Disks & Filestore

### 2.1 Disk Types & Performance Characteristics

#### Persistent Disk Overview

Persistent Disks (PD) are block storage devices that attach to Compute Engine VMs. Unlike local SSDs, they persist independently of the VM lifecycle.

---

#### Disk Type Comparison

| Disk Type | Technology | Max IOPS (per GB) | Throughput | Best For |
|---|---|---|---|---|
| **Standard (pd-standard)** | HDD | 0.75 read / 1.5 write | Low | Cold data, backups |
| **Balanced (pd-balanced)** | SSD | 6 read / 6 write | Medium | General workloads |
| **SSD (pd-ssd)** | SSD | 30 read / 30 write | High | Databases, high IOPS |
| **Extreme (pd-extreme)** | SSD | Provisioned (up to 120k) | Very High | SAP HANA, high-perf DB |
| **Hyperdisk Balanced** | Next-gen | Provisioned | Flexible | Modern general workloads |
| **Hyperdisk Extreme** | Next-gen | Provisioned (up to 350k) | Very High | Mission-critical DB |

> **Performance Scales with Disk Size** — IOPS and throughput scale linearly with disk size up to the instance limit.

---

#### Creating a Persistent Disk (UI)

**Navigation Path:** 
`Compute Engine` → `Disks` → `Create Disk`

| Field | Description |
|---|---|
| Name | Descriptive identifier |
| Type | Standard / Balanced / SSD / Extreme |
| Region & Zone | Must match the target VM's zone |
| Size (GB) | Determines performance ceiling |
| Snapshot schedule | Attach a backup schedule at creation |
| Encryption | Google / Customer / CMEK |

---

#### Attaching a Disk to a VM (UI)

**Navigation Path:** 
`Compute Engine` → `VM Instances` → *(Select VM)* → `Edit` → `Additional Disks` → `Attach existing disk`

**Attach modes:**

| Mode | Description |
|---|---|
| Read/Write | Default — single VM can write |
| Read-only | Multiple VMs can mount simultaneously |

---

### 2.2 Snapshot & Backup Strategies

#### Snapshots Overview

Snapshots capture the **state of a Persistent Disk** at a point in time. They are:
- **Incremental** after the first snapshot
- **Stored in Cloud Storage** (managed internally)
- **Usable across projects** with correct IAM

---

#### Creating a Manual Snapshot (UI)

**Navigation Path:** 
`Compute Engine` → `Disks` → *(Select disk)* → `Create Snapshot`

| Field | Notes |
|---|---|
| Name | Use date/version convention |
| Description | Document purpose |
| Location | Multi-regional for DR |
| Storage location type | Regional / Multi-regional |
| Labels | Add environment/team/cost-center labels |

---

#### Snapshot Schedules (Automated)

**Navigation Path:** 
`Compute Engine` → `Resource Policies` → `Create Resource Policy` → `Snapshot Schedule`

**Schedule Configuration:**

| Setting | Options |
|---|---|
| Frequency | Hourly / Daily / Weekly |
| Start time | UTC time of first snapshot |
| Retention | Number of days to keep snapshots |
| Storage location | Regional / Multi-regional |

**Attach schedule to disk:** 
`Compute Engine` → `Disks` → *(Select disk)* → `Edit` → `Snapshot Schedule` → Select policy

---

#### Snapshot Best Practices

- Use **multi-regional storage** for disaster recovery scenarios
- Label snapshots with `env`, `team`, and `date` for cost attribution
- Detach database connections or use **application-consistent snapshots** for live DB disks
- Set retention to balance recovery needs vs. cost
- Test snapshot **restore** procedures quarterly

---

### 2.3 Filestore for NFS Requirements

#### What is Filestore?

Google Cloud Filestore is a fully managed **NFS file server** service. It provides shared file storage accessible by multiple Compute Engine VMs or GKE nodes simultaneously — ideal for:

- Shared code and configuration directories
- Content management system (CMS) media libraries
- HPC and rendering farm shared scratch
- Enterprise apps requiring POSIX-compliant storage

---

#### Filestore Tiers

| Tier | Use Case | Capacity Range | Performance |
|---|---|---|---|
| **Basic HDD** | Dev/test, low-perf | 1 TB – 63.9 TB | 100 MB/s |
| **Basic SSD** | General production | 2.5 TB – 63.9 TB | 1.2 GB/s |
| **Zonal (formerly High Scale)** | Workloads needing scale | 10 TB – 100 TB | Up to 4.8 GB/s |
| **Enterprise** | Mission-critical, HA | 1 TB – 10 TB | 800 MB/s with HA |

---

#### Creating a Filestore Instance (UI)

**Navigation Path:** 
`Cloud Console` → `Filestore` → `Instances` → `Create Instance`

| Field | Notes |
|---|---|
| Instance ID | Descriptive name |
| Tier | Choose based on IOPS/throughput needs |
| Region & Zone | Co-locate with consuming VMs |
| Network | Shared VPC or project VPC |
| IP Range | Reserve a /29 CIDR for NFS mount points |
| File share name | Name of the exported NFS share |
| Capacity | Set size in TB |
| Access controls | IP-based or VPC restrictions |

---

#### Connecting to Filestore from a VM

> All steps performed inside the VM via SSH — no gcloud required once the instance is created via UI.

1. **Get NFS IP** from Filestore Instance page → `Connection Details`
2. **Install NFS client** on your VM OS (e.g., `nfs-common` on Debian/Ubuntu via OS package manager)
3. **Create a mount point directory** (e.g., `/mnt/shared`)
4. **Mount the share** using the NFS IP and share name shown in the Console
5. **Verify** with a write test

---

## 3. Storage Best Practices

### 3.1 Data Organization & Naming Conventions

#### Bucket Naming Strategy

| Pattern | Example | Purpose |
|---|---|---|
| `{project}-{env}-{purpose}` | `myapp-prod-assets` | Application asset buckets |
| `{team}-{region}-{datatype}` | `analytics-us-rawlogs` | Team-scoped data |
| `{company}-backup-{year}` | `acme-backup-2025` | Archive buckets |

**Bucket Naming Rules:**
- 3–63 characters
- Lowercase letters, numbers, and hyphens only
- Must start/end with a letter or number
- Cannot contain "google" or be an IP address

---

#### Object Prefix (Folder) Strategy

GCS uses **flat namespace** — "folders" are just prefixes in object names.

**Recommended prefix structure:**
```
{bucket}/
 {environment}/
 {year}/
 {month}/
 {day}/
 {object-name}
```

**Example:**
```
myapp-prod-assets/
 logs/
 2025/
 06/
 15/
 app-server-1.log.gz
```

> Avoid using sequential prefixes (001, 002…) — GCS distributes load by prefix hash, so similar prefixes may hit the same shard and reduce throughput.

---

### 3.2 Security & Access Patterns

#### Least Privilege IAM

| Principle | Implementation |
|---|---|
| Avoid `Storage Admin` for apps | Use `Storage Object Creator` for write-only apps |
| Separate read from write | Different service accounts per function |
| Use service accounts | Avoid user credentials for application access |
| Audit permissions quarterly | `IAM & Admin` → `Policy Analyzer` |

---

#### Signed URLs for Temporary Access

When you need to share a private object without granting IAM access:

**Navigation Path:** 
`Cloud Storage` → `Buckets` → *(Select bucket)* → *(Select object)* → `⋮ (kebab menu)` → `Create signed URL`

| Option | Description |
|---|---|
| Expiration | Set time-to-live (minutes to 7 days) |
| HTTP method | GET (download) / PUT (upload) |
| Service account | Identity used to sign |

---

#### Data Encryption Options

| Option | Control Level | Use Case |
|---|---|---|
| **Google-managed** | Google controls keys | Default, low overhead |
| **Customer-managed (CMEK)** | You manage keys in Cloud KMS | Compliance requirements |
| **Customer-supplied (CSEK)** | You provide keys per-request | Highest control, manual key management |

**CMEK Configuration Path:** 
`Cloud KMS` → Create Key Ring & Key → Assign to bucket at creation 
`Bucket Creation` → `Advanced Settings` → `Encryption` → `Customer-managed key`

---

#### VPC Service Controls

For highly sensitive data, use **VPC Service Controls** to create a perimeter that prevents data exfiltration:

**Navigation Path:** 
`Security` → `VPC Service Controls` → `Create Perimeter` → Add `storage.googleapis.com` to restricted services

---

### 3.3 Cost Optimization Strategies

#### Storage Cost Levers

| Strategy | Savings Potential | How to Implement |
|---|---|---|
| Lifecycle policies to Coldline/Archive | 40–80% on archived data | `Bucket` → `Lifecycle` → Add age-based transition |
| Delete incomplete multipart uploads | Eliminates hidden waste | Lifecycle rule: `AbortIncompleteMultipartUpload` after 7 days |
| Remove old object versions | Reduces versioning overhead | Lifecycle rule: `Delete` when `NumberOfNewerVersions > 3` |
| Right-size storage class at upload | Avoid early deletion fees | Upload directly to Nearline/Coldline if access is rare |
| Enable requester-pays for shared buckets | Shifts egress cost to consumer | `Bucket Settings` → `Requester Pays` → Enable |

---

#### Cost Monitoring

**Navigation Path:** 
`Billing` → `Reports` → Filter by `Service: Cloud Storage` 
`Cloud Storage` → `Buckets` → `Monitoring` tab → Storage, request, and bandwidth metrics

**Recommended Labels for Cost Attribution:**

| Label Key | Example Values |
|---|---|
| `environment` | prod, staging, dev |
| `team` | analytics, backend, frontend |
| `cost-center` | cc-1234, cc-5678 |
| `data-classification` | public, internal, confidential |

---

## 4. Hands-On Labs

---

### Lab 1: Static Website Hosting with Cloud Storage

**Objective:** Host a static HTML website publicly using a Cloud Storage bucket.

**Estimated Time:** 30 minutes 
**Skill Level:** Beginner

---

#### Prerequisites
- Active GCP project with billing enabled
- Owner or Storage Admin IAM role

---

#### Step 1 — Create a Bucket for Website Hosting

1. Navigate to **Cloud Storage** → **Buckets** → **Create**
2. Set **bucket name** to match your domain (e.g., `www.example.com`) or use a unique name like `lab1-static-site-[yourname]`
3. **Location type:** Region → Select `us-central1`
4. **Default storage class:** Standard
5. **Access control:** Fine-grained *(required for public website)*
6. Leave other settings as default → Click **Create**

---

#### Step 2 — Upload Website Files

1. Click into your newly created bucket
2. Click **Upload Files**
3. Upload an `index.html` file with basic HTML content
4. Upload a custom `404.html` file for error handling
5. Optionally upload CSS, JS, and image assets

---

#### Step 3 — Make Objects Public

**Option A — Make all objects public (uniform):**
1. Go to **Permissions** tab → **Grant Access**
2. New principals: `allUsers`
3. Role: **Storage Object Viewer**
4. Click **Save** → Accept the public access warning

**Option B — Make individual objects public:**
1. Click the `⋮` menu next to `index.html`
2. Select **Edit access**
3. Add entity: **Public** → Permission: **Reader**
4. Repeat for each file

---

#### Step 4 — Configure Website Settings

1. Go to bucket → **Website Configuration** (found under `⋮` → `Edit website configuration` or bucket settings)
2. Set **Main page suffix:** `index.html`
3. Set **Not found page:** `404.html`
4. Click **Save**

---

#### Step 5 — Test Your Website

1. Copy the bucket's **public URL** from the bucket details page: 
 Format: `https://storage.googleapis.com/[BUCKET_NAME]/index.html`
2. Open in browser — your HTML page should render
3. Test 404 by visiting a non-existent path

---

#### Lab 1 Success Criteria
- [ ] Bucket created with fine-grained access
- [ ] `index.html` accessible publicly via browser
- [ ] Custom 404 page loads for invalid paths
- [ ] Website configuration saved with main page suffix

---

### Lab 2: Lifecycle Policy for Log Archive Automation

**Objective:** Automatically transition log files from Standard to Coldline after 30 days, then delete after 365 days.

**Estimated Time:** 20 minutes 
**Skill Level:** Beginner–Intermediate

---

#### Step 1 — Create a Log Archive Bucket

1. **Cloud Storage** → **Buckets** → **Create**
2. Name: `lab2-log-archive-[yourname]`
3. Location: `us-central1` (Region)
4. Storage class: **Standard**
5. Access control: **Uniform**
6. Click **Create**

---

#### Step 2 — Enable Object Versioning (Optional but Recommended)

1. Click into your bucket → **Protection** tab
2. Under **Object versioning** → Click **Enable**
3. Click **Confirm**

---

#### Step 3 — Upload Sample Log Files

1. In the **Objects** tab → **Upload Files**
2. Upload several `.log` or `.txt` files to simulate log data
3. Optionally create a folder structure: `logs/2025/06/`

---

#### Step 4 — Create Lifecycle Rule: Transition to Coldline

1. Go to **Lifecycle** tab → **Add a Rule**
2. **Action:** Set storage class to **Coldline**
3. **Conditions:**
 - Age: `30` days
 - Matches storage class: `Standard`
4. Click **Continue** → Review → **Create**

---

#### Step 5 — Create Lifecycle Rule: Delete After 1 Year

1. **Add another Rule**
2. **Action:** Delete object
3. **Conditions:**
 - Age: `365` days
4. Click **Continue** → Review → **Create**

---

#### Step 6 — Create Lifecycle Rule: Clean Up Incomplete Uploads

1. **Add another Rule**
2. **Action:** Abort incomplete multipart uploads
3. **Conditions:**
 - Age: `7` days
4. Click **Continue** → Review → **Create**

---

#### Step 7 — Review All Lifecycle Rules

Your **Lifecycle** tab should now show 3 rules:

| Rule | Action | Condition |
|---|---|---|
| Transition to Coldline | Set storage class: Coldline | Age ≥ 30 days, Class = Standard |
| Auto-delete | Delete object | Age ≥ 365 days |
| Cleanup incomplete uploads | Abort multipart | Age ≥ 7 days |

---

#### Lab 2 Success Criteria
- [ ] Bucket created with uniform access
- [ ] Object versioning enabled
- [ ] Three lifecycle rules active
- [ ] Rules visible in the Lifecycle tab

---

### Lab 3: Persistent Disk Setup for a VM Workload

**Objective:** Create a VM with a system disk, attach a secondary SSD data disk, and configure a snapshot schedule.

**Estimated Time:** 35 minutes 
**Skill Level:** Intermediate

---

#### Step 1 — Create a Compute Engine VM

1. **Compute Engine** → **VM Instances** → **Create Instance**
2. Configure:
 - Name: `lab3-disk-vm`
 - Region: `us-central1` / Zone: `us-central1-a`
 - Machine type: `e2-medium`
 - Boot disk: **Balanced persistent disk**, 20 GB, Debian 11
3. Click **Create**

---

#### Step 2 — Create a Secondary Persistent Disk

1. **Compute Engine** → **Disks** → **Create Disk**
2. Configure:
 - Name: `lab3-data-disk`
 - Type: **SSD persistent disk**
 - Region/Zone: `us-central1-a` *(must match VM zone)*
 - Size: `100 GB`
 - Encryption: Google-managed
3. Click **Create**

---

#### Step 3 — Attach the Disk to Your VM

1. **VM Instances** → Click `lab3-disk-vm` → **Edit**
2. Scroll to **Additional disks** → **Attach existing disk**
3. Select `lab3-data-disk`
4. Mode: **Read/Write**
5. Deletion rule: **Keep disk** *(preserves data if VM is deleted)*
6. Click **Save**

---

#### Step 4 — Verify Disk Attachment

1. Click the **VM name** → View **Storage** section in VM details
2. You should see both the boot disk and `lab3-data-disk` listed
3. Note the **device name** assigned (e.g., `persistent-disk-1`)

---

#### Step 5 — Create a Snapshot Schedule

1. **Compute Engine** → **Resource Policies** → **Create Resource Policy**
2. Choose: **Snapshot Schedule**
3. Configure:
 - Name: `lab3-daily-snapshot`
 - Region: `us-central1`
 - Frequency: **Daily**
 - Start time: `02:00 UTC` (off-peak)
 - Retention: `7` days
 - Storage location: **Multi-regional** → `us`
4. Click **Create**

---

#### Step 6 — Attach Schedule to the Data Disk

1. **Compute Engine** → **Disks** → Click `lab3-data-disk`
2. Click **Edit**
3. Under **Snapshot schedule** → Select `lab3-daily-snapshot`
4. Click **Save**

---

#### Step 7 — Take a Manual Snapshot

1. **Compute Engine** → **Disks** → `lab3-data-disk` → **Create Snapshot**
2. Name: `lab3-manual-snap-001`
3. Description: `Initial baseline snapshot before data load`
4. Storage location: **Multi-regional** → `us`
5. Click **Create**
6. Monitor status under **Compute Engine** → **Snapshots**

---

#### Lab 3 Success Criteria
- [ ] VM created with boot disk
- [ ] Secondary SSD disk created in the same zone
- [ ] Disk attached to VM in Read/Write mode
- [ ] Daily snapshot schedule created and attached to disk
- [ ] Manual snapshot visible under Snapshots with status = Ready

---

### Lab 4: Filestore Share for Multi-VM NFS Access

**Objective:** Create a Filestore instance and mount the NFS share on two VMs for shared file access.

**Estimated Time:** 45 minutes 
**Skill Level:** Intermediate

---

#### Step 1 — Create Two VMs in the Same Zone

Repeat VM creation twice:

**VM 1:**
- Name: `lab4-vm-1`
- Zone: `us-central1-a`
- Machine type: `e2-small`

**VM 2:**
- Name: `lab4-vm-2`
- Zone: `us-central1-a`
- Machine type: `e2-small`

> Both VMs must be in the same zone as the Filestore instance.

---

#### Step 2 — Create a Filestore Instance

1. **Cloud Console** → **Filestore** → **Instances** → **Create Instance**
2. Configure:
 - Instance ID: `lab4-nfs-share`
 - Tier: **Basic HDD** (sufficient for lab)
 - Region: `us-central1` / Zone: `us-central1-a`
 - Network: `default`
 - File share name: `shared_data`
 - Capacity: `1 TB` (minimum)
3. Click **Create**
4. Wait 3–5 minutes for provisioning

---

#### Step 3 — Note the Filestore IP Address

1. Click into `lab4-nfs-share`
2. Under **Connection details**, copy the **NFS mount point:** 
 Format: `[IP_ADDRESS]:/shared_data`
3. Keep this handy for the mount commands

---

#### Step 4 — Prepare VMs for NFS Mount

SSH into `lab4-vm-1` via Cloud Console:
1. **VM Instances** → `lab4-vm-1` → **SSH** (browser SSH)
2. Run these commands in the terminal:
```bash
sudo apt-get update
sudo apt-get install -y nfs-common
sudo mkdir -p /mnt/shared_data
sudo mount [FILESTORE_IP]:/shared_data /mnt/shared_data
df -h | grep shared_data
```

Repeat for `lab4-vm-2`.

---

#### Step 5 — Test Shared File Access

On **VM 1**, create a test file:
```bash
sudo touch /mnt/shared_data/hello_from_vm1.txt
ls /mnt/shared_data/
```

On **VM 2**, verify the file is visible:
```bash
ls /mnt/shared_data/
# Should show: hello_from_vm1.txt
```

---

#### Step 6 — Monitor Filestore Metrics

1. **Filestore** → `lab4-nfs-share` → **Monitoring** tab
2. Review:
 - Disk utilization
 - Read/Write throughput
 - Operations per second

---

#### Lab 4 Success Criteria
- [ ] Filestore instance created and in Ready state
- [ ] NFS share mounted on both VMs
- [ ] File written from VM1 is visible on VM2
- [ ] Filestore monitoring shows activity

---

### Lab 5: Cross-Region Data Migration with Transfer Service

**Objective:** Use Storage Transfer Service to copy data from one GCS bucket to another in a different region, with a recurring schedule.

**Estimated Time:** 30 minutes 
**Skill Level:** Intermediate–Advanced

---

#### Step 1 — Create Source and Destination Buckets

**Source Bucket:**
1. **Cloud Storage** → **Buckets** → **Create**
2. Name: `lab5-source-[yourname]`
3. Location: `us-central1` (Region)
4. Storage class: Standard
5. Access: Uniform

**Destination Bucket:**
1. **Create** another bucket
2. Name: `lab5-destination-[yourname]`
3. Location: `us-east1` (Region) — *different from source*
4. Storage class: Standard
5. Access: Uniform

---

#### Step 2 — Upload Sample Data to Source Bucket

1. Click into `lab5-source-[yourname]`
2. Create folder structure: `data/batch1/`
3. Upload several sample files (text, CSV, or JSON)

---

#### Step 3 — Create a Transfer Job

1. **Cloud Console** → **Storage Transfer Service** → **Create Transfer Job**
2. **Source:**
 - Source type: **Google Cloud Storage**
 - Bucket: `lab5-source-[yourname]`
 - Path filter: Leave blank (transfer all) or specify `data/`
3. **Destination:**
 - Bucket: `lab5-destination-[yourname]`
4. **Transfer options:**
 - When to overwrite: **If different** (checksum mismatch)
 - When to delete: **Never** (keep source intact)
 - Metadata options: Preserve ACLs and metadata
5. **Schedule:**
 - Run every day
 - Start date: Today
 - Repeat until: Select end date or leave ongoing
6. **Description:** `lab5-daily-cross-region-sync`
7. Click **Create**

---

#### Step 4 — Monitor the Transfer Job

1. **Storage Transfer Service** → **Jobs** → Click `lab5-daily-cross-region-sync`
2. Review:
 - **Status** — In Progress / Success / Failed
 - **Operations** tab — Per-run execution history
 - **Counters** — Files copied, bytes transferred, errors
3. Click **Run now** to trigger an immediate execution for testing

---

#### Step 5 — Verify Data in Destination Bucket

1. **Cloud Storage** → `lab5-destination-[yourname]`
2. Confirm all objects from source are present
3. Compare object counts between source and destination

---

#### Step 6 — Add a File Filter (Optional Enhancement)

1. Return to the transfer job → **Edit**
2. Under **Source** → **Filters**:
 - Include prefixes: `data/batch1/`
 - Exclude prefixes: `data/temp/`
 - Last modified time: Objects modified in **last 24 hours**
3. Save changes

---

#### Lab 5 Success Criteria
- [ ] Source and destination buckets created in different regions
- [ ] Sample data uploaded to source bucket
- [ ] Transfer job created with daily schedule
- [ ] At least one successful transfer operation visible
- [ ] All objects confirmed in destination bucket

---

## Reference Quick Guide

### Cloud Storage Pricing Snapshot (us-central1)

| Class | Storage/GB/mo | Retrieval/GB | Min Duration |
|---|---|---|---|
| Standard | $0.020 | $0.00 | None |
| Nearline | $0.010 | $0.01 | 30 days |
| Coldline | $0.004 | $0.02 | 90 days |
| Archive | $0.0012 | $0.05 | 365 days |

> Prices are approximate. Always verify current rates at [cloud.google.com/storage/pricing](https://cloud.google.com/storage/pricing)

---

### Persistent Disk Pricing Snapshot

| Type | Price/GB/mo |
|---|---|
| Standard | $0.040 |
| Balanced | $0.100 |
| SSD | $0.170 |
| Extreme | $0.125 + IOPS charge |

---

### Troubleshooting Common Issues

| Issue | Likely Cause | Resolution (UI Path) |
|---|---|---|
| 403 Forbidden on public object | Public access prevention enabled | `Bucket` → `Permissions` → Check public access setting |
| Lifecycle rules not executing | XML may have conflict; check rule order | `Lifecycle` tab → Review/delete conflicting rules |
| Snapshot job failing | Disk in use, permissions issue | `Snapshots` → Check error message; verify IAM |
| Filestore mount fails | Firewall blocking NFS port 2049 | `VPC` → `Firewall Rules` → Allow port 2049 from VM CIDR |
| Transfer job 0 files copied | Source bucket permissions not granted | Grant `Storage Object Viewer` to Transfer Service account |

---

*Workshop developed for Google Cloud Storage fundamentals training.* 
*All labs are designed for the Google Cloud Console UI — no CLI or automation required.* 
*Last updated: June 2025*
