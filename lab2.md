# GCP Session 2 - Lab Document
## Project Setup and Organization

Session: 2 | Level: Beginner | Estimated Time: 90 minutes

---

## What You Will Learn

By the end of this lab, you will be able to:

- Understand how GCP organizes resources (Org to Folder to Project to Resource)
- Create a project using your own name
- Add sample labels to your project resources
- Enable commonly used APIs
- Set up a billing account and link it to your project
- Create a $10 budget alert with email notifications enabled

---

## Before You Start

Make sure you have:

- A Google account (Gmail works)
- Access to console.cloud.google.com
- A credit/debit card (for billing setup - GCP gives $300 free credits to new accounts)
- Basic understanding of what a "cloud project" means

---

## Part 1 - GCP Resource Hierarchy

### What is the Resource Hierarchy?

GCP organizes everything into 4 levels. Think of it like folders inside folders on your computer.

```
Organization  (your whole company - e.g. mycompany.com)
    └── Folder  (a group of projects - e.g. Engineering, Finance)
            └── Project  (one app or environment - e.g. dev, prod)
                    └── Resource  (actual things - VMs, databases, storage)
```

Key rule: Permissions set at a higher level flow DOWN automatically. If you give someone access at the Folder level, they automatically get access to all Projects inside that folder.

---

### Lab 1.1 - Explore the Resource Hierarchy

1. Go to console.cloud.google.com
2. Click the project selector at the top of the page (next to "Google Cloud")
3. You will see a panel showing your Organization, Folders, and Projects
4. Notice how they are nested inside each other

Note: If you are using a personal Gmail account, you will not see an Organization. That is expected. We will work without an organization in this lab.

---

## Part 2 - Create Your Project

### Lab 2.1 - Create a Project Using Your Name

Each participant will create their own GCP project using their name so it is easy to identify during the lab.

Naming format:

```
{yourname}-lab-dev
```

Examples:

| Your Name | Project Name |
|---|---|
| Rahul Sharma | rahulsharma-lab-dev |
| Priya Singh | priyasingh-lab-dev |
| Amit Kumar | amitkumar-lab-dev |

Rule: Use only lowercase letters, numbers, and hyphens. No spaces or special characters.

Steps:

1. Go to console.cloud.google.com
2. Click the project selector at the top and click New Project
3. Fill in the details:
   - Project name: {yourname}-lab-dev (replace with your actual name)
   - Organization: Leave as No organization
   - Location: Leave as No organization
4. Click Create
5. Wait about 30 seconds for the project to be created
6. Select your new project from the project selector

Important: Write down your Project ID. It is auto-generated and may be slightly different from the name. You will need it later.

My Project Name: ___________________________

My Project ID: ___________________________

---

### Lab 2.2 - Add Sample Labels to Your Project

Labels help you organize resources and track costs. You will now add sample labels to your project.

Apply these labels when creating any resource:

| Label Key | Value | Why It Matters |
|---|---|---|
| env | dev | Identifies this as a development environment |
| owner | yourname | Identifies you as the owner |
| team | lab-batch-1 | Identifies the lab group |
| app | gcp-lab | Identifies this as a lab project |

How to add labels to your project:

1. In the GCP Console, go to IAM and Admin then Labels (or search Labels in the top bar)
2. Click Add Label
3. Add each label from the table above:
   - Key: env | Value: dev
   - Key: owner | Value: yourname (your actual name in lowercase)
   - Key: team | Value: lab-batch-1
   - Key: app | Value: gcp-lab
4. Click Save

Note: Every time you create a VM, storage bucket, or any resource in this lab, scroll down to the Labels section and add these same labels. This is a real-world best practice.

Write your label values below:

- env = ___________________________
- owner = ___________________________
- team = ___________________________
- app = ___________________________

---

## Part 3 - Enable APIs

APIs are GCP services. They are off by default. You must turn them on before you can use them.

### Lab 3.1 - Enable the Required APIs

