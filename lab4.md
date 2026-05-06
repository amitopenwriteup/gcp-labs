# Module 4 — Hands-On Lab: GCP Security Best Practices

**Level:** Beginner  
**Duration:** 60–90 minutes  
**Prerequisites:** A Google Cloud account with billing enabled (Free Tier is sufficient)

---

## Lab Overview

In this lab you will manually explore and configure three core GCP security features:

- Understand the Shared Responsibility boundary using IAM
- Explore Security Command Center findings
- Configure Identity-Aware Proxy (IAP) on a simple App Engine app

No scripts or automation tools are used. Every step is performed through the **Google Cloud Console** UI.

---

## Lab Architecture

```
[Your Browser]
     |
     v
[Google Cloud Console]
     |
     |--- IAM (Shared Responsibility demo)
     |--- Security Command Center (findings review)
     |--- App Engine App + IAP (Zero-Trust access control)
```

---

## Part 1 — Shared Responsibility: Exploring IAM

The goal here is to see, firsthand, what YOU (the customer) are responsible for managing — specifically identity and access.

### Step 1.1 — Open IAM

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Select your project from the top dropdown
3. In the left sidebar, click **IAM & Admin** → **IAM**

You will see a list of principals (accounts) and their roles. This is your side of the shared responsibility model.

### Step 1.2 — Review Existing Roles

1. Look at the **Role** column for each principal
2. Identify any accounts with **Owner** or **Editor** roles
3. Ask yourself: *Does every account here need this level of access?*

> **Concept Check:** Google manages the underlying infrastructure, but you are fully responsible for who has access to your project. An overprivileged account is a customer-side security gap, not a Google-side one.

### Step 1.3 — Create a Read-Only Service Account

1. In the left sidebar click **IAM & Admin** → **Service Accounts**
2. Click **+ Create Service Account**
3. Fill in:
   - **Name:** `lab-readonly-sa`
   - **Description:** Lab service account with minimal permissions
4. Click **Create and Continue**
5. In the **Grant this service account access** section, choose the role: **Viewer**
6. Click **Continue** → **Done**

> **Why this matters:** Viewer is the least privilege needed to read resources. Assigning Owner to a service account is a common misconfiguration that Security Command Center will flag.

### Step 1.4 — Observe What Viewer Cannot Do

1. Note that the `lab-readonly-sa` has **Viewer** role
2. Think about what this account cannot do: it cannot create, delete, or modify any resource
3. This is the principle of **least privilege** — a core customer responsibility

---

## Part 2 — Security Command Center: Reviewing Findings

Security Command Center (SCC) gives you visibility into misconfigurations and vulnerabilities across your GCP project.

### Step 2.1 — Open Security Command Center

1. In the left sidebar, click **Security** → **Security Command Center**
2. If prompted, click **Get Started** and activate SCC for your project (Standard tier is free)
3. Wait 1–2 minutes for the initial scan to complete

### Step 2.2 — Explore the Dashboard

Once loaded, you will see:

- **Findings** — active security issues detected
- **Assets** — a full inventory of your GCP resources
- **Compliance** — how your project maps to security benchmarks

Take a few minutes to click through each tab and read what is listed.

### Step 2.3 — View Active Findings

1. Click the **Findings** tab
2. Look at the **Category** column — common findings include:
   - `PUBLIC_BUCKET_ACL` — a Cloud Storage bucket is publicly accessible
   - `OPEN_FIREWALL` — a firewall rule allows traffic from `0.0.0.0/0`
   - `MFA_NOT_ENFORCED` — multi-factor authentication is not required

3. Click on any finding to expand its details
4. Read the **Explanation** and **Recommendation** sections

> **Observation:** SCC is telling you exactly what to fix and why. This is the tool that bridges visibility and action on your side of the shared responsibility boundary.

### Step 2.4 — Check Asset Inventory

1. Click the **Assets** tab
2. Use the **Filter** bar to filter by **Asset Type**: `storage.googleapis.com/Bucket`
3. Review what buckets exist in your project and their properties

> **Question to reflect on:** Are there any assets here you did not know existed? SCC's asset inventory catches shadow resources that developers may have created and forgotten.

### Step 2.5 — Create a Misconfiguration to Observe (Optional)

If you want to see SCC detect something live:

1. Go to **Cloud Storage** → **Buckets**
2. Click **Create Bucket**, name it `lab-test-public-bucket-[yourname]`, click through defaults and click **Create**
3. Click on the bucket → **Permissions** tab → **Grant Access**
4. Add principal: `allUsers`, role: **Storage Object Viewer** → click **Save**
5. Confirm the public access warning
6. Wait 5–10 minutes, then return to SCC → **Findings**
7. You should see a new `PUBLIC_BUCKET_ACL` finding appear

