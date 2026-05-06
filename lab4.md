# GCP Compute Engine Custom Role Lab — Hands-On Beginner Exercise

**Level:** Beginner
**Duration:** 30–45 minutes
**Prerequisites:** A Google Cloud account (Free Tier is sufficient)

---

## What You Will Do

In this lab you will:

1. Create a **custom IAM role** with minimum Compute Engine permissions
2. Create a **service account** and assign the custom role to it
3. Attach the service account to a **Compute Engine VM**
4. Verify the setup using **Security Command Center**

This is a focused, single-path exercise — no scripts, no automation, everything done through the **Google Cloud Console UI**.

---

## Lab Architecture

```
[Google Cloud Console]
        |
        |--- IAM & Admin
        |       |--- Custom Role  (lab-ce-viewer-role)
        |       |--- Service Account  (lab-ce-sa)
        |           |--- Assigned: lab-ce-viewer-role
        |
        |--- Compute Engine
        |       |--- VM Instance
        |           |--- Attached: lab-ce-sa
        |
        |--- Security Command Center
                |--- Verify no overprivilege findings
```

---

## Part 1 — Create a Custom IAM Role for Compute Engine

### Step 1.1 — Open IAM Roles

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Select your project from the top dropdown
3. In the left sidebar click **IAM & Admin** → **Roles**

### Step 1.2 — Create the Custom Role

1. Click **+ Create Role** at the top of the page
2. Fill in the following fields:

| Field | Value |
|---|---|
| Title | `Lab CE Viewer` |
| Description | `Custom role — minimum read-only access for Compute Engine` |
| ID | `labCEViewer` |
| Role launch stage | `General Availability` |

3. Click **+ Add Permissions**

### Step 1.3 — Add Specific Permissions

In the permissions filter box, search for and add **each** of the following permissions one by one:

| Permission | What It Allows |
|---|---|
| `compute.instances.get` | Read metadata of a VM instance |
| `compute.instances.list` | List all VM instances in the project |
| `compute.zones.list` | List available zones |
| `compute.regions.list` | List available regions |
| `logging.logEntries.create` | Write logs to Cloud Logging |
| `monitoring.metricDescriptors.list` | List metric descriptors |
| `monitoring.timeSeries.create` | Send metrics to Cloud Monitoring |

> **Why these seven?** The first four allow read-only visibility into Compute Engine. The last three (`logging` and `monitoring`) are the minimum required for any VM to send logs and metrics to GCP's observability services — without them, Cloud Logging and Cloud Monitoring will not receive data from the instance.

4. After adding all seven, click **Create**

You should now see `Lab CE Viewer` listed on the Roles page with **7 permissions**.

---

## Part 2 — Create a Service Account and Assign the Custom Role

### Step 2.1 — Create a Service Account

1. In the left sidebar click **IAM & Admin** → **Service Accounts**
2. Click **+ Create Service Account**
3. Fill in:

| Field | Value |
|---|---|
| Name | `lab-ce-sa` |
| Description | `Service account for Compute Engine custom role lab` |

4. Click **Create and Continue**

### Step 2.2 — Assign Your Custom Role

1. In the **Grant this service account access to project** section, click the **Role** dropdown
2. In the filter box, type `Lab CE Viewer`
3. Select the custom role you just created
4. Click **Continue** → **Done**

### Step 2.3 — Confirm the Assignment

1. Go back to **IAM & Admin** → **IAM**
2. Find `lab-ce-sa` in the principals list
3. Confirm the **Role** column shows `Lab CE Viewer`

> **Concept Check:** The default Compute Engine service account has the broad `Editor` role on the entire project. Your custom service account has only 7 permissions — scoped entirely to reading Compute Engine resources and writing observability data. Same VM, far smaller blast radius if compromised.

---

## Part 3 — Attach the Service Account to a Compute Engine VM

### Step 3.1 — Open Compute Engine

1. In the left sidebar click **Compute Engine** → **VM Instances**
2. Click **+ Create Instance**

### Step 3.2 — Configure the VM

Fill in the basic settings:

| Field | Value |
|---|---|
| Name | `lab-ce-vm` |
| Region | Any (e.g., `us-central1`) |
| Zone | Any (e.g., `us-central1-a`) |
| Machine type | `e2-micro` (free tier eligible) |

### Step 3.3 — Attach the Service Account

1. Scroll down to the **Identity and API access** section
2. Under **Service account**, click the dropdown and select `lab-ce-sa`
3. Under **Access scopes**, select **Allow full access to all Cloud APIs**

