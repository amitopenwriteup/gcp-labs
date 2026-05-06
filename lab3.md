# GCP IAM Fundamentals — Google Cloud Console UI Lab
### Migration Project: Identity & Access Management Setup

---

> **Interface:** Google Cloud Console (Web UI — no CLI required)
> **Level:** Intermediate
> **Duration:** ~3–4 Hours
> **URL:** [console.cloud.google.com](https://console.cloud.google.com)
> **Goal:** Configure a complete, production-grade IAM setup for a cloud migration project using only Console UI navigation

---

## How to Read This Lab

Each step uses this notation to guide you through the UI:

| Symbol | Meaning |
|--------|---------|
| **Navigation >** | Left sidebar menu path |
| `[ Button ]` | A button to click |
| **"Text"** | Text to type into a field |
| Checkbox | A checkbox to enable |
| Search Box | Use the search/filter box |
| Checkpoint | Verify your result before continuing |

---

## Lab Setup — Sign In & Select Project

### Step 1 — Open the Console

1. Open your browser and go to **[https://console.cloud.google.com](https://console.cloud.google.com)**
2. Sign in using your Google account credentials

### Step 2 — Select Your Project

1. Click the **project selector dropdown** in the blue top navigation bar (shows current project name or "Select a project")
2. In the modal that appears, click `[ Select a project ]`
3. Browse or search for your migration project (e.g., `migration-lab-XXXX`)
4. Click the project name to select it
5. The top bar now shows your project name

> **No project yet?** Click `[ NEW PROJECT ]` in the selector modal, enter a project name like `migration-lab`, and click `[ CREATE ]`. Wait ~30 seconds for it to provision.

### Step 3 — Enable Required APIs

1. Click the **Navigation menu** (hamburger icon, top-left)
2. Navigate to **APIs & Services > Library**
3. In the search bar, type **"Identity and Access Management"**
4. Click **Cloud Identity-Aware Proxy API** result — click `[ ENABLE ]`
5. Go back (browser Back button) and search **"IAM Service Account Credentials"** — click `[ ENABLE ]`
6. Search **"Cloud Resource Manager"** — click `[ ENABLE ]`
7. Search **"Recommender"** — click `[ ENABLE ]`

**Checkpoint:** All APIs show a green **"API Enabled"** badge on their detail pages.

---

---

# Module 1 — IAM Fundamentals

---

## Lab 1.1 — Exploring Principals, Roles & Permissions

**Objective:** Navigate the IAM dashboard, understand how principals are bound to roles, and grant your first IAM role through the Console.

---

### Key Concepts

| Term | What it Means |
|------|---------------|
| **Principal** | Who gets access — a user, group, or service account |
| **Role** | A named collection of permissions |
| **Permission** | One specific action (e.g., `storage.objects.get`) |
| **Binding** | The link between a Principal and a Role on a resource |

---

### Task A — Open the IAM Dashboard

1. Click **Navigation menu** (top-left)
2. Scroll to the **IAM & Admin** section
3. Click **IAM & Admin > IAM**
4. The IAM page loads — you will see a table with columns:
 - **Principal** — email addresses of members
 - **Role** — what role they hold
 - **Inheritance** — where the role was granted (this project or a parent)

> The **Inherited** tag (grey) means the role came from an Organization or Folder above this project.

---

### Task B — Inspect an Existing Role Binding

1. On the IAM page, locate your own account in the **Principal** column
2. Click the **pencil (Edit Edit) icon** on the right side of your row
3. A right-side panel opens showing your assigned roles
4. Click the **role chip** (e.g., `Owner` or `Editor`) to expand the permissions list
5. Scroll through the permissions listed — note the format: `service.resource.action`
6. Click `[ CANCEL ]` to close without making changes

---

### Task C — Use the Filter to Find Specific Roles

1. On the IAM page, click the **Filter icon ()** or the filter bar above the table
2. Click the filter dropdown and select **Role**
3. Type **"viewer"** in the search box
4. Observe — only principals holding a Viewer role are shown
5. Clear the filter by clicking the **** on the filter chip

---

### Task D — Grant a Role to a New Principal

1. On the IAM page, click `[ + GRANT ACCESS ]` (top of the page)
2. A right-side panel opens titled **"Grant access"**
3. In the **New principals** field, type:
 ```
 user:YOUR_TEST_EMAIL@gmail.com
 ```
 *(Use a real Google account email you have access to, or a colleague's email)*
4. Click the **Select a role** dropdown
5. In the role search box, type **"BigQuery"**
6. Select **BigQuery > BigQuery Data Viewer**
7. Click `[ SAVE ]`

**Checkpoint:** The IAM table now shows your new principal with the `BigQuery Data Viewer` role.

---

### Task E — Verify the Binding Using Policy Troubleshooter

1. Click **Navigation menu**
2. Navigate to **IAM & Admin > Policy Troubleshooter**
3. Fill in the form:
 - **Principal:** the email you just granted access to
 - **Resource:** select **Project** and choose your project
 - **Permission:** type `bigquery.tables.get` and select it
4. Click `[ CHECK ACCESS ]`
5. Review the result — it should show **"Access is granted"** with a green checkmark
6. Expand the **Binding** section to see exactly which binding grants the access

**Checkpoint:** Policy Troubleshooter confirms access is granted via the binding you just created.

---

### Lab 1.1 Checklist

- [ ] Opened IAM dashboard and identified principal/role/inheritance columns
- [ ] Inspected an existing role binding and explored its permissions
- [ ] Used the Role filter to narrow down the IAM table
- [ ] Granted BigQuery Data Viewer to a new principal
- [ ] Confirmed the grant with Policy Troubleshooter

---

---

## Lab 1.2 — Predefined vs Custom Roles

**Objective:** Browse predefined GCP roles, compare their permissions, and create a custom role tailored for the migration project.

---

### Key Concepts

```
Predefined Roles Custom Roles
──────────────────────── ────────────────────────────────
Maintained by Google You create and maintain these
Auto-updated with new features Updated manually by you
Broad or service-specific Tailored to exact permissions needed
Example: roles/storage.admin Example: roles/migrationAnalyst
```

---

### Task A — Explore Predefined Roles

1. Click ** Navigation menu > IAM & Admin > Roles**
2. The Roles page shows a list of all available roles
3. In the **Filter** bar at the top, type **"Storage"**
4. Click **Storage Object Admin** in the results
5. A detail panel opens — scroll through the **Included permissions** list
6. Note permissions like: `storage.objects.create`, `storage.objects.delete`, `storage.objects.get`
7. Click the **back arrow (Back)** to return to the roles list
8. Now click **Storage Object Viewer**
9. Compare permissions — Viewer only has read permissions like `storage.objects.get`, `storage.objects.list`
10. Click **Back** to return

---

### Task B — Create a Custom Migration Analyst Role

1. Still on the **IAM & Admin > Roles** page
2. Click `[ + CREATE ROLE ]` at the top
3. Fill in the role details:

 | Field | Value |
 |-------|-------|
 | **Title** | `Migration Analyst` |
 | **Description** | `Read-only access to migration project resources` |
 | **ID** | `migrationAnalyst` |
 | **Role launch stage** | `General Availability` |

4. Click `[ + ADD PERMISSIONS ]`
5. In the **Filter permissions** search box that appears, type **"bigquery.datasets"**
6. Check the box next to `bigquery.datasets.get`
7. Check the box next to `bigquery.datasets.getIamPolicy`
8. Clear the search and type **"bigquery.tables"**
9. Check boxes next to:
 - `bigquery.tables.get`
 - `bigquery.tables.list`
10. Clear the search and type **"storage.buckets"**
11. Check boxes next to:
 - `storage.buckets.get`
 - `storage.buckets.list`
12. Clear and type **"storage.objects"**
13. Check boxes next to:
 - `storage.objects.get`
 - `storage.objects.list`
14. Clear and type **"cloudsql.databases"**
15. Check boxes next to:
 - `cloudsql.databases.get`
 - `cloudsql.databases.list`
16. Click `[ ADD ]`
17. Click `[ CREATE ]`

**Checkpoint:** The custom role **Migration Analyst** now appears in the Roles list with a **CUSTOM** badge.

---

### Task C — Assign the Custom Role to a Service Account

1. Click ** Navigation menu > IAM & Admin > IAM**
2. Click `[ + GRANT ACCESS ]`
3. In the **New principals** field, type:
 ```
 serviceAccount:migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com
 ```
 *(We will create this service account in Module 2 — skip this step if it doesn't exist yet and return after Lab 2.1)*
4. In **Select a role**, type `Migration Analyst` and select your custom role
5. Click `[ SAVE ]`

---

### Task D — Edit a Custom Role to Add More Permissions

1. Navigate to **IAM & Admin > Roles**
2. Click on your **Migration Analyst** custom role
3. Click `[ EDIT ROLE ]` (pencil icon at the top)
4. Click `[ + ADD PERMISSIONS ]`
5. Search for **"compute.instances"**
6. Check boxes next to:
 - `compute.instances.get`
 - `compute.instances.list`
7. Click `[ ADD ]`
8. Click `[ UPDATE ]`

**Checkpoint:** The Migration Analyst role now includes Compute Engine permissions in its list.

---

### Lab 1.2 Checklist

- [ ] Explored and compared Storage Admin vs Storage Object Viewer permissions
- [ ] Created the Migration Analyst custom role with 10+ permissions
- [ ] Assigned the custom role to a principal via IAM page
- [ ] Edited the custom role to add Compute Engine permissions

---

---

## Lab 1.3 — Policy Inheritance & Evaluation

**Objective:** Observe how IAM policies propagate down the GCP resource hierarchy, and understand effective permissions by comparing project-level vs resource-level bindings.

---

### Key Concepts

```
GCP Resource Hierarchy
──────────────────────────────────────────────────
Organization
 └── Folder: Production
 └── Project: migration-lab Back You are here
 ├── Cloud Storage Bucket
 └── BigQuery Dataset

IAM Inheritance Rules:
 Roles granted at Organization apply to ALL resources below
 Roles granted at Folder apply to all projects in that folder
 Roles granted at Project apply to all resources in the project
 You can add MORE permissions at lower levels (never fewer)
```

---

### Task A — View the Resource Hierarchy

1. Click ** Navigation menu > IAM & Admin > Manage Resources**
2. The page shows the hierarchy tree — expand folders by clicking the arrow **>** next to each
3. Find your project in the tree
4. Note the **Organization** and **Folder** above it (if present)
5. Click on your project name to highlight it

---

### Task B — Create a Cloud Storage Bucket (for inheritance demo)

1. Click ** Navigation menu > Cloud Storage > Buckets**
2. Click `[ + CREATE ]`
3. Fill in bucket settings:

 | Setting | Value |
 |---------|-------|
 | **Name** | `migration-lab-bucket-YOURNAME` (must be globally unique) |
 | **Region** | Select `asia-south1 (Mumbai)` |
 | **Storage class** | Standard |
 | **Access control** | Uniform *(leave default)* |

4. Click `[ CREATE ]`
5. If prompted about public access, click `[ CONFIRM ]`

**Checkpoint:** The bucket appears in the Buckets list.

---

### Task C — View Bucket-Level IAM (Resource-Level)

1. Click your newly created bucket name in the list
2. Click the **PERMISSIONS** tab at the top of the bucket detail page
3. Click **VIEW BY PRINCIPALS**
4. Observe the list — any principal here has access **only to this bucket**
5. Look for the **Inheritance** column:
 - **"This resource"** — role was set directly on this bucket
 - **"Project: YOUR_PROJECT"** — role was inherited from the project
6. Note how project-level roles (like your Owner role) appear here as inherited

---

### Task D — Add a Bucket-Specific Role Binding

1. Still on the bucket **PERMISSIONS** tab
2. Click `[ + GRANT ACCESS ]`
3. In **New principals**, type a test email address:
 ```
 user:YOUR_TEST_EMAIL@gmail.com
 ```
4. In **Select a role**, choose **Cloud Storage > Storage Object Viewer**
5. Click `[ SAVE ]`
6. The principal now appears in the list with **"This resource"** as inheritance source

> **Inheritance in action:** This user can read objects in *this bucket only* — they have no storage access to other buckets in the project unless granted separately.

---

### Task E — Compare Project vs Bucket Permissions

1. Navigate back to **IAM & Admin > IAM** (the project-level page)
2. Find the same test email you added at the bucket level
3. Notice: this user does **NOT** appear in the project IAM list (they only have bucket-level access)
4. Now go back to **Cloud Storage > Buckets > your bucket > PERMISSIONS**
5. The user appears here — this is a resource-level binding, not inherited from project

---

### Task F — Review IAM Audit Logs

1. Click ** Navigation menu > Logging > Log Explorer**
2. In the query editor, paste this query:

 ```
 resource.type="project"
 protoPayload.serviceName="cloudresourcemanager.googleapis.com"
 protoPayload.methodName=~"SetIamPolicy"
 ```

3. Click `[ Run Query ]`
4. Expand a log entry by clicking the **> arrow**
5. Inside the log, find:
 - `authenticationInfo.principalEmail` — who made the IAM change
 - `request.policy.bindings` — what was changed
 - `timestamp` — when it happened

**Checkpoint:** You can see audit log entries for all the IAM changes you made in this module.

---

### Lab 1.3 Checklist

- [ ] Viewed the resource hierarchy in Manage Resources
- [ ] Created a Cloud Storage bucket in asia-south1
- [ ] Viewed bucket-level IAM permissions tab with inheritance labels
- [ ] Added a bucket-specific role binding for a test user
- [ ] Compared project-level vs bucket-level IAM bindings
- [ ] Reviewed IAM change audit logs in Log Explorer

---

---

# Module 2 — Service Accounts

---

## Lab 2.1 — Service Account Types & Use Cases

**Objective:** Create and configure dedicated service accounts for each migration project workload.

---

### Migration Service Accounts to Create

| Service Account Name | Purpose |
|---------------------|---------|
| `migration-reader` | Reads source data from GCS and BigQuery |
| `migration-writer` | Writes transformed data to destination |
| `migration-orchestrator` | Runs Dataflow and orchestration pipelines |
| `migration-audit` | Writes logs and monitoring metrics |

---

### Task A — View Existing Service Accounts

1. Click ** Navigation menu > IAM & Admin > Service Accounts**
2. The page lists all service accounts in the project
3. Look for the **default compute service account** — it looks like:
 ```
 PROJECT_NUMBER-compute@developer.gserviceaccount.com
 ```
4. Click on it to view its detail page — note the **Keys** tab and **Permissions** tab

---

### Task B — Create the Migration Reader Service Account

1. Go back to **IAM & Admin > Service Accounts**
2. Click `[ + CREATE SERVICE ACCOUNT ]`
3. Fill in **Step 1 — Service account details:**

 | Field | Value |
 |-------|-------|
 | **Service account name** | `Migration Reader` |
 | **Service account ID** | `migration-reader` *(auto-filled)* |
 | **Description** | `Reads source data during migration` |

4. Click `[ CREATE AND CONTINUE ]`
5. In **Step 2 — Grant this service account access to project:**
 - Click the **Select a role** dropdown
 - Type **"BigQuery Data Viewer"** and select it
 - Click `[ + ADD ANOTHER ROLE ]`
 - Type **"Storage Object Viewer"** and select it
6. Click `[ CONTINUE ]`
7. In **Step 3 — Grant users access to this service account:** leave blank
8. Click `[ DONE ]`

**Checkpoint:** `migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com` appears in the list.

---

### Task C — Create the Migration Writer Service Account

1. Click `[ + CREATE SERVICE ACCOUNT ]`
2. Fill in:

 | Field | Value |
 |-------|-------|
 | **Name** | `Migration Writer` |
 | **ID** | `migration-writer` |
 | **Description** | `Writes transformed data to destination GCP services` |

3. Click `[ CREATE AND CONTINUE ]`
4. Add these roles:
 - **BigQuery Data Editor**
 - **Storage Object Creator**
5. Click `[ CONTINUE ]` → `[ DONE ]`

---

### Task D — Create the Migration Orchestrator Service Account

1. Click `[ + CREATE SERVICE ACCOUNT ]`
2. Fill in:

 | Field | Value |
 |-------|-------|
 | **Name** | `Migration Orchestrator` |
 | **ID** | `migration-orchestrator` |
 | **Description** | `Orchestrates migration pipelines` |

3. Click `[ CREATE AND CONTINUE ]`
4. Add these roles:
 - **Dataflow Admin**
 - **Service Account User**
5. Click `[ CONTINUE ]` → `[ DONE ]`

---

### Task E — Create the Migration Audit Service Account

1. Click `[ + CREATE SERVICE ACCOUNT ]`
2. Fill in:

 | Field | Value |
 |-------|-------|
 | **Name** | `Migration Audit` |
 | **ID** | `migration-audit` |
 | **Description** | `Writes migration logs and monitoring metrics` |

3. Click `[ CREATE AND CONTINUE ]`
4. Add these roles:
 - **Logs Writer**
 - **Monitoring Metric Writer**
5. Click `[ CONTINUE ]` → `[ DONE ]`

**Checkpoint:** All 4 migration service accounts appear in the Service Accounts list.

---

### Task F — Inspect a Service Account's Effective Permissions

1. Click on `migration-reader@...` to open its detail page
2. Click the **PERMISSIONS** tab
3. This shows who has access **to** this service account (not what it can access)
4. Now navigate to **IAM & Admin > IAM**
5. In the filter bar, type `migration-reader`
6. You'll see both role bindings granted to the reader SA at the project level

---

### Lab 2.1 Checklist

- [ ] Viewed default compute service account and its details
- [ ] Created migration-reader SA with BigQuery Data Viewer + Storage Object Viewer
- [ ] Created migration-writer SA with BigQuery Data Editor + Storage Object Creator
- [ ] Created migration-orchestrator SA with Dataflow Admin + Service Account User
- [ ] Created migration-audit SA with Logs Writer + Monitoring Metric Writer
- [ ] Verified all 4 SAs appear in the IAM list

---

---

## Lab 2.2 — Key Management & Rotation Strategies

**Objective:** Understand service account key creation and the key rotation workflow through the Console, and configure Secret Manager for secure key storage.

---

### Security Note

> Long-lived service account key files are a security risk. Only create keys when no alternative exists. In production, always prefer **Workload Identity Federation** or **impersonation** over key files.

---

### Key Management Best Practice Flow

```
RECOMMENDED (No Key Files):
 Compute Engine / Cloud Run → Attached Service Account → Automatic Auth

IF Keys Are Unavoidable:
 Create Key → Store in Secret Manager → Delete Local File
 │
 └── Rotate Every 90 Days:
 Create New Key → Update App → Delete Old Key
```

---

### Task A — View the Keys Tab for a Service Account

1. Navigate to **IAM & Admin > Service Accounts**
2. Click on `migration-orchestrator@...`
3. Click the **KEYS** tab
4. You should see: *"This service account does not have any keys"* — this is the secure default state
5. Note the `[ ADD KEY ]` dropdown — it offers **Create new key** or **Upload existing key**

---

### Task B — Create a Service Account Key (Lab Demonstration Only)

> Do this only for lab learning — in production, avoid key files.

1. Still on the **KEYS** tab for `migration-orchestrator@...`
2. Click `[ ADD KEY ]` → select **Create new key**
3. In the modal:
 - **Key type:** JSON *(leave selected)*
4. Click `[ CREATE ]`
5. The browser automatically downloads a `.json` file — this is your key
6. Note the key now appears in the Keys table with:
 - **Key ID** — a unique identifier
 - **Created** — today's date
 - **Status** — Active

> **Do not share or commit this file to any code repository.**

---

### Task C — Store the Key in Secret Manager

1. Click ** Navigation menu > Security > Secret Manager**
2. Click `[ + CREATE SECRET ]`
3. Fill in the secret details:

 | Field | Value |
 |-------|-------|
 | **Name** | `migration-orchestrator-sa-key` |
 | **Secret value** | Click **Upload file** and select the `.json` key file downloaded |
 | **Replication policy** | Automatic |

4. Click `[ CREATE SECRET ]`
5. The secret appears in the list — click on it to view its **Versions** tab
6. Note the **Version 1** entry — this is your stored key

**Checkpoint:** Secret created. Now delete the local key file from your Downloads folder.

---

### Task D — Simulate Key Rotation via Console

1. Return to **IAM & Admin > Service Accounts**
2. Click `migration-orchestrator@...` → **KEYS** tab
3. Click `[ ADD KEY ]` → **Create new key** → `[ CREATE ]`
4. A second key appears in the table — this is the **new key** (rotation step 1)
5. *(In a real rotation, you would now update your application to use the new key, then proceed to step 6)*
6. Find the **first/old key** in the table — click the **three-dot menu (Menu)** on that row
7. Click **Delete**
8. In the confirmation dialog, click `[ DELETE ]`

**Checkpoint:** Only one key remains — the newly rotated key. The old key has been revoked.

---

### Task E — View Key Usage in Audit Logs

1. Click ** Navigation menu > Logging > Log Explorer**
2. In the query editor, paste:

 ```
 resource.type="service_account"
 protoPayload.methodName=~"CreateServiceAccountKey|DeleteServiceAccountKey"
 ```

3. Click `[ Run Query ]`
4. Expand a log entry to see who created or deleted a key and when

---

### Lab 2.2 Checklist

- [ ] Viewed the Keys tab and confirmed default "no keys" state
- [ ] Created a service account key (JSON format)
- [ ] Stored the key in Secret Manager
- [ ] Simulated key rotation by creating a new key and deleting the old one
- [ ] Reviewed key creation/deletion events in Audit Logs

---

---

## Lab 2.3 — Service Account Impersonation

**Objective:** Configure service account impersonation through the Console, allowing principals to act as a service account without needing a key file.

---

### Impersonation Concept

```
Without Impersonation (Key File): With Impersonation:
────────────────────────────────── ─────────────────────────────────
User → Downloads SA Key File User → Has Token Creator role
 → Uses key to authenticate → Requests short-lived token
 → Key valid for 10 years → Token expires in 1 hour
 → High security risk → No key file ever stored
```

---

### Task A — Grant Impersonation Permission to Your User

1. Navigate to **IAM & Admin > Service Accounts**
2. Click on **migration-reader@...**
3. Click the **PERMISSIONS** tab on the service account detail page
4. Click `[ + GRANT ACCESS ]`
5. In **New principals**, type your own Google account email
6. In **Select a role**, choose:
 **Service Accounts > Service Account Token Creator**
7. Click `[ SAVE ]`

**Checkpoint:** Your email now appears under the Permissions tab of the migration-reader SA with the Token Creator role.

---

### Task B — Allow Orchestrator to Impersonate Reader (SA-to-SA)

1. Navigate to **IAM & Admin > Service Accounts**
2. Click on **migration-reader@...**
3. Click the **PERMISSIONS** tab
4. Click `[ + GRANT ACCESS ]`
5. In **New principals**, type the orchestrator SA email:
 ```
 serviceAccount:migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com
 ```
6. In **Select a role**, choose:
 **Service Accounts > Service Account Token Creator**
7. Click `[ SAVE ]`

**Checkpoint:** Both your user account and the orchestrator SA appear in the migration-reader Permissions tab with Token Creator role.

---

### Task C — Review Impersonation Setup Across All SAs

1. Navigate to **IAM & Admin > Service Accounts**
2. Click each migration SA and review its **PERMISSIONS** tab:

 | Service Account | Should have Token Creator granted to |
 |----------------|--------------------------------------|
 | migration-reader | Your user, migration-orchestrator |
 | migration-writer | migration-orchestrator |
 | migration-audit | migration-orchestrator |

3. For `migration-writer@...`, click the **PERMISSIONS** tab
4. Click `[ + GRANT ACCESS ]`
5. Add `migration-orchestrator@...` with **Service Account Token Creator** role
6. Click `[ SAVE ]`
7. Repeat for `migration-audit@...`

---

### Task D — Review Impersonation Events in Audit Logs

1. Navigate to ** Logging > Log Explorer**
2. Paste this query:

 ```
 protoPayload.serviceName="iamcredentials.googleapis.com"
 protoPayload.methodName="GenerateAccessToken"
 ```

3. Click `[ Run Query ]`
4. If entries appear, expand one to see:
 - **principalEmail** — who performed impersonation
 - **request.name** — which SA was impersonated
 - **response** — the short-lived token metadata

---

### Lab 2.3 Checklist

- [ ] Granted Service Account Token Creator to your user on migration-reader
- [ ] Granted Token Creator to migration-orchestrator on migration-reader
- [ ] Configured orchestrator impersonation permissions on migration-writer and migration-audit
- [ ] Reviewed the impersonation chain across all SAs
- [ ] Checked Audit Logs for GenerateAccessToken events

---

---

# Module 3 — Advanced IAM Concepts

---

## Lab 3.1 — Conditional Access Policies

**Objective:** Use IAM Conditions in the Console to restrict access based on time windows, resource paths, and expiry dates.

---

### Condition Expression Language (CEL) Quick Reference

```
Time-Based:
 request.time.getHours("Asia/Kolkata") >= 9 Back Hour in IST
 request.time.getDayOfWeek("Asia/Kolkata") >= 1 Back 0=Sunday, 1=Monday

Resource-Based:
 resource.name.startsWith("projects/_/buckets/migration")

Expiry-Based:
 request.time < timestamp('2025-12-31T00:00:00Z')
```

---

### Task A — Add a Time-Based Condition to a Role Binding

1. Navigate to **IAM & Admin > IAM**
2. Find the `migration-writer@...` service account row
3. Click the **pencil (Edit Edit) icon** on that row
4. On the edit panel, locate the **BigQuery Data Editor** role chip
5. Click `[ + ADD CONDITION ]` next to that role
6. Fill in:

 | Field | Value |
 |-------|-------|
 | **Title** | `Migration Window IST` |
 | **Description** | `Allow writes only during migration hours (2 AM–6 AM IST weekdays)` |

7. Click the **CONDITION EDITOR** tab
8. Paste this expression:

 ```
 request.time.getHours("Asia/Kolkata") >= 2 &&
 request.time.getHours("Asia/Kolkata") <= 6 &&
 request.time.getDayOfWeek("Asia/Kolkata") >= 1 &&
 request.time.getDayOfWeek("Asia/Kolkata") <= 5
 ```

9. Click `[ SAVE ]` on the condition panel
10. Click `[ SAVE ]` on the main edit panel

**Checkpoint:** The BigQuery Data Editor role for migration-writer now shows a **condition tag** badge on the IAM table.

---

### Task B — Add a Resource-Scoped Condition (Bucket-Specific Access)

1. On the **IAM & Admin > IAM** page, click `[ + GRANT ACCESS ]`
2. In **New principals**, enter:
 ```
 serviceAccount:migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com
 ```
3. In **Select a role**, choose **Cloud Storage > Storage Object Viewer**
4. Click `[ + ADD IAM CONDITION (Optional) ]`
5. Fill in:

 | Field | Value |
 |-------|-------|
 | **Title** | `Migration Bucket Only` |
 | **Description** | `Limit reader access to the migration bucket only` |

6. Click **CONDITION EDITOR** tab and paste:

 ```
 resource.name.startsWith(
 "projects/_/buckets/migration-lab-bucket"
 )
 ```

7. Click `[ SAVE ]` on condition → `[ SAVE ]` on main panel

---

### Task C — Add a Temporary Expiring Access Condition

1. Click `[ + GRANT ACCESS ]` on the IAM page
2. Add a temporary analyst user:
 - **New principals:** a test user email
 - **Role:** `BigQuery > BigQuery Data Viewer`
3. Click `[ + ADD IAM CONDITION (Optional) ]`
4. Fill in:

 | Field | Value |
 |-------|-------|
 | **Title** | `Temp Migration Access` |
 | **Description** | `Expires after migration window closes` |

5. Click **CONDITION BUILDER** tab (not Editor)
6. Click `[ + ADD CONDITION ]`
7. Set:
 - **Condition type:** Request timestamp
 - **Operator:** is before
 - **Value:** pick a date 7 days from today using the date picker
8. Click `[ SAVE ]` → `[ SAVE ]`

**Checkpoint:** The binding shows a condition with an expiry date tag.

---

### Task D — Test Conditions with Policy Troubleshooter

1. Navigate to **IAM & Admin > Policy Troubleshooter**
2. Set:
 - **Principal:** `migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com`
 - **Resource:** your project
 - **Permission:** `bigquery.tables.updateData`
3. Click `[ CHECK ACCESS ]`
4. Scroll to the **Binding analysis** section
5. Observe how the condition is evaluated — the result may vary based on current time

---

### Lab 3.1 Checklist

- [ ] Added a time-window condition to migration-writer's BigQuery Data Editor role
- [ ] Added a resource-scoped condition to limit migration-reader to one bucket
- [ ] Created an expiring time-bound access condition for a temp analyst
- [ ] Tested condition evaluation in Policy Troubleshooter

---

---

## Lab 3.2 — IAM Recommender & Security Insights

**Objective:** Use the IAM Recommender to identify overpermissioned accounts, review security insights, and apply least-privilege recommendations.

---

### How Recommender Works

```
Your IAM bindings → Google ML analyzes → Insights generated
 90 days of usage │
 patterns ├── "Role never used"
 ├── "Use smaller role"
 └── "Remove binding"
```

---

### Task A — View Recommendations from the IAM Page

1. Navigate to **IAM & Admin > IAM**
2. Look at the IAM table — some rows may show a **lightbulb icon** in the right column
3. If visible, click any lightbulb icon to expand the insight panel
4. The panel shows:
 - **Recommendation type** (e.g., Replace role with smaller role)
 - **Confidence level** (High / Medium / Low)
 - **Permission usage** (which permissions were actually used in 90 days)

> **New projects** won't show recommendations yet — they require 90 days of usage data. Continue reading for how to navigate to the recommendations view.

---

### Task B — Navigate to the Recommendations Hub

1. Click ** Navigation menu > IAM & Admin > Recommender**
 *(Alternatively, search "Recommender" in the top search bar)*
2. The Recommender page shows all active recommendations
3. Use the **Filter** to select **Type: IAM**
4. Browse any recommendations listed

---

### Task C — Review a Recommendation Detail (If Available)

1. Click on any **IAM recommendation** row in the Recommender page
2. The detail panel opens showing:
 - **Current role** — what the principal currently has
 - **Recommended role** — what it should be reduced to
 - **Justification** — which permissions were never used
 - **Impact** — what access is removed if applied
3. You have three options:
 - `[ APPLY ]` — automatically apply the recommendation
 - `[ DISMISS ]` — mark as reviewed and not applicable
 - `[ SNOOZE ]` — remind again later

> Do not apply recommendations on service accounts you just created — they won't have usage history yet.

---

### Task D — View Security Insights in IAM

1. Navigate to **IAM & Admin > IAM**
2. Click the **INSIGHTS** tab at the top of the page (if available)
3. Review insights such as:
 - **Service account key not rotated** — key older than 90 days
 - **Role binding unused** — principal never used the granted permissions
 - **Lateral movement risk** — SA has cross-project access

---

### Task E — Check Security Command Center for IAM Findings

1. Click ** Navigation menu > Security > Security Command Center**
2. In the left panel, click **Findings**
3. In the **Filter** bar, add:
 - **Category:** contains `IAM`
4. Review findings such as:
 - **Service Account Key Exposed**
 - **Over Privileged Service Account**
 - **Admin Activity on SA Keys**
5. Click a finding to expand its details and remediation guidance

---

### Lab 3.2 Checklist

- [ ] Viewed IAM table for lightbulb insight icons
- [ ] Navigated to the Recommender hub and filtered by IAM type
- [ ] Reviewed a recommendation detail panel (applied or noted)
- [ ] Checked IAM Insights tab for usage analysis
- [ ] Reviewed IAM-related findings in Security Command Center

---

---

## Lab 3.3 — Organization Policies & Constraints

**Objective:** Configure Organization Policies in the Console to enforce infrastructure-level governance guardrails for the migration project.

---

### IAM vs Organization Policy

```
IAM Policy Organization Policy
───────────────────────────── ─────────────────────────────────────
Controls WHO can do things Controls WHAT can be done
"Alice can create buckets" "Buckets must be in asia-south1"
Applied to identities Applied to resources and configurations
Can be overridden by IAM grants CANNOT be overridden by IAM grants
```

---

### Task A — Browse Available Constraints

1. Click ** Navigation menu > IAM & Admin > Organization Policies**
2. The page lists all available constraints
3. Use the filter and type **"iam"** to see IAM-related constraints:
 - `constraints/iam.disableServiceAccountKeyCreation`
 - `constraints/iam.allowedPolicyMemberDomains`
 - `constraints/iam.disableServiceAccountKeyUpload`
4. Click on **`iam.disableServiceAccountKeyCreation`** to view it
5. Review the **Effective policy** section and note whether it is currently **Enforced** or **Not enforced**
6. Click **Back** to go back

---

### Task B — Enforce Uniform Bucket-Level Access

1. Still on **IAM & Admin > Organization Policies**
2. In the filter, type **"uniform"**
3. Click **`storage.uniformBucketLevelAccess`**
4. Click `[ MANAGE POLICY ]`
5. Under **Policy source**, select **Override parent's policy**
6. Under **Policy values**, select **Enforcement: On**
7. Click `[ SET POLICY ]`

**Checkpoint:** The policy shows **Enforced** status. New buckets in this project will be forced to use uniform access control.

---

### Task C — Restrict Resource Locations

1. Go to **IAM & Admin > Organization Policies**
2. Search for **"resourceLocations"**
3. Click **`gcp.resourceLocations`**
4. Click `[ MANAGE POLICY ]`
5. Under **Policy source**, select **Override parent's policy**
6. Under **Policy values**, select **Allow**
7. Click `[ Add values ]`
8. Add these location values one at a time:
 - `in:asia-south1-locations`
 - `in:us-central1-locations`
9. Click `[ DONE ]` after adding each
10. Click `[ SET POLICY ]`

**Checkpoint:** The policy is set. Attempting to create resources in other regions will now be blocked.

---

### Task D — Test the Location Restriction

1. Navigate to **Cloud Storage > Buckets**
2. Click `[ + CREATE ]`
3. Set a test bucket name: `test-location-constraint-123`
4. In the **Location type** dropdown, select **Region**
5. Select a **blocked** region like `us-east1`
6. Click `[ CREATE ]`
7. Observe: you should receive a **Policy violation error** preventing creation
8. Click `[ CANCEL ]` — the policy is working correctly

**Checkpoint:** The error message confirms the organization policy is blocking the creation.

---

### Task E — Review Policy Inheritance in Hierarchy

1. Navigate to **IAM & Admin > Organization Policies**
2. Click on any policy
3. Scroll to the **Effective Policy** section at the bottom
4. This shows the combined/merged policy from:
 - Organization level
 - Folder level (if applicable)
 - Project level (what you set)
5. The **Hierarchy** section shows where each rule comes from

---

### Lab 3.3 Checklist

- [ ] Browsed available Organization Policy constraints
- [ ] Enforced uniform bucket-level access at the project level
- [ ] Configured resource location restriction to asia-south1 and us-central1
- [ ] Tested the location restriction by attempting a blocked resource creation
- [ ] Reviewed the effective policy hierarchy for a constraint

---

---

# Module 4 — Setting Up IAM for the Migration Project

---

## Lab 4.1 — Full IAM Configuration for the Migration Project

**Objective:** Apply all previous concepts in a single end-to-end workflow to configure a complete, production-ready IAM setup for the cloud migration project.

---

### Target IAM Architecture

```
╔══════════════════════════════════════════════════════╗
║ MIGRATION PROJECT IAM BLUEPRINT ║
╠══════════════════════════════════════════════════════╣
║ TEAMS ║
║ ───────────────────────────────────────── ║
║ Migration Lead → Editor (project-scoped) ║
║ Data Engineers → BigQuery Admin + GCS Admin ║
║ App Engineers → Cloud Run + GKE Developer ║
║ Security Team → Custom: MigrationSecurityReviewer║
║ Audit Team → Logging Viewer + BQ Viewer ║
║ ║
║ SERVICE ACCOUNTS ║
║ ───────────────────────────────────────── ║
║ migration-reader → Storage + BQ read ║
║ migration-writer → Storage + BQ write ║
║ migration-orchestrator→ Dataflow + SA User ║
║ migration-audit → Logging + Monitoring write ║
║ ║
║ GUARDRAILS ║
║ ───────────────────────────────────────── ║
║ Uniform bucket access : ENFORCED ║
║ Resource locations : asia-south1 only ║
║ Migration window : 2 AM–6 AM IST weekdays ║
╚══════════════════════════════════════════════════════╝
```

---

### Task A — Create the Security Reviewer Custom Role

1. Navigate to **IAM & Admin > Roles**
2. Click `[ + CREATE ROLE ]`
3. Fill in:

 | Field | Value |
 |-------|-------|
 | **Title** | `Migration Security Reviewer` |
 | **Description** | `Read-only security and IAM audit access for migration reviewers` |
 | **ID** | `MigrationSecurityReviewer` |
 | **Stage** | `General Availability` |

4. Click `[ + ADD PERMISSIONS ]`
5. Add the following permissions (search each group):

 **IAM permissions:**
 - `iam.roles.get`
 - `iam.roles.list`
 - `iam.serviceAccounts.get`
 - `iam.serviceAccounts.list`
 - `iam.serviceAccounts.getIamPolicy`
 - `resourcemanager.projects.getIamPolicy`

 **Logging permissions:**
 - `logging.logEntries.list`
 - `logging.logs.list`

 **Storage permissions:**
 - `storage.buckets.getIamPolicy`
 - `storage.buckets.list`

 **BigQuery permissions:**
 - `bigquery.datasets.getIamPolicy`
 - `bigquery.datasets.get`

 **Recommender permissions:**
 - `recommender.iamPolicyRecommendations.get`
 - `recommender.iamPolicyRecommendations.list`

6. Click `[ ADD ]` → `[ CREATE ]`

**Checkpoint:** `MigrationSecurityReviewer` appears in the Roles list with **CUSTOM** badge.

---

### Task B — Assign Roles to All Teams

Navigate to **IAM & Admin > IAM** and perform the following grants using `[ + GRANT ACCESS ]` for each team.

**Migration Lead:**

| Principal | Role |
|-----------|------|
| `user:lead@yourdomain.com` | `Editor` |

**Data Engineers:**

| Principal | Role |
|-----------|------|
| `user:dataengineer@yourdomain.com` | `BigQuery Admin` |
| `user:dataengineer@yourdomain.com` | `Storage Admin` |

*(Click `[ + ADD ANOTHER ROLE ]` to add a second role in the same grant)*

**Security Team:**

| Principal | Role |
|-----------|------|
| `user:security@yourdomain.com` | `Migration Security Reviewer` *(your custom role)* |

**Audit Team:**

| Principal | Role |
|-----------|------|
| `user:auditor@yourdomain.com` | `Logs Viewer` |
| `user:auditor@yourdomain.com` | `BigQuery Data Viewer` |

> If you don't have team member emails, substitute with your own email for each role for demonstration purposes.

---

### Task C — Apply Migration Window Condition to Writer SA

1. Navigate to **IAM & Admin > IAM**
2. Find `migration-writer@...` and click the **Edit Edit icon**
3. Find the **Storage Object Creator** role chip
4. Click `[ + ADD CONDITION ]`
5. Enter:
 - **Title:** `Migration Window Only`
 - **Description:** `Restrict writes to 2–6 AM IST on weekdays`
6. Click **CONDITION EDITOR** tab and paste:

 ```
 request.time.getHours("Asia/Kolkata") >= 2 &&
 request.time.getHours("Asia/Kolkata") <= 6 &&
 request.time.getDayOfWeek("Asia/Kolkata") >= 1 &&
 request.time.getDayOfWeek("Asia/Kolkata") <= 5
 ```

7. Click `[ SAVE ]` → `[ SAVE ]`

---

### Task D — Verify the Complete IAM Configuration

Navigate through each section of the Console and verify:

**Service Accounts Check:**
1. Go to **IAM & Admin > Service Accounts**
2. Confirm all 4 migration SAs are listed:
 - [ ] `migration-reader@...`
 - [ ] `migration-writer@...`
 - [ ] `migration-orchestrator@...`
 - [ ] `migration-audit@...`

**Custom Roles Check:**
1. Go to **IAM & Admin > Roles**
2. Filter by **Type: Custom**
3. Confirm these roles exist:
 - [ ] `migrationAnalyst`
 - [ ] `MigrationSecurityReviewer`

**IAM Bindings Check:**
1. Go to **IAM & Admin > IAM**
2. In the filter, type `migration-`
3. Confirm SA bindings are present and some show the **condition tag**

**Organization Policies Check:**
1. Go to **IAM & Admin > Organization Policies**
2. Confirm:
 - [ ] `storage.uniformBucketLevelAccess` → **Enforced**
 - [ ] `gcp.resourceLocations` → **Allowed: asia-south1, us-central1**

---

### Task E — Run a Final Security Audit via Policy Troubleshooter

Test 3 critical access scenarios:

**Test 1 — Reader should have BQ read access:**
1. Navigate to **IAM & Admin > Policy Troubleshooter**
2. Principal: `migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com`
3. Resource: your project
4. Permission: `bigquery.tables.get`
5. Expected result: **Access is granted**

**Test 2 — Reader should NOT have write access:**
1. Same principal
2. Permission: `bigquery.tables.create`
3. Expected result: **Access is denied**

**Test 3 — Orchestrator should have Dataflow access:**
1. Principal: `migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com`
2. Permission: `dataflow.jobs.create`
3. Expected result: **Access is granted**

**Checkpoint:** All 3 tests return expected results.

---

### Task F — Enable Audit Logging for IAM Changes

1. Navigate to **IAM & Admin > Audit Logs**
2. In the **Filter services** search, type **"Identity and Access Management"**
3. Check the box next to **Identity and Access Management (IAM) API**
4. In the right panel, enable:
 - - **Admin Read**
 - - **Data Read**
 - - **Data Write**
5. Click `[ SAVE ]`

**Checkpoint:** All IAM operations will now generate audit log entries in Cloud Logging.

---

### Final Lab Checklist

- [ ] MigrationSecurityReviewer custom role created with all required permissions
- [ ] Team role bindings applied for Lead, Data Engineers, Security Team, Audit Team
- [ ] Migration window time condition applied to migration-writer
- [ ] All 4 migration SAs confirmed in Service Accounts page
- [ ] Both custom roles confirmed in Roles page (filtered by Custom)
- [ ] Conditional bindings visible in IAM table with condition tags
- [ ] Organization policies confirmed: uniform bucket access + location restriction
- [ ] Policy Troubleshooter tests pass for all 3 scenarios
- [ ] Audit logging enabled for IAM API

---

---

## Key Takeaways

**IAM Fundamentals**
- Use **groups** for human users — easier to manage than individual bindings
- Use **Policy Troubleshooter** any time you're debugging unexpected access
- IAM policies always **union** — child resources inherit all parent permissions

**Service Accounts**
- Create **one SA per workload** with only what that workload needs
- Use **impersonation** (Token Creator role) instead of key files wherever possible
- Store keys in **Secret Manager** if key files cannot be avoided, then delete the local file

**Advanced IAM**
- IAM **Conditions** restrict when and where a binding is active — use them for migration windows
- **IAM Recommender** catches over-permissioned accounts after 90 days of usage
- **Organization Policies** enforce constraints that IAM cannot — they cannot be overridden by granting more IAM permissions

**Migration Best Practices**
- Apply the **time-window condition** to write SAs to prevent unauthorized data writes outside planned migration windows
- Use **location constraints** to ensure migrated data stays in approved regions
- Run **Policy Troubleshooter** tests as part of your migration go-live checklist

---

## Console Navigation Quick Reference

| Task | Navigation Path |
|------|----------------|
| View IAM bindings | IAM & Admin > IAM |
| Create/edit roles | IAM & Admin > Roles |
| Manage service accounts | IAM & Admin > Service Accounts |
| Troubleshoot access | IAM & Admin > Policy Troubleshooter |
| Org policies | IAM & Admin > Organization Policies |
| Audit logs | Logging > Log Explorer |
| Recommendations | IAM & Admin > Recommender |
| Security findings | Security > Security Command Center |
| Secret Manager | Security > Secret Manager |
| Manage resources | IAM & Admin > Manage Resources |

---

*GCP IAM Fundamentals — Cloud Console UI Navigation Lab*
*Migration Project Track | Cloud Console Lab Series*
