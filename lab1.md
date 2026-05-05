# Day 0 Lab — GCP Console Orientation
### Learn Your Way Around the Google Cloud Panel

> **Goal:** Before touching any service, get completely comfortable with the GCP Console interface. By the end of this lab you will know where everything lives, what every panel option does, and how to navigate confidently.
>
> **Time:** 45–60 minutes | **Cost:** Free (no resources created) | **Level:** Absolute Beginner

---

## What You Need

- A Google account (Gmail is fine)
- A web browser (Chrome recommended)
- Go to → **https://console.cloud.google.com**

---

## The Big Picture — What Is the GCP Console?

The GCP Console is a **web-based dashboard** to manage all your Google Cloud resources. Think of it like a control room — every button, menu, and panel controls something in Google's infrastructure.

```
┌─────────────────────────────────────────────────────────────┐
│  Top Bar        │  Search │  Project Selector │  Icons      │
├──────────┬──────────────────────────────────────────────────┤
│          │                                                   │
│  Left    │          Main Content Area                       │
│  Nav     │          (changes per service)                   │
│  Menu    │                                                   │
│          │                                                   │
├──────────┴──────────────────────────────────────────────────┤
│  Bottom Bar (Cloud Shell)                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 1 — The Top Bar (The Control Strip)

Open the console and look at the very top. From left to right:

### 1a. The Hamburger Menu (☰)
- **What it is:** Three horizontal lines on the top-left
- **What it does:** Opens and closes the left navigation menu
- **Try it:** Click it — the left panel collapses. Click again — it comes back.

---

### 1b. Google Cloud Logo
- **What it does:** Clicking it takes you back to the **Home Dashboard** from anywhere
- **Try it:** Navigate to any page, then click the logo — you return to Home

---

### 1c. Project Selector (Most Important Button on the Page)

This is the dropdown that shows your current project name (e.g., `My First Project`).

**Why it matters:** Every single resource in GCP lives inside a project. If you are in the wrong project, you will not see your resources — this is the #1 cause of confusion for beginners.

**Try it:**
1. Click the project dropdown
2. A dialog opens showing:
   - **RECENT** tab — projects you recently used
   - **ALL** tab — all projects you have access to
   - **Search box** — search by project name or ID
3. Notice each project has:
   - A **Name** (human-readable, e.g., `My Project`)
   - An **ID** (unique, e.g., `my-project-382910`) — this is permanent and cannot be changed
   - A **Number** (numeric ID used internally by GCP)
4. Click **NEW PROJECT** (top right of the dialog) and explore the creation form — but do not create one yet, just look at the options:
   - Project name
   - Organization (if you are in a Google Workspace account)
   - Billing account linkage
5. Press **Cancel** and go back

> **Remember:** Project Name is for humans. Project ID is for systems. Project Number is for internal GCP use.

---

### 1d. Search Bar (The Fastest Way to Navigate)

The search bar at the top center is the fastest way to find anything.

**Try searching for these — observe what types of results appear:**

| Search Term | What You See |
|---|---|
| `compute engine` | Service shortcut + documentation links |
| `vm instances` | Direct link to VM list page |
| `cloud storage` | Service + recent buckets |
| `billing` | Billing dashboard |
| `iam` | IAM & Admin page |
| `quota` | Quotas page |

> **Pro tip:** Press `/` on your keyboard to jump to the search bar from anywhere.

---

### 1e. Top-Right Icon Strip

These small icons in the top-right corner are used constantly. Learn each one:

| Icon | Name | What It Does |
|---|---|---|
| `>_` | **Cloud Shell** | Opens a terminal in your browser — a Linux VM with `gcloud` pre-installed |
| 🔔 | **Notifications** | Alerts about your resources (errors, completions, billing alerts) |
| `?` | **Help** | Opens documentation, support, and keyboard shortcuts |
| ⚙️ | **Settings** | Console preferences (language, theme, project defaults) |

**Try each one:**

**Cloud Shell (`>_`):**
1. Click it — a terminal panel slides up from the bottom
2. Wait for it to initialize (10–20 seconds on first use)
3. You get a **free VM** (e2-micro, Debian Linux) with:
   - `gcloud` CLI pre-installed and authenticated
   - `kubectl`, `terraform`, `git`, `python3`, `node`, `docker` all available
   - 5GB persistent home directory at `/home/YOUR_USERNAME`
4. Try typing: `gcloud --version`
5. Try typing: `echo "Hello GCP"`
6. Click the **X** or the down arrow to close it

**Notifications (🔔):**
1. Click it — a panel slides in from the right
2. This shows recent activity like "VM instance created", "Deployment succeeded", "Budget alert triggered"
3. If you have no notifications yet, it will be empty — that is fine

**Help (`?`):**
1. Click it — options appear:
   - **Documentation** — opens GCP docs
   - **Send Feedback** — report issues to Google
   - **Keyboard Shortcuts** — very useful, read through these
   - **Release Notes** — what changed recently in GCP

**Settings (⚙️):**
1. Click it
2. Explore **Preferences**:
   - Language
   - **Theme** — try switching to Dark mode
   - Default region for new resources
   - Accessibility options

---

## Step 2 — The Left Navigation Menu

This is the **main menu** of the entire console. Click the ☰ hamburger to make sure it is open.

The menu is organized into sections. Scroll through the entire menu slowly and read every section. Here is what each section contains:

---

### Section: Pinned (top of menu)

This is your personal quick-access bar. By default GCP pins a few common services.

**How to pin a service:**
1. Hover over any menu item
2. A pin icon (📌) appears on the right
3. Click it — the service moves to the top of your menu
4. **Try pinning:** Compute Engine, Cloud Storage, IAM & Admin, Billing

**How to unpin:** Hover over a pinned item → click the pin icon again

---

### Section: HOME

| Item | What It Is |
|---|---|
| **Home** | Your dashboard — project overview, recent activity, resource summary |
| **Activity** | A log of every action taken in this project (by you or others) |

**Try it:** Click **Activity** — you can see a timeline of everything that has happened in this project. This is useful for auditing and debugging.

---

### Section: COMPUTE

These are all the services for running code.

| Item | What It Is | When You Use It |
|---|---|---|
| **Compute Engine** | Virtual Machines | Need full OS control |
| **Kubernetes Engine** | Managed Kubernetes (GKE) | Running containers at scale |
| **Cloud Run** | Serverless container platform | Stateless HTTP apps, no server management |
| **Cloud Run functions** | Serverless functions (Gen 2) | Event-triggered code |
| **Cloud Functions** | Serverless functions (Gen 1) | Legacy serverless functions |
| **App Engine** | Platform-as-a-Service | Web apps with automatic scaling |
| **Batch** | Managed batch jobs | High-performance computing, scientific workloads |
| **VMware Engine** | Run VMware workloads | Lift-and-shift VMware to GCP |
| **Bare Metal Solution** | Physical servers in GCP | Oracle and other bare metal workloads |

**Explore:** Click **Compute Engine** — if you have never enabled it, it will ask you to enable the API. Click **Enable** (takes 30–60 seconds). Then explore the submenu on the left.

---

### Section: STORAGE

Everything for storing data.

| Item | What It Is |
|---|---|
| **Cloud Storage** | Object storage (files, images, backups) |
| **Cloud Storage for Firebase** | Object storage optimized for Firebase apps |
| **Filestore** | Managed NFS (shared file system) |
| **NetApp Volumes** | Enterprise NetApp file storage |

**Explore:** Click **Cloud Storage** → look at the submenu items: Buckets, Monitoring, Settings.

---

### Section: DATABASES

All managed database services.

| Item | What It Is |
|---|---|
| **SQL** | Cloud SQL — managed MySQL, PostgreSQL, SQL Server |
| **Spanner** | Globally distributed relational database |
| **Firestore** | Document NoSQL database |
| **Firebase Realtime Database** | Simple real-time database for Firebase apps |
| **Bigtable** | Wide-column NoSQL for high-throughput workloads |
| **Memorystore** | Managed Redis and Valkey (in-memory cache) |
| **AlloyDB** | PostgreSQL-compatible, high performance |
| **Database Migration Service** | Migrate databases to GCP |

**Explore:** Click **SQL** — notice the submenu: Instances, Operations. This is where you create and manage your managed databases.

---

### Section: NETWORKING

All network infrastructure services.

| Item | What It Is |
|---|---|
| **VPC network** | Virtual Private Cloud — your private network |
| **Network services** | Load balancers, Cloud DNS, Cloud CDN, Traffic Director |
| **Hybrid Connectivity** | Cloud VPN, Dedicated/Partner Interconnect |
| **Network Intelligence Center** | Network monitoring, topology viewer |
| **Network Security** | Cloud Armor (WAF/DDoS), Certificate Manager |
| **Network Connectivity Center** | Hub-and-spoke network management |

**Explore:** Click **VPC network** → look at the submenu: VPC networks, Subnets, Firewall policies, Routes, External IP addresses.

> **Tip:** Click **VPC networks** and find the **default** VPC that Google creates automatically in every project. Click into it and explore its subnets.

---

### Section: OPERATIONS

Monitoring, logging, and debugging tools.

| Item | What It Is |
|---|---|
| **Monitoring** | Dashboards, alerts, uptime checks (Cloud Monitoring) |
| **Logging** | Log Explorer — search all logs (Cloud Logging) |
| **Error Reporting** | Automatically groups application errors |
| **Trace** | Request tracing across distributed services |
| **Profiler** | CPU and memory profiling for running applications |
| **Debugger** | Inspect running application state without stopping it |

**Explore — try the Log Explorer:**
1. Click **Logging → Log Explorer**
2. Look at the query box at the top — this uses Google's logging query language
3. Try this query: `resource.type="project"` → click **Run Query**
4. You should see project-level activity logs
5. Look at the fields on the left — you can filter by severity, resource type, time range

---

### Section: TOOLS

Developer and automation tools.

| Item | What It Is |
|---|---|
| **Cloud Build** | CI/CD — automated build, test, deploy pipelines |
| **Cloud Deploy** | Managed continuous delivery to GKE/Cloud Run |
| **Artifact Registry** | Store Docker images, npm, Maven, Python packages |
| **Container Registry** | Older Docker image registry (being replaced by Artifact Registry) |
| **Source Repositories** | Private Git hosting |
| **Deployment Manager** | Infrastructure as code (Google's version of Terraform) |
| **Cloud Scheduler** | Cron jobs — schedule tasks at regular intervals |
| **Cloud Tasks** | Manage and dispatch asynchronous task queues |

---

### Section: BIG DATA

Data analytics and processing services.

| Item | What It Is |
|---|---|
| **BigQuery** | Serverless data warehouse — SQL on petabytes |
| **Pub/Sub** | Async messaging and event streaming |
| **Dataflow** | Managed Apache Beam — batch and stream pipelines |
| **Dataproc** | Managed Hadoop and Spark |
| **Looker Studio** | Data visualization and dashboards |
| **Data Catalog** | Metadata management and data discovery |
| **Dataform** | SQL workflow management for BigQuery |
| **Dataplex** | Data mesh — unified data management |

**Explore — try BigQuery:**
1. Click **BigQuery**
2. The BigQuery Studio opens — this is a full SQL workbench
3. In the Explorer panel (left), look for **bigquery-public-data** — click the pin icon to add it
4. Expand it and find `samples` → `shakespeare`
5. Click the table → look at the **Schema** and **Preview** tabs
6. Click **Query** → run this SQL:

```sql
SELECT word, word_count
FROM `bigquery-public-data.samples.shakespeare`
WHERE corpus = 'hamlet'
ORDER BY word_count DESC
LIMIT 10;
```

7. Results appear in seconds — on a dataset with millions of rows. No server setup needed.

---

### Section: AI AND ML (Artificial Intelligence)

AI/ML services offered by Google Cloud.

| Item | What It Is |
|---|---|
| **Vertex AI** | Unified ML platform — train, deploy, manage models |
| **Natural Language** | Analyze text — sentiment, entities, syntax |
| **Vision** | Analyze images — labels, faces, text (OCR) |
| **Video Intelligence** | Analyze video content |
| **Translation** | Neural machine translation API |
| **Speech-to-Text** | Convert audio to text |
| **Text-to-Speech** | Convert text to audio |
| **Recommendations AI** | Personalized recommendations engine |
| **Document AI** | Parse and understand documents (invoices, forms) |
| **Dialogflow** | Build chatbots and virtual agents |

---

### Section: SECURITY

Identity, access, and security management.

| Item | What It Is |
|---|---|
| **IAM & Admin** | Control who can do what on which resources |
| **Identity-Aware Proxy** | Protect apps without VPN |
| **Secret Manager** | Store API keys, passwords securely |
| **Certificate Authority Service** | Manage SSL/TLS certificates |
| **Security Command Center** | Security posture management |
| **Binary Authorization** | Enforce trusted container images on GKE |
| **Access Context Manager** | Define access levels based on context |

**Explore — IAM & Admin (very important):**
1. Click **IAM & Admin → IAM**
2. You see a list of **principals** (accounts with access to this project)
3. Each principal has one or more **roles** (sets of permissions)
4. Click **+ Grant Access** — explore how you would add another person to your project
5. Cancel without saving
6. Click **IAM & Admin → Service Accounts**
7. You may see a list of service accounts — these are identities for applications (not humans)

---

## Step 3 — The Home Dashboard

Click the **Google Cloud logo** to go back to the Home dashboard. Understand every widget here:

### 3a. Project Info Card
Shows:
- **Project name** — the human-readable name
- **Project ID** — the permanent unique identifier (copy this — you will use it in CLI commands)
- **Project number** — GCP's internal numeric ID
- **Go to project settings** link

**Try it:** Click **Go to project settings** — this is where you can rename the project, set labels, or delete the project entirely.

---

### 3b. Resources Card
Shows counts of resources you have created:
- Number of VM instances
- Number of Cloud Storage buckets
- Number of SQL databases
- etc.

This is a quick health check — is anything running that should not be?

---

### 3c. Billing Summary Card
Shows:
- Estimated cost for the current month
- A small chart of daily spend
- Link to the full billing dashboard

> **Habit to build:** Check this card every time you log in. Unexpected charges usually mean you forgot to delete a resource.

---

### 3d. Error Reporting Card
Shows any recent application errors. When your apps crash in production, errors appear here grouped by type.

---

### 3e. Google Cloud Platform Status Card
Shows the current health of GCP services globally. If a service is having an incident (outage), it appears here. You can also bookmark: **https://status.cloud.google.com**

---

### 3f. APIs Card
Shows which APIs are enabled in this project and their recent traffic. Click **Go to APIs overview** to see the full list.

---

## Step 4 — Billing Console (Must Know)

Click **Navigation Menu → Billing**. If you have never set this up, GCP will prompt you to create a billing account.

### What to understand in Billing:

| Section | What It Shows |
|---|---|
| **Overview** | Current month cost estimate and trend chart |
| **Reports** | Break down costs by service, project, label, time |
| **Cost Table** | Detailed line-item costs per service per day |
| **Credits** | Free trial credits, promotional credits balance |
| **Budgets & Alerts** | Set spending limits and email alerts |
| **Committed Use** | Pre-purchase discounts for long-term usage |
| **Payment Method** | Credit card or invoice settings |

**Must-Do — Set a Budget Alert:**
1. Go to **Budgets & Alerts → Create Budget**
2. Set:
   - Name: `session1-budget`
   - Scope: this project
   - Budget amount: `$10` (or however much you are comfortable with)
   - Alerts: 50%, 90%, 100% thresholds
   - Email notifications: your email
3. Click **Save**

> This will email you when you reach those thresholds — essential for beginners to avoid surprise bills.

---

## Step 5 — IAM & Roles (Access Control)

Click **Navigation Menu → IAM & Admin → IAM**

This is where you control **who can do what**. Understand the three key concepts:

### Principal
Who is getting access. Can be:
- A Google account (person)
- A service account (application/robot)
- A Google group
- An entire domain

### Role
What they are allowed to do. Roles are collections of permissions. Common roles:

| Role | What It Can Do |
|---|---|
| **Owner** | Full access including billing and deleting the project |
| **Editor** | Create, modify, delete most resources — cannot manage IAM |
| **Viewer** | Read-only access to all resources |
| **Compute Admin** | Full control of Compute Engine only |
| **Storage Admin** | Full control of Cloud Storage only |
| **BigQuery Data Viewer** | Read data in BigQuery only |

### Resource
What they have access to. Access can be granted at:
- Organization level (applies to everything)
- Folder level (applies to all projects in a folder)
- Project level (applies to this project only)
- Individual resource level (e.g., one specific bucket)

**Explore:**
1. Look at your own account — click the pencil icon next to your email
2. See what roles you have (you should be Owner on your own project)
3. Click **Cancel**
4. Click **IAM & Admin → Roles** in the left submenu
5. Search for `storage` — look at all the Storage-related roles
6. Click **Storage Object Viewer** — see the exact list of permissions it includes

---

## Step 6 — APIs & Services

Click **Navigation Menu → APIs & Services → Library**

### Why this matters:
Every GCP service has an API. Before you can use a service, you must **enable its API** in your project. This is a common gotcha for beginners — if you get an error saying "API not enabled", this is where you fix it.

**Explore:**
1. Look at the API Library — thousands of APIs organized by category
2. Search for `Compute Engine API` — click it — notice the Enable/Disable button
3. Go to **APIs & Services → Enabled APIs & Services** — see every API currently active in your project
4. Go to **APIs & Services → Credentials** — this is where you create:
   - **API Keys** (for simple API access)
   - **OAuth 2.0 Client IDs** (for apps that access user data)
   - **Service Account Keys** (for applications to authenticate as a service account)

> **Security note:** Never commit API keys or service account key files to Git. Use Secret Manager instead.

---

## Step 7 — Cloud Shell In Depth

Click the `>_` icon to open Cloud Shell again. This is your browser-based terminal.

### What Cloud Shell gives you:

```
Free VM (e2-micro, Debian Linux)
├── 5 GB persistent home directory
├── gcloud CLI (pre-authenticated to your account)
├── kubectl (Kubernetes CLI)
├── git
├── python3, pip
├── node.js, npm
├── docker
├── terraform
├── vim, nano (text editors)
└── Resets after 1 hour of inactivity (your /home persists)
```

### Try these commands in Cloud Shell:

```bash
# Who am I logged in as?
gcloud auth list

