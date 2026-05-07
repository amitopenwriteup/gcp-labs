
## . Hands-On Labs

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
