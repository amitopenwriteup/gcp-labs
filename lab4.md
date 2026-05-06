# GCP Custom Role Lab — Hands-On Beginner Exercise

**Level:** Beginner  
**Duration:** 30–45 minutes  
**Prerequisites:** A Google Cloud account (Free Tier is sufficient)

---

## What You Will Do

In this lab you will:

1. Create a **custom IAM role** with a small, specific set of permissions
2. Attach it to a **service account**
3. Verify the role works as expected using **Security Command Center**

This is a focused, single-path exercise — no scripts, no automation, everything done through the **Google Cloud Console UI**.

---

## Lab Architecture

```
[Google Cloud Console]
        |
        |--- IAM & Admin
        |       |--- Custom Role  (lab-storage-viewer-role)
        |       |--- Service Account  (lab-custom-sa)
        |           |--- Assigned: lab-storage-viewer-role
        |
        |--- Security Command Center
                |--- Verify no overprivilege findings
```

---

## Part 1 — Create a Custom IAM Role

### Step 1.1 — Open IAM Roles

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Select your project from the top dropdown
3. In the left sidebar click **IAM & Admin** → **Roles**

### Step 1.2 — Create the Custom Role

1. Click **+ Create Role** at the top of the page
2. Fill in the following fields:

| Field | Value |
|---|---|
| Title | `Lab Storage Viewer` |
| Description | `Custom role — read-only access to Cloud Storage only` |
| ID | `labStorageViewer` |
| Role launch stage | `General Availability` |

3. Click **+ Add Permissions**

### Step 1.3 — Add Specific Permissions

In the permissions filter box, search for and add **each** of the following permissions one by one:

| Permission | What It Allows |
|---|---|
| `storage.buckets.get` | Read metadata of a bucket |
| `storage.buckets.list` | List all buckets in the project |
| `storage.objects.get` | Read/download objects in a bucket |
| `storage.objects.list` | List objects inside a bucket |

> **Why these four?** Together they allow someone to browse and read Cloud Storage — but nothing else. They cannot create, delete, or modify any resource. This is least-privilege in practice.

4. After adding all four, click **Create**

You should now see `Lab Storage Viewer` listed on the Roles page with **4 permissions**.

---

## Part 2 — Create a Service Account and Assign the Custom Role

### Step 2.1 — Create a Service Account

1. In the left sidebar click **IAM & Admin** → **Service Accounts**
2. Click **+ Create Service Account**
3. Fill in:

| Field | Value |
|---|---|
| Name | `lab-custom-sa` |
| Description | `Service account for custom role lab` |

4. Click **Create and Continue**

### Step 2.2 — Assign Your Custom Role

1. In the **Grant this service account access** section, click the **Role** dropdown
2. In the filter box, type `Lab Storage Viewer`
3. Select the custom role you just created
4. Click **Continue** → **Done**

### Step 2.3 — Confirm the Assignment

1. Go back to **IAM & Admin** → **IAM**
2. Find `lab-custom-sa` in the principals list
3. Confirm the **Role** column shows `Lab Storage Viewer`

> **Concept Check:** Notice the difference between this and the built-in `Viewer` role. The built-in `Viewer` grants read access to almost *everything* in GCP. Your custom role grants read access to Cloud Storage *only*. Same concept, far smaller blast radius if compromised.

---

## Part 3 — Verify in Security Command Center

### Step 3.1 — Open Security Command Center

1. In the left sidebar click **Security** → **Security Command Center**
2. If this is your first time, click **Get Started** and activate the Standard tier (it is free)
3. Wait 2–3 minutes for the initial scan

### Step 3.2 — Check for Findings Related to Your Service Account

1. Click the **Findings** tab
2. In the filter bar, search for: `lab-custom-sa`
3. Observe the result — there should be **no findings** for this service account

> **Why?** SCC flags overly permissive accounts (like those with Owner or Editor). Because your service account uses a minimal custom role, it does not trigger any misconfiguration alerts. This is the goal.

### Step 3.3 — Compare with an Overprivileged Account (Optional Observation)

1. Still in the **Findings** tab, look for findings with category `ADMIN_SERVICE_ACCOUNT` or `SERVICE_ACCOUNT_KEY_EXPOSED`
2. These are examples of what SCC catches when accounts are configured poorly
3. Your `lab-custom-sa` should not appear in any of these

---

## Part 4 — Test What the Role Can and Cannot Do

This section helps you internalize what the role actually enforces.

### Permissions the Role HAS

| Action | Allowed? |
|---|---|
| List all Cloud Storage buckets | Yes |
| View bucket metadata | Yes |
| Download a file from a bucket | Yes |
| List files inside a bucket | Yes |

### Permissions the Role DOES NOT Have

| Action | Allowed? |
|---|---|
| Create a new bucket | No |
| Delete a bucket or file | No |
| Upload a file | No |
| Access Compute Engine VMs | No |
| Access BigQuery datasets | No |
| Modify IAM policies | No |

> **Key insight:** A compromised service account with this role can *see* your storage but cannot damage, delete, or exfiltrate to other services. Limiting the role limits the damage.

---

## Part 5 — Reflection Questions

Write your answers in your notes:

1. What is the difference between your custom role and the built-in `Viewer` role?

2. Why is it better to create a custom role rather than always using built-in roles?

3. If a developer needed to *upload* files as well, which single permission would you add to this role?

4. What did Security Command Center show for your service account, and why?

5. What would happen in SCC if you had assigned the `Owner` role to this service account instead?

---

## Cleanup — Avoid Any Charges

Everything created in this lab is free, but clean up to keep your project tidy:

| Resource | How to Delete |
|---|---|
| Service Account (`lab-custom-sa`) | IAM & Admin → Service Accounts → select → Delete |
| Custom Role (`Lab Storage Viewer`) | IAM & Admin → Roles → select → Delete Role |

> **Note:** Custom roles and service accounts have no cost. SCC Standard tier is also free.

---

## Summary

| Step | What You Did | Concept Practiced |
|---|---|---|
| Part 1 | Created a custom role with 4 storage permissions | Least privilege, custom roles |
| Part 2 | Created a service account and assigned the custom role | IAM principal and role binding |
| Part 3 | Verified no SCC findings for the account | Misconfiguration detection |
| Part 4 | Reviewed what the role allows and blocks | Permission boundary awareness |

---

*GCP Security — Custom Role Hands-On Lab*