> **Important:** Setting access scopes to "Allow full access" is safe here **because** the service account role is already tightly restricted. The IAM role — not the access scope — enforces the actual permission boundary.

4. Leave all other settings as default
5. Click **Create**

Wait 1–2 minutes for the VM to start. The status indicator will turn green when it is running.

### Step 3.4 — Confirm the Service Account on the VM

1. Click the VM name `lab-ce-vm` to open its details page
2. Under the **API and identity management** section, confirm:
   - Service account: `lab-ce-sa@<your-project>.iam.gserviceaccount.com`

---

## Part 4 — Verify in Security Command Center

### Step 4.1 — Open Security Command Center

1. In the left sidebar click **Security** → **Security Command Center**
2. If this is your first time, click **Get Started** and activate the Standard tier (it is free)
3. Wait 2–3 minutes for the initial scan to complete

### Step 4.2 — Check for Findings Related to Your Service Account

1. Click the **Findings** tab
2. In the filter bar, search for: `lab-ce-sa`
3. Observe the result — there should be **no findings** for this service account

> **Why?** SCC flags overly permissive service accounts such as those assigned `Owner`, `Editor`, or `roles/iam.serviceAccountTokenCreator`. Because your VM uses a minimal custom role, it does not trigger any misconfiguration alerts. This is the goal.

### Step 4.3 — Compare with an Overprivileged Account (Optional Observation)

1. Still in the **Findings** tab, look for findings with category `ADMIN_SERVICE_ACCOUNT` or `SERVICE_ACCOUNT_KEY_EXPOSED`
2. These show what SCC catches when VMs use the default Compute Engine service account or an overly broad role
3. Your `lab-ce-sa` should not appear in any of these categories

---

## Part 5 — Test What the Role Can and Cannot Do

### Permissions the Role HAS

| Action | Allowed? |
|---|---|
| List all VM instances in the project | ✅ Yes |
| Read VM instance metadata | ✅ Yes |
| List zones and regions | ✅ Yes |
| Write logs to Cloud Logging | ✅ Yes |
| Send metrics to Cloud Monitoring | ✅ Yes |

### Permissions the Role DOES NOT Have

| Action | Allowed? |
|---|---|
| Create or delete a VM | ❌ No |
| SSH into a VM | ❌ No |
| Stop or restart a VM | ❌ No |
| Access Cloud Storage buckets | ❌ No |
| Access BigQuery datasets | ❌ No |
| Modify IAM policies | ❌ No |
| Access Secret Manager | ❌ No |

> **Key insight:** A compromised service account with this role can read your Compute Engine inventory but cannot create, delete, modify, or move laterally to other GCP services. Limiting the role limits the damage.

---

## Part 6 — Reflection Questions

Write your answers in your notes:

1. What is the difference between your custom role and the default Compute Engine service account role?
2. Why is it risky to let GCP attach the default service account (`Editor`) to every new VM?
3. If a developer also needed the VM to **write files to Cloud Storage**, which role would you add?
4. What did Security Command Center show for your service account, and why?
5. What would happen in SCC if you had assigned the `Owner` role to this service account instead?

---

## Cleanup — Avoid Any Charges

The VM (`e2-micro`) is free under the GCP Free Tier but delete it to stay tidy:

| Resource | How to Delete |
|---|---|
| VM (`lab-ce-vm`) | Compute Engine → VM Instances → select → Delete |
| Service Account (`lab-ce-sa`) | IAM & Admin → Service Accounts → select → Delete |
| Custom Role (`Lab CE Viewer`) | IAM & Admin → Roles → select → Delete Role |

> **Note:** Custom roles and service accounts have no cost. SCC Standard tier is also free. The `e2-micro` VM is free in `us-central1`, `us-west1`, and `us-east1` under Free Tier limits.

---

## Summary

| Step | What You Did | Concept Practiced |
|---|---|---|
| Part 1 | Created a custom role with 7 Compute Engine permissions | Least privilege, custom roles |
| Part 2 | Created a service account and assigned the custom role | IAM principal and role binding |
| Part 3 | Created a VM and attached the service account to it | Identity and API access on Compute Engine |
| Part 4 | Verified no SCC findings for the account | Misconfiguration detection |
| Part 5 | Reviewed what the role allows and blocks | Permission boundary awareness |

---

*GCP Security — Compute Engine Custom Role Hands-On Lab*