# What project am I working in?
gcloud config get-value project

# What region/zone am I set to?
gcloud config list

# What version of gcloud am I using?
gcloud --version

# List all GCP projects I have access to
gcloud projects list

# View the current project's details
gcloud projects describe $(gcloud config get-value project)

# List all APIs enabled in this project
gcloud services list --enabled

# Check my account's IAM permissions in this project
gcloud projects get-iam-policy $(gcloud config get-value project)
```

### Cloud Shell Editor:

Click the **pencil icon** (Open Editor) in the Cloud Shell toolbar. This opens a full **VS Code-like editor** in your browser — you can write code, edit files, and sync with Cloud Shell terminal, all without installing anything locally.

**Try it:**
1. Open the editor
2. Create a new file: `File → New File` → name it `hello.py`
3. Type:

```python
print("Hello from Cloud Shell Editor!")
for i in range(5):
    print(f"GCP Service #{i+1}")
```

4. Open a terminal: `Terminal → New Terminal`
5. Run: `python3 hello.py`

---

## Step 8 — Notifications & Activity

### Activity Feed:
1. Click **Navigation Menu → Home → Activity**
2. You see a timeline of every operation in this project
3. Filter options:
   - By resource type
   - By user/service account
   - By time range
4. Each entry shows: Who did What to Which resource at When

### Notifications Panel:
1. Click the 🔔 bell icon (top right)
2. Long-running operations (like creating a GKE cluster) post updates here when they complete
3. Billing alerts also appear here

---

## Step 9 — Console Keyboard Shortcuts

Press `?` anywhere in the console to see all shortcuts. Most useful ones:

| Shortcut | Action |
|---|---|
| `/` | Jump to search bar |
| `g + h` | Go to Home |
| `g + a` | Go to Activity |
| `g + s` | Go to Cloud Shell |
| `Ctrl + \`` | Toggle Cloud Shell |
| `g + n` | Go to Notifications |