**Clean up:** After observing the finding, remove the `allUsers` permission from the bucket.

---

## Part 3 — Identity-Aware Proxy: Zero-Trust Access Control

IAP lets you protect an application so that only specific Google accounts can access it — no VPN needed.

### Step 3.1 — Enable the Required APIs

1. In the left sidebar, go to **APIs & Services** → **Library**
2. Search for and enable the following APIs (click each → Enable):
   - `App Engine API`
   - `Cloud Build API`
   - `Identity-Aware Proxy API`

### Step 3.2 — Deploy a Simple App Engine Application

You will deploy a minimal Hello World app manually through Cloud Shell.

1. Click the **Cloud Shell** icon (terminal icon) in the top-right corner of the console
2. In the Cloud Shell terminal, run these commands **one by one**:

```bash
# Create a project directory
mkdir iap-lab && cd iap-lab
```

```bash
# Create the application file
cat > main.py << 'EOF'
from flask import Flask, request
app = Flask(__name__)

@app.route('/')
def hello():
    email = request.headers.get('X-Goog-Authenticated-User-Email', 'Unknown')
    return f'<h1>Hello from IAP Lab!</h1><p>Logged in as: {email}</p>'

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=8080)
EOF
```

```bash
# Create the requirements file
cat > requirements.txt << 'EOF'
Flask==2.3.3
EOF
```

```bash
# Create the App Engine config
cat > app.yaml << 'EOF'
runtime: python311
EOF
```

```bash
# Deploy the app (replace YOUR_PROJECT_ID with your actual project ID)
gcloud app deploy --project=YOUR_PROJECT_ID --quiet
```

3. When prompted to select a region, choose one close to you (e.g., `us-central`)
4. Deployment takes 3–5 minutes. When done, note the URL shown (e.g., `https://your-project-id.uc.r.appspot.com`)

### Step 3.3 — Access the App Without IAP

1. Open the App Engine URL in your browser
2. You should see the Hello World page — **anyone** with the link can access it
3. This is the problem IAP will solve

### Step 3.4 — Enable IAP on the App Engine App

1. In the Cloud Console sidebar, go to **Security** → **Identity-Aware Proxy**
2. You will see your App Engine app listed under **HTTPS Resources**
3. Toggle the **IAP** switch to **ON** for your app
4. In the confirmation dialog, click **Turn On**

### Step 3.5 — Try Accessing the App Again

1. Open the App Engine URL in a **new incognito/private browser window**
2. You should now see a **Google Sign-In page** or a **403 Access Denied** error
3. This confirms IAP is blocking unauthenticated access

> **What just happened:** IAP placed itself in front of your app. Every request now goes through Google's identity verification before reaching your application code. No code changes were needed.

### Step 3.6 — Grant Yourself Access Through IAP

1. Return to **Security** → **Identity-Aware Proxy**
2. Check the checkbox next to your App Engine app
3. In the right-hand panel, click **Add Principal**
4. Enter your own Google account email
5. Assign the role: **IAP-secured Web App User**
6. Click **Save**

### Step 3.7 — Verify Access

1. Return to the App Engine URL in your browser
2. Sign in with the Google account you just added
3. You should now see the Hello World page with your email displayed
4. Try accessing it from a different Google account — it should be denied

> **Key Insight:** You just implemented zero-trust access. The application is on the public internet, but only your explicitly allowed identity can reach it. This is what IAP + BeyondCorp replaces a VPN with.

---

## Part 4 — Reflection Questions

Answer these in your notes after completing the lab:

1. In Part 1, which roles did you find in your IAM page? Were any overly permissive?

2. In Part 2, what findings did SCC surface? Were any surprising?

3. Before enabling IAP, who could access your App Engine app?

4. After enabling IAP, what changed — in the infrastructure, in the code, or in the access policy?

5. How does IAP demonstrate the zero-trust principle of "never trust, always verify"?

---

## Cleanup — Avoid Charges

Perform these steps after the lab to avoid ongoing costs:

| Resource | How to Delete |
|---|---|
| App Engine app | Cloud Shell → `gcloud app versions delete VERSION_ID` |
| Test storage bucket | Cloud Storage → select bucket → Delete |
| Service account | IAM & Admin → Service Accounts → select → Delete |
| IAP config | Automatically removed when App Engine app is deleted |

> **Note:** App Engine has a free daily quota. If you stay within it, no charges apply. The service account and SCC are always free.

---

## Summary

| Part | What You Did | Security Concept |
|---|---|---|
| Part 1 | Created a least-privilege service account | Shared Responsibility — customer owns IAM |
| Part 2 | Reviewed SCC findings and asset inventory | Visibility and misconfiguration detection |
| Part 3 | Protected an app with IAP | Zero-trust, identity-based access control |

---

*Module 4 Lab — GCP Security Best Practices & Compliance*
