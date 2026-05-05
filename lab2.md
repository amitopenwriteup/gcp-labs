# GCP Session 2 — Lab Document
## Project Setup & Organization

> **Session:** 2 | **Level:** Beginner | **Estimated Time:** 90 minutes

---

## What You Will Learn

By the end of this lab, you will be able to:

- Understand how GCP organizes resources (Org → Folder → Project → Resource)
- Create and configure projects for dev, staging, and production
- Enable APIs and check quota limits
- Set up a billing account and link it to a project
- Create budget alerts so you don't get surprise bills
- Apply basic cost optimization strategies

---

## Before You Start

Make sure you have:

- [ ] A Google account (Gmail works)
- [ ] Access to [console.cloud.google.com](https://console.cloud.google.com)
- [ ] A credit/debit card (for billing setup — GCP gives $300 free credits to new accounts)
- [ ] Basic understanding of what a "cloud project" means

---

## Part 1 — GCP Resource Hierarchy

### What is the Resource Hierarchy?

GCP organizes everything into 4 levels. Think of it like folders inside folders on your computer.

```
Organization  (your whole company — e.g. mycompany.com)
    └── Folder  (a group of projects — e.g. Engineering, Finance)
            └── Project  (one app or environment — e.g. dev, prod)
                    └── Resource  (actual things — VMs, databases, storage)
```

**Key rule:** Permissions set at a higher level flow DOWN automatically.
If you give someone access at the Folder level, they automatically get access to all Projects inside that folder.

---

### Lab 1.1 — Explore the Resource Hierarchy

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Click the **project selector** at the top of the page (next to "Google Cloud")
3. You will see a panel showing your Organization, Folders, and Projects
4. Notice how they are nested inside each other

> **Note:** If you're using a personal Gmail account, you may not see an Organization — that's okay. Organizations are mainly used by companies with Google Workspace.

---

### Lab 1.2 — Naming Convention Practice

Use this naming pattern for all your resources:

```
{company}-{team}-{environment}-{app}
```

**Examples:**

| Resource | Bad Name | Good Name |
|---|---|---|
| Project | project1 | myco-eng-dev-api |
| Storage Bucket | bucket123 | myco-data-prod-logs |
| VM Instance | vm-new | myco-web-staging-server |

**Exercise:** Write down the names you would use for your own setup:

- Dev project name: `_______________________________`
- Staging project name: `_______________________________`
- Prod project name: `_______________________________`

---

### Lab 1.3 — Labels / Tags

Labels help you track costs and control access. Always add these labels to every resource you create:

| Label Key | Example Value | Why It Matters |
|---|---|---|
| `env` | `dev`, `staging`, `prod` | Know which environment |
| `team` | `engineering`, `data` | Know who owns it |
| `cost-center` | `123`, `finance` | Track costs by team |
| `app` | `api`, `frontend` | Know which application |

---

## Part 2 — Project Creation & Configuration

### Why Separate Projects per Environment?

Each environment (dev, staging, prod) should be its own GCP project. This means:

- A bug in dev **cannot** touch prod data
- You can give different people access to different environments
- You can set separate billing limits per environment
- It's easier to delete dev resources without risking prod

---

### Lab 2.1 — Create a Development Project

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Click the project selector at the top → click **New Project**
3. Fill in the details:
   - **Project name:** `myco-eng-dev` (use your own naming pattern)
   - **Organization:** Select your org if you have one
   - **Location:** Select a Folder if you have one, otherwise leave as No organization
4. Click **Create**
5. Wait about 30 seconds for the project to be created
6. Select the new project from the project selector

> **Tip:** Write down your Project ID — it's different from the project name and you'll need it later.
>
> My Project ID: `_______________________________`

---

### Lab 2.2 — Enable an API

APIs are GCP services. They are OFF by default — you have to turn them on before you can use them.

1. Make sure your dev project is selected
2. Go to **APIs & Services** in the left menu (or search "APIs" in the top search bar)
3. Click **Enable APIs and Services**
4. Search for **Compute Engine API**
5. Click on it, then click **Enable**
6. Wait about 30 seconds
7. You should now see Compute Engine in your enabled APIs list

**Repeat this** for:
- [ ] Cloud Storage API
- [ ] Cloud Run API
- [ ] Cloud Logging API

> **Common mistake:** Forgetting to enable an API before running code. If you get a 403 error saying "API not enabled", this is the fix.

---

### Lab 2.3 — Check Your Quotas

Quotas are limits GCP sets to protect against mistakes. For example, you can only create a certain number of VMs per region.

1. Go to **IAM & Admin** → **Quotas**
2. You will see a list of all services and their limits
3. Filter by service (e.g. "Compute Engine")
4. Look for any quota that is close to its limit

**What to do if you hit a quota:**
1. Click on the quota you want to increase
2. Click **Edit Quotas**
3. Fill in the form explaining why you need more
4. GCP usually responds within 2–3 business days

> **Note:** Most quota increases for dev work are approved automatically. Production increases may need more explanation.

---

## Part 3 — Billing & Cost Management

### Lab 3.1 — Set Up a Billing Account

> **Skip this lab if** you already have a billing account set up. Check at **Billing** in the left menu.

1. Go to **Billing** in the left menu
2. Click **Create Billing Account**
3. Enter your account name (e.g. "My Learning Account")
4. Choose your country
5. Enter your credit card details
6. Click **Submit and Enable Billing**

> **Note:** GCP gives new accounts **$300 in free credits** valid for 90 days. You won't be charged unless you upgrade your account OR use all your credits.

---

### Lab 3.2 — Link Billing to Your Project

1. Go to **Billing**
2. Click **My Projects** in the left sidebar
3. Find your dev project in the list
4. Click the 3-dot menu (⋮) on the right
5. Click **Change Billing**
6. Select your billing account
7. Click **Set Account**

**Verify it worked:**
- Go back to your dev project
- Try enabling any API (e.g. Cloud Translate)
- If it works, billing is correctly linked ✅
- If you get a billing error, go back and repeat the steps above

---

### Lab 3.3 — Create a Budget Alert

> **Important:** GCP will NOT automatically stop charging you when you reach a limit. Budget alerts only send you an email — you have to take action yourself.

1. Go to **Billing** → **Budgets & alerts**
2. Click **Create Budget**
3. Fill in the form:
   - **Name:** `dev-monthly-budget`
   - **Projects:** Select your dev project
   - **Budget type:** Specified amount
   - **Amount:** `$20` (or whatever you're comfortable with)
4. Click **Next**
5. Set up alert thresholds:
   - Click **Add Item** and set: `50%` → "Actual spend"
   - Click **Add Item** and set: `90%` → "Actual spend"
   - Click **Add Item** and set: `100%` → "Actual spend"
6. Make sure **Email alerts to billing admins and users** is checked
7. Click **Finish**

**Result:** You will now get an email when you hit 50%, 90%, and 100% of your $20 limit.

> **Tip:** Set a budget for EACH project separately (dev, staging, prod). Prod should have a higher limit than dev.

---

### Lab 3.4 — Cost Optimization Checklist

Run through this checklist once a month to keep your bill low:

#### Quick Wins (Do These Now)

- [ ] **Delete unused VMs** — Go to Compute Engine and stop/delete any VMs you're not using. Stopped VMs still charge for disk storage.
- [ ] **Release unused IP addresses** — Go to VPC Network → IP Addresses. Any "In use: No" IPs cost ~$7/month each.
- [ ] **Delete old snapshots** — Go to Compute Engine → Snapshots. Delete any older than 30 days that you don't need.

#### Medium Term

- [ ] **Use Preemptible / Spot VMs for dev work** — Up to 80% cheaper. GCP can shut them down anytime, so only use for batch jobs or testing.
  - When creating a VM, go to **Availability policy** → set to **Spot**
- [ ] **Right-size your VMs** — Go to **Recommender** → **VM Machine Type** → follow the suggestions to downsize over-powered VMs
- [ ] **Choose the right region** — US regions (us-central1, us-east1) are usually 20–30% cheaper than EU or Asia regions

#### Long Term (When Running Production Workloads)

- [ ] **Committed Use Discounts (CUD)** — If you know a VM will run 24/7 for 1+ years, commit to it for 30–55% savings. Go to Compute Engine → Committed Use Discounts.
- [ ] **Export billing to BigQuery** — Go to Billing → Billing Export → BigQuery Export. This lets you run SQL queries on your spending data to find exactly what's costing the most.

---

## Troubleshooting Common Errors

| Error | What It Means | How to Fix |
|---|---|---|
| `403 - API not enabled` | The API you're trying to use is turned off | Enable it in APIs & Services |
| `429 - Quota exceeded` | You've hit a usage limit | Request a quota increase in IAM & Admin → Quotas |
| `403 - Billing not enabled` | No billing account linked | Link billing in Billing → My Projects |
| `400 - Project name taken` | Project IDs must be globally unique | Add more words to your project name |
| Budget alert not arriving | Email went to spam or wrong address | Check billing account email settings |

---

## Summary

| Topic | Key Point |
|---|---|
| Resource Hierarchy | Org → Folder → Project → Resource. Permissions flow downward. |
| Naming | Use `company-team-env-app` pattern everywhere. |
| Environments | Dev, Staging, Prod must be separate projects. Never mix them. |
| APIs | Always enable APIs before using a service. They are off by default. |
| Billing | Link billing to each project. GCP won't work without it. |
| Budget Alerts | Set at 50%, 90%, 100%. They notify only — they don't stop spending. |
| Cost Saving | Delete unused resources monthly. Use Spot VMs for dev and testing. |

---

## What's Next

**Session 3 → IAM & Security**
- Identity and Access Management (IAM)
- Roles and Permissions
- Service Accounts
- Best practices for access control

---

*GCP Session 2 Lab Document | Beginner Friendly*