---

## Step 10 — Google Cloud Mobile App (Optional)

GCP also has a mobile app:
- **Android:** Google Play → search "Google Cloud"
- **iOS:** App Store → search "Google Cloud"

The mobile app lets you:
- View resource status
- Start/stop VMs
- Read logs
- Get billing alerts
- SSH into VMs (basic)

---

## Orientation Checklist

Go through this list — check that you can do each without help:

### Navigation
- [ ] Open and close the left navigation menu
- [ ] Switch between projects using the project selector
- [ ] Use the search bar to find any GCP service
- [ ] Navigate back to Home by clicking the logo
- [ ] Pin and unpin services in the left menu

### Console Areas
- [ ] Identify all the sections in the left navigation menu
- [ ] Open the Billing dashboard and set a budget alert
- [ ] View the IAM page and understand principals and roles
- [ ] Open the API Library and find an API
- [ ] View enabled APIs for your project
- [ ] Open the Log Explorer and run a basic query

### Cloud Shell
- [ ] Open Cloud Shell from the top bar
- [ ] Run `gcloud auth list` and confirm your account
- [ ] Run `gcloud config get-value project` and confirm your project
- [ ] Open the Cloud Shell Editor and create a file
- [ ] Close and reopen Cloud Shell

### Monitoring & Ops
- [ ] Find the Activity feed for your project
- [ ] Open the Notifications panel
- [ ] Navigate to Cloud Monitoring → Dashboards
- [ ] Find the GCP Status page link