1. Make sure your project ({yourname}-lab-dev) is selected at the top
2. Go to APIs and Services in the left menu (or search APIs in the top search bar)
3. Click Enable APIs and Services
4. Search for and enable each API below:

| API Name | What It Does |
|---|---|
| Compute Engine API | Create and manage virtual machines |
| Cloud Storage API | Store files and objects |
| Cloud Run API | Deploy containerized apps |
| Cloud Logging API | View and store logs |

For each API: Search the name, click on it, click Enable, and wait about 30 seconds.

Track your progress:

- Compute Engine API - done / not done
- Cloud Storage API - done / not done
- Cloud Run API - done / not done
- Cloud Logging API - done / not done

Common mistake: Forgetting to enable an API before running code. If you get a 403 API not enabled error, come back here and enable the missing API.

---

## Part 4 - Billing and Budget Alert

### Lab 4.1 - Set Up a Billing Account

Skip this step if you already have a billing account. Check under Billing in the left menu.

1. Go to Billing in the left menu
2. Click Create Billing Account
3. Enter a name: {yourname} Learning Account
4. Choose your country
5. Enter your credit/debit card details
6. Click Submit and Enable Billing

Note: GCP gives new accounts $300 in free credits valid for 90 days. You will not be charged unless you upgrade or use all your credits.

---

### Lab 4.2 - Link Billing to Your Project

1. Go to Billing then My Projects (left sidebar)
2. Find your project ({yourname}-lab-dev) in the list
3. Click the three-dot menu on the right
4. Click Change Billing
5. Select your billing account
6. Click Set Account

Verify it worked:

- Go back to your project
- Try enabling any API such as Cloud Translate API
- If it enables successfully, billing is correctly linked
- If you see a billing error, repeat the steps above

---

### Lab 4.3 - Create a $10 Budget Alert with Notifications

Important: GCP will not automatically stop charging you when you hit a limit. Budget alerts only send an email. You must take action yourself.

Steps:

1. Go to Billing then Budgets and alerts
2. Click Create Budget
3. Fill in the form:
   - Name: {yourname}-lab-budget
   - Projects: Select your project ({yourname}-lab-dev)
   - Services: Leave as All services
   - Budget type: Specified amount
   - Amount: $10
4. Click Next
5. Set the alert thresholds by clicking Add Item for each:

| Threshold | Type | What Happens |
|---|---|---|
| 50% | Actual spend | Email sent when you spend $5 |
| 90% | Actual spend | Email sent when you spend $9 |
| 100% | Actual spend | Email sent when you spend $10 |

6. Under Manage notifications:
   - Check the box: Email alerts to billing admins and users
   - This ensures you receive the alert at your Google account email
7. Click Finish

Verify your budget was created:

- You should now see {yourname}-lab-budget listed under Budgets and alerts
- The amount should show $10.00

Tip: Check your Gmail inbox. GCP may send a confirmation email when the budget is created.

---

## Troubleshooting Common Errors

| Error | What It Means | How to Fix |
|---|---|---|
| 403 - API not enabled | The API you are trying to use is turned off | Enable it in APIs and Services |
| 403 - Billing not enabled | No billing account linked to the project | Link billing in Billing then My Projects |
| 400 - Project name taken | Project IDs must be globally unique | Add numbers after your name e.g. rahul-lab-dev-01 |
| Budget alert email not received | Email went to spam | Check your spam or promotions folder |
| Labels not saving | Missing required fields | Make sure both Key and Value are filled |

---

## Summary

| Task | What You Did |
|---|---|
| Created a Project | Named it {yourname}-lab-dev using your own name with no organization |
| Added Labels | Applied env, owner, team, app labels |
| Enabled APIs | Compute Engine, Cloud Storage, Cloud Run, Cloud Logging |
| Linked Billing | Connected billing account to your project |
| Created Budget Alert | Set $10 limit with alerts at 50%, 90%, and 100% with email notifications on |

---

## What's Next

Session 3 - IAM and Security

- Identity and Access Management (IAM)
- Roles and Permissions
- Service Accounts
- Best practices for access control

---

GCP Session 2 Lab Document | Beginner Friendly
