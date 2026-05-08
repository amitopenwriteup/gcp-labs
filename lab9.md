# Hands-On Labs

---

## Lab 1: Static Website Hosting with Cloud Storage

**Objective:** Host a static HTML website publicly using a Cloud Storage bucket.

**Estimated Time:** 30 minutes  
**Skill Level:** Beginner

---

### Prerequisites
- Active GCP project with billing enabled
- Owner or Storage Admin IAM role

---

### Step 1 — Create a Bucket for Website Hosting

1. Navigate to **Cloud Storage** → **Buckets** → **Create**
2. Set **bucket name** to match your domain (e.g., `www.example.com`) or use a unique name like `lab1-static-site-[yourname]`
3. **Location type:** Region → Select `us-central1`
4. **Default storage class:** Standard
5. **Access control:** Fine-grained *(required for public website)*
6. Leave other settings as default → Click **Create**

---

### Step 2 — Upload Website Files

1. Click into your newly created bucket
2. Click **Upload Files**
3. Upload an `index.html` file with basic HTML content
4. Upload a custom `404.html` file for error handling
5. Optionally upload CSS, JS, and image assets

Use the following sample files to get started:

**index.html**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>My GCS Static Site</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      max-width: 800px;
      margin: 60px auto;
      text-align: center;
      color: #333;
    }
    h1 { color: #1a73e8; }
  </style>
</head>
<body>
  <h1>Welcome to My Static Website</h1>
  <p>This page is hosted on Google Cloud Storage.</p>
</body>
</html>
```

**404.html**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Page Not Found</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      max-width: 800px;
      margin: 60px auto;
      text-align: center;
      color: #333;
    }
    h1 { color: #d93025; }
    a { color: #1a73e8; text-decoration: none; }
    a:hover { text-decoration: underline; }
  </style>
</head>
<body>
  <h1>404 — Page Not Found</h1>
  <p>The page you're looking for doesn't exist.</p>
  <a href="/index.html">← Back to Home</a>
</body>
</html>
```

---

### Step 3 — Make Objects Public

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

### Step 4 — Configure Website Settings

1. Go to your bucket → click the **Settings** tab
2. Scroll down to the **Static website** section
3. Set **Main page suffix:** `index.html`
4. Set **Not found page:** `404.html`
5. Click **Save**

> **Note:** The Static website section is only visible when the bucket's access control is set to **Fine-grained**. If you don't see it, check **Permissions** tab and confirm the access control setting.

---

### Step 5 — Test Your Website

1. Copy the bucket's **public URL** from the bucket details page:  
   Format: `https://storage.googleapis.com/[BUCKET_NAME]/index.html`
2. Open in browser — your HTML page should render
3. Test 404 by visiting a non-existent path

---

### Lab 1 Success Criteria
- [ ] Bucket created with fine-grained access
- [ ] `index.html` accessible publicly via browser
- [ ] Custom 404 page loads for invalid paths
- [ ] Website configuration saved with main page suffix

---

## Lab 2: Lifecycle Policy for Log Archive Automation

**Objective:** Automatically transition log files from Standard to Coldline after 30 days, then delete after 365 days.

**Estimated Time:** 20 minutes  
**Skill Level:** Beginner–Intermediate

---

### Step 1 — Create a Log Archive Bucket

1. **Cloud Storage** → **Buckets** → **Create**
2. Name: `lab2-log-archive-[yourname]`
3. Location: `us-central1` (Region)
4. Storage class: **Standard**
5. Access control: **Uniform**
6. Click **Create**

---

### Step 2 — Enable Object Versioning (Optional but Recommended)

1. Click into your bucket → **Protection** tab
2. Under **Object versioning** → Click **Enable**
3. Click **Confirm**

---

### Step 3 — Upload Sample Log Files

1. In the **Objects** tab → **Upload Files**
2. Upload several `.log` or `.txt` files to simulate log data
3. Optionally create a folder structure: `logs/2025/06/`

---

### Step 4 — Create Lifecycle Rule: Transition to Coldline

1. Go to **Lifecycle** tab → **Add a Rule**
2. **Action:** Set storage class to **Coldline**
3. **Conditions:**
   - Age: `30` days
   - Matches storage class: `Standard`
4. Click **Continue** → Review → **Create**

---

### Step 5 — Create Lifecycle Rule: Delete After 1 Year

1. **Add another Rule**
2. **Action:** Delete object
3. **Conditions:**
   - Age: `365` days
4. Click **Continue** → Review → **Create**

---

### Step 6 — Create Lifecycle Rule: Clean Up Incomplete Uploads

1. **Add another Rule**
2. **Action:** Abort incomplete multipart uploads
3. **Conditions:**
   - Age: `7` days
4. Click **Continue** → Review → **Create**

---

### Step 7 — Review All Lifecycle Rules

Your **Lifecycle** tab should now show 3 rules:

| Rule | Action | Condition |
|---|---|---|
| Transition to Coldline | Set storage class: Coldline | Age ≥ 30 days, Class = Standard |
| Auto-delete | Delete object | Age ≥ 365 days |
| Cleanup incomplete uploads | Abort multipart | Age ≥ 7 days |

---

### Lab 2 Success Criteria
- [ ] Bucket created with uniform access
- [ ] Object versioning enabled
- [ ] Three lifecycle rules active
- [ ] Rules visible in the Lifecycle tab

---

## Lab 3: Persistent Disk Setup for a VM Workload

**Objective:** Create a VM with a system disk, attach a secondary SSD data disk, and configure a snapshot schedule.

**Estimated Time:** 35 minutes  
**Skill Level:** Intermediate

---

### Step 1 — Create a Compute Engine VM

1. **Compute Engine** → **VM Instances** → **Create Instance**
2. Configure:
   - Name: `lab3-disk-vm`
   - Region: `us-central1` / Zone: `us-central1-a`
   - Machine type: `e2-medium`
   - Boot disk: **Balanced persistent disk**, 20 GB, Debian 11
3. Click **Create**

---

### Step 2 — Create a Secondary Persistent Disk

1. **Compute Engine** → **Disks** → **Create Disk**
2. Configure:
   - Name: `lab3-data-disk`
   - Type: **SSD persistent disk**
   - Region/Zone: `us-central1-a` *(must match VM zone)*
   - Size: `10 GB`
   - Encryption: Google-managed
3. Click **Create**

---

### Step 3 — Attach the Disk to Your VM

1. **VM Instances** → Click `lab3-disk-vm` → **Edit**
2. Scroll to **Additional disks** → **Attach existing disk**
3. Select `lab3-data-disk`
4. Mode: **Read/Write**
5. Deletion rule: **Keep disk** *(preserves data if VM is deleted)*
6. Click **Save**

---

### Step 4 — Verify Disk Attachment

1. Click the **VM name** → View **Storage** section in VM details
2. You should see both the boot disk and `lab3-data-disk` listed
3. Note the **device name** assigned (e.g., `persistent-disk-1`)

---

### Step 5 — Create a Snapshot Schedule

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

### Step 6 — Attach Schedule to the Data Disk

1. **Compute Engine** → **Disks** → Click `lab3-data-disk`
2. Click **Edit**
3. Under **Snapshot schedule** → Select `lab3-daily-snapshot`
4. Click **Save**

---

### Step 7 — Take a Manual Snapshot

1. **Compute Engine** → **Disks** → `lab3-data-disk` → **Create Snapshot**
2. Name: `lab3-manual-snap-001`
3. Description: `Initial baseline snapshot before data load`
4. Storage location: **Multi-regional** → `us`
5. Click **Create**
6. Monitor status under **Compute Engine** → **Snapshots**

---

### Step 8 — Cleanup

Remove all resources created in this lab to avoid ongoing charges.

**Delete the Manual Snapshot:**
1. **Compute Engine** → **Snapshots**
2. Check the box next to `lab3-manual-snap-001`
3. Click **Delete** → Confirm

**Detach the Snapshot Schedule from the Disk:**
1. **Compute Engine** → **Disks** → Click `lab3-data-disk`
2. Click **Edit**
3. Under **Snapshot schedule** → Select **No schedule**
4. Click **Save**

**Delete the Snapshot Schedule (Resource Policy):**
1. **Compute Engine** → **Resource Policies**
2. Check the box next to `lab3-daily-snapshot`
3. Click **Delete** → Confirm

**Detach and Delete the Data Disk:**
1. **VM Instances** → Click `lab3-disk-vm` → **Edit**
2. Under **Additional disks** → Click the **✕** next to `lab3-data-disk`
3. Click **Save**
4. **Compute Engine** → **Disks** → Check `lab3-data-disk`
5. Click **Delete** → Confirm

**Delete the VM:**
1. **VM Instances** → Check the box next to `lab3-disk-vm`
2. Click **Delete** → Confirm

> **Note:** Deleting the VM also deletes its boot disk (default behavior). Verify no other resources reference these disks before deletion.

---

### Lab 3 Success Criteria
- [ ] VM created with boot disk
- [ ] Secondary SSD disk created (10 GB) in the same zone
- [ ] Disk attached to VM in Read/Write mode
- [ ] Daily snapshot schedule created and attached to disk
- [ ] Manual snapshot visible under Snapshots with status = Ready
- [ ] All resources deleted at end of lab

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