---

## Common Beginner Mistakes to Avoid

| Mistake | What Happens | How to Avoid |
|---|---|---|
| Wrong project selected | Can't find your resources | Always check the project dropdown first |
| API not enabled | "API not enabled" error | Go to APIs & Services → Library → Enable |
| Forgot to delete resources | Surprise bill at month end | Use the cleanup checklist at end of every lab |
| No budget alert set | Bill grows silently | Set a $10 budget alert on Day 0 |
| Using Owner role for everything | Security risk | Use minimum permissions needed |
| Sharing service account keys | Security breach | Use Workload Identity or Secret Manager |

---

## What's Next — Day 1 Preview

Now that you know the console, Day 1 will cover:
- Creating your first **VPC network** and understanding subnets
- Launching your first **Compute Engine VM** and SSH-ing in
- Creating a **Cloud Storage bucket** and uploading files
- Exploring **IAM roles** by creating a read-only service account

---

## Quick Reference Card

```
GCP CONSOLE — WHERE IS EVERYTHING?
────────────────────────────────────────────────────────
RUN CODE          Left Menu → COMPUTE section
STORE DATA        Left Menu → STORAGE or DATABASES
NETWORKING        Left Menu → NETWORKING section
MONITOR & LOGS    Left Menu → OPERATIONS section
BUILD & DEPLOY    Left Menu → TOOLS section
ANALYTICS         Left Menu → BIG DATA section
AI/ML             Left Menu → ARTIFICIAL INTELLIGENCE
ACCESS CONTROL    Left Menu → SECURITY → IAM & Admin
BILLING           Left Menu → BILLING
APIs              Left Menu → APIs & Services

FASTEST SHORTCUTS
  Search anything          Press /
  Open Cloud Shell         Click >_ in top bar
  Go Home                  Click Google Cloud logo
  Switch Project           Click project name dropdown
  View all logs            Logging → Log Explorer
  Check who has access     IAM & Admin → IAM
  See what you're spending Billing → Reports
────────────────────────────────────────────────────────
```

---

*Day 0 Lab — GCP Console Orientation | No resources created | No charges incurred*
