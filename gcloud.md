# GCP IAM Fundamentals - gcloud Command Lab
### Migration Project: Identity and Access Management

---

> **Tool:** gcloud CLI
> **Format:** Individual commands only - no shell scripting
> **Duration:** 3 to 4 Hours
> **Access:** Google Cloud Shell at shell.cloud.google.com

---

## Before You Begin

Open Google Cloud Shell and run these commands one at a time to set up your environment.

### Set Your Project

```
gcloud config set project YOUR_PROJECT_ID
```

### Confirm Active Project

```
gcloud config get-value project
```

### Enable Required APIs

```
gcloud services enable iam.googleapis.com
```

```
gcloud services enable iamcredentials.googleapis.com
```

```
gcloud services enable cloudresourcemanager.googleapis.com
```

```
gcloud services enable recommender.googleapis.com
```

```
gcloud services enable secretmanager.googleapis.com
```

```
gcloud services enable storage.googleapis.com
```

```
gcloud services enable bigquery.googleapis.com
```

### Verify APIs Are Enabled

```
gcloud services list --enabled --filter="name:iam.googleapis.com"
```

```
gcloud services list --enabled --filter="name:recommender.googleapis.com"
```

---

---

# Module 1 - IAM Fundamentals

---

## Lab 1.1 - Principals, Roles and Permissions

**Objective:** View and manage IAM policy bindings using gcloud commands.

---

### View the Full IAM Policy for Your Project

```
gcloud projects get-iam-policy YOUR_PROJECT_ID
```

### View IAM Policy in JSON Format

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --format=json
```

### View IAM Policy in YAML Format

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --format=yaml
```

### List All Principals and Their Roles in a Table

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --flatten="bindings[].members" --format="table(bindings.role,bindings.members)"
```

### Filter IAM Policy to Show Only Owner Role Bindings

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --flatten="bindings[].members" --format="table(bindings.role,bindings.members)" --filter="bindings.role:roles/owner"
```

### Filter IAM Policy to Show Only Editor Role Bindings

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --flatten="bindings[].members" --format="table(bindings.role,bindings.members)" --filter="bindings.role:roles/editor"
```

### Filter IAM Policy to Show Only Viewer Role Bindings

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --flatten="bindings[].members" --format="table(bindings.role,bindings.members)" --filter="bindings.role:roles/viewer"
```

### Grant a Role to a User

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="user:ANALYST_EMAIL@gmail.com" --role="roles/bigquery.dataViewer"
```

### Verify the Binding Was Created

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --flatten="bindings[].members" --format="table(bindings.role,bindings.members)" --filter="bindings.members:ANALYST_EMAIL"
```

### Remove a Role from a User

```
gcloud projects remove-iam-policy-binding YOUR_PROJECT_ID --member="user:ANALYST_EMAIL@gmail.com" --role="roles/bigquery.dataViewer"
```

### Test Permissions Available to the Current User

```
gcloud projects test-iam-permissions YOUR_PROJECT_ID --permissions="storage.buckets.get,storage.objects.list,bigquery.tables.get"
```

---

### Checkpoint

Run this command and confirm you can see your own account and its roles in the output.

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --flatten="bindings[].members" --format="table(bindings.role,bindings.members)" --filter="bindings.members:YOUR_EMAIL"
```

---

---

## Lab 1.2 - Predefined vs Custom Roles

**Objective:** Explore predefined roles and create a custom role for the migration project.

---

### List All Predefined Roles in GCP

```
gcloud iam roles list
```

### List Only Custom Roles in Your Project

```
gcloud iam roles list --project=YOUR_PROJECT_ID
```

### Describe the Storage Admin Predefined Role

```
gcloud iam roles describe roles/storage.admin
```

### Describe the Storage Object Viewer Role

```
gcloud iam roles describe roles/storage.objectViewer
```

### Describe the BigQuery Data Editor Role

```
gcloud iam roles describe roles/bigquery.dataEditor
```

### Describe the BigQuery Data Viewer Role

```
gcloud iam roles describe roles/bigquery.dataViewer
```

### List Available Permissions for the Storage Service

```
gcloud iam list-testable-permissions //cloudresourcemanager.googleapis.com/projects/YOUR_PROJECT_ID --filter="name:storage"
```

### List Available Permissions for the BigQuery Service

```
gcloud iam list-testable-permissions //cloudresourcemanager.googleapis.com/projects/YOUR_PROJECT_ID --filter="name:bigquery"
```

---

### Create a Custom Migration Analyst Role

Create a file named `migration-analyst-role.yaml` with the content below.

```
gcloud iam roles create migrationAnalyst --project=YOUR_PROJECT_ID --title="Migration Analyst" --description="Read-only access to migration resources" --stage="GA" --permissions="bigquery.datasets.get,bigquery.tables.get,bigquery.tables.list,storage.buckets.get,storage.buckets.list,storage.objects.get,storage.objects.list,cloudsql.databases.get,cloudsql.databases.list"
```

### Verify the Custom Role Was Created

```
gcloud iam roles describe migrationAnalyst --project=YOUR_PROJECT_ID
```

### List All Custom Roles in the Project

```
gcloud iam roles list --project=YOUR_PROJECT_ID --format="table(name,title,stage)"
```

### Update the Custom Role - Add Compute Permissions

```
gcloud iam roles update migrationAnalyst --project=YOUR_PROJECT_ID --add-permissions="compute.instances.get,compute.instances.list,compute.disks.get,compute.disks.list"
```

### Verify the Updated Permissions

```
gcloud iam roles describe migrationAnalyst --project=YOUR_PROJECT_ID --format="value(includedPermissions)"
```

### Assign the Custom Role to a User

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="user:ANALYST_EMAIL@gmail.com" --role="projects/YOUR_PROJECT_ID/roles/migrationAnalyst"
```

### Disable a Custom Role

```
gcloud iam roles update migrationAnalyst --project=YOUR_PROJECT_ID --stage="DISABLED"
```

### Re-enable a Custom Role

```
gcloud iam roles update migrationAnalyst --project=YOUR_PROJECT_ID --stage="GA"
```

### Delete a Custom Role

```
gcloud iam roles delete migrationAnalyst --project=YOUR_PROJECT_ID
```

### Undelete a Custom Role

```
gcloud iam roles undelete migrationAnalyst --project=YOUR_PROJECT_ID
```

---

### Checkpoint

Run this command and confirm the custom role shows all expected permissions.

```
gcloud iam roles describe migrationAnalyst --project=YOUR_PROJECT_ID --format="table(title,stage,includedPermissions)"
```

---

---

## Lab 1.3 - Policy Inheritance and Evaluation

**Objective:** Observe how IAM policies work at different resource levels.

---

### View Project-Level IAM Policy

```
gcloud projects get-iam-policy YOUR_PROJECT_ID
```

### View IAM Policy for a Specific Folder

```
gcloud resource-manager folders get-iam-policy FOLDER_ID
```

### View IAM Policy for the Organization

```
gcloud organizations get-iam-policy ORGANIZATION_ID
```

### Create a Cloud Storage Bucket for This Lab

```
gcloud storage buckets create gs://migration-lab-bucket-YOUR_PROJECT_ID --location=asia-south1 --project=YOUR_PROJECT_ID
```

### View the Bucket-Level IAM Policy

```
gcloud storage buckets get-iam-policy gs://migration-lab-bucket-YOUR_PROJECT_ID
```

### Add a Role Binding at the Bucket Level Only

```
gcloud storage buckets add-iam-policy-binding gs://migration-lab-bucket-YOUR_PROJECT_ID --member="user:TEST_EMAIL@gmail.com" --role="roles/storage.objectViewer"
```

### Verify the Bucket-Level Binding

```
gcloud storage buckets get-iam-policy gs://migration-lab-bucket-YOUR_PROJECT_ID
```

### Remove a Binding from the Bucket Level

```
gcloud storage buckets remove-iam-policy-binding gs://migration-lab-bucket-YOUR_PROJECT_ID --member="user:TEST_EMAIL@gmail.com" --role="roles/storage.objectViewer"
```

### Check What Permissions a Principal Has on a Resource

```
gcloud projects test-iam-permissions YOUR_PROJECT_ID --permissions="storage.buckets.get,storage.buckets.list,storage.objects.create,bigquery.tables.get"
```

### List All Bindings Containing a Specific Member

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --flatten="bindings[].members" --format="table(bindings.role,bindings.members)" --filter="bindings.members:TEST_EMAIL"
```

---

### Checkpoint

Run this command and confirm the bucket-level policy is separate from the project policy.

```
gcloud storage buckets get-iam-policy gs://migration-lab-bucket-YOUR_PROJECT_ID --format="table(bindings.role,bindings.members)"
```

---

---

# Module 2 - Service Accounts

---

## Lab 2.1 - Service Account Types and Use Cases

**Objective:** Create and configure service accounts for each migration workload.

---

### List All Existing Service Accounts

```
gcloud iam service-accounts list --project=YOUR_PROJECT_ID
```

### List Service Accounts in Table Format

```
gcloud iam service-accounts list --project=YOUR_PROJECT_ID --format="table(displayName,email,disabled)"
```

### Create the Migration Reader Service Account

```
gcloud iam service-accounts create migration-reader --display-name="Migration Reader" --description="Reads source data during migration" --project=YOUR_PROJECT_ID
```

### Create the Migration Writer Service Account

```
gcloud iam service-accounts create migration-writer --display-name="Migration Writer" --description="Writes transformed data to destination" --project=YOUR_PROJECT_ID
```

### Create the Migration Orchestrator Service Account

```
gcloud iam service-accounts create migration-orchestrator --display-name="Migration Orchestrator" --description="Orchestrates migration pipelines" --project=YOUR_PROJECT_ID
```

### Create the Migration Audit Service Account

```
gcloud iam service-accounts create migration-audit --display-name="Migration Audit" --description="Writes migration logs and monitoring metrics" --project=YOUR_PROJECT_ID
```

### Verify All Migration Service Accounts Exist

```
gcloud iam service-accounts list --project=YOUR_PROJECT_ID --filter="email:migration-" --format="table(displayName,email)"
```

### Describe a Specific Service Account

```
gcloud iam service-accounts describe migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### Grant BigQuery Data Viewer to Migration Reader

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/bigquery.dataViewer"
```

### Grant Storage Object Viewer to Migration Reader

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/storage.objectViewer"
```

### Grant BigQuery Data Editor to Migration Writer

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/bigquery.dataEditor"
```

### Grant Storage Object Creator to Migration Writer

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/storage.objectCreator"
```

### Grant Dataflow Admin to Migration Orchestrator

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/dataflow.admin"
```

### Grant Service Account User to Migration Orchestrator

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/iam.serviceAccountUser"
```

### Grant Logs Writer to Migration Audit

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-audit@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/logging.logWriter"
```

### Grant Monitoring Metric Writer to Migration Audit

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-audit@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/monitoring.metricWriter"
```

### View All Roles Granted to Migration Reader

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --flatten="bindings[].members" --format="table(bindings.role,bindings.members)" --filter="bindings.members:migration-reader"
```

### View All Roles Granted to Migration Writer

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --flatten="bindings[].members" --format="table(bindings.role,bindings.members)" --filter="bindings.members:migration-writer"
```

### View IAM Policy on a Specific Service Account (Who Can Use It)

```
gcloud iam service-accounts get-iam-policy migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### Disable a Service Account

```
gcloud iam service-accounts disable migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### Re-enable a Service Account

```
gcloud iam service-accounts enable migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

---

### Checkpoint

Run this command and confirm all 4 migration service accounts appear.

```
gcloud iam service-accounts list --project=YOUR_PROJECT_ID --filter="email:migration-" --format="table(displayName,email,disabled)"
```

---

---

## Lab 2.2 - Key Management and Rotation

**Objective:** Create, list, rotate, and delete service account keys through gcloud commands.

---

### List Keys for a Service Account

```
gcloud iam service-accounts keys list --iam-account=migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### List Keys in Table Format

```
gcloud iam service-accounts keys list --iam-account=migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com --format="table(name.basename(),validAfterTime,validBeforeTime,keyType)"
```

### Create a New Service Account Key

```
gcloud iam service-accounts keys create migration-orchestrator-key.json --iam-account=migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### Verify the Key File Was Created

```
gcloud iam service-accounts keys list --iam-account=migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com --format="table(name.basename(),validAfterTime,keyType)"
```

### Store the Key in Secret Manager - Create the Secret

```
gcloud secrets create migration-orchestrator-sa-key --replication-policy="automatic" --project=YOUR_PROJECT_ID
```

### Add the Key File as a Secret Version

```
gcloud secrets versions add migration-orchestrator-sa-key --data-file=migration-orchestrator-key.json --project=YOUR_PROJECT_ID
```

### Verify the Secret Was Stored

```
gcloud secrets list --project=YOUR_PROJECT_ID --filter="name:migration-orchestrator"
```

### View Secret Version Metadata

```
gcloud secrets versions list migration-orchestrator-sa-key --project=YOUR_PROJECT_ID
```

### Rotate - Create a New Key

```
gcloud iam service-accounts keys create migration-orchestrator-key-new.json --iam-account=migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### List All Keys to Identify the Old One

```
gcloud iam service-accounts keys list --iam-account=migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com --format="table(name.basename(),validAfterTime,keyType)"
```

### Delete the Old Key by Key ID

Replace KEY_ID with the old key ID shown in the list above.

```
gcloud iam service-accounts keys delete KEY_ID --iam-account=migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### Verify Only the New Key Remains

```
gcloud iam service-accounts keys list --iam-account=migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com --format="table(name.basename(),validAfterTime,keyType)"
```

### Delete the Local Key Files After Secure Storage

```
gcloud secrets versions access latest --secret="migration-orchestrator-sa-key" --project=YOUR_PROJECT_ID
```

### List All Service Accounts and Check for Keys

```
gcloud iam service-accounts keys list --iam-account=migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com --filter="keyType=USER_MANAGED"
```

```
gcloud iam service-accounts keys list --iam-account=migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com --filter="keyType=USER_MANAGED"
```

```
gcloud iam service-accounts keys list --iam-account=migration-audit@YOUR_PROJECT_ID.iam.gserviceaccount.com --filter="keyType=USER_MANAGED"
```

---

### Checkpoint

Run this command and confirm only one user-managed key exists for the orchestrator SA.

```
gcloud iam service-accounts keys list --iam-account=migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com --filter="keyType=USER_MANAGED" --format="table(name.basename(),validAfterTime)"
```

---

---

## Lab 2.3 - Service Account Impersonation

**Objective:** Configure and test service account impersonation using gcloud commands.

---

### View Who Can Currently Impersonate Migration Reader

```
gcloud iam service-accounts get-iam-policy migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### Grant Your User the Token Creator Role on Migration Reader

Replace YOUR_EMAIL with your own Google account.

```
gcloud iam service-accounts add-iam-policy-binding migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com --member="user:YOUR_EMAIL@gmail.com" --role="roles/iam.serviceAccountTokenCreator"
```

### Grant Orchestrator the Token Creator Role on Migration Reader

```
gcloud iam service-accounts add-iam-policy-binding migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com --member="serviceAccount:migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/iam.serviceAccountTokenCreator"
```

### Grant Orchestrator Token Creator on Migration Writer

```
gcloud iam service-accounts add-iam-policy-binding migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com --member="serviceAccount:migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/iam.serviceAccountTokenCreator"
```

### Grant Orchestrator Token Creator on Migration Audit

```
gcloud iam service-accounts add-iam-policy-binding migration-audit@YOUR_PROJECT_ID.iam.gserviceaccount.com --member="serviceAccount:migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/iam.serviceAccountTokenCreator"
```

### Verify Impersonation Policy on Migration Reader

```
gcloud iam service-accounts get-iam-policy migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com --format="table(bindings.role,bindings.members)"
```

### Impersonate Migration Reader to List Storage Buckets

```
gcloud storage buckets list --project=YOUR_PROJECT_ID --impersonate-service-account=migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### Generate a Short-Lived Token for Migration Reader

```
gcloud auth print-access-token --impersonate-service-account=migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### Run a BigQuery Command as Migration Reader

```
gcloud bigquery datasets list --project=YOUR_PROJECT_ID --impersonate-service-account=migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### Verify the Token Creator Binding on Migration Writer

```
gcloud iam service-accounts get-iam-policy migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com --format="table(bindings.role,bindings.members)"
```

### Remove an Impersonation Binding

```
gcloud iam service-accounts remove-iam-policy-binding migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com --member="user:YOUR_EMAIL@gmail.com" --role="roles/iam.serviceAccountTokenCreator"
```

---

### Checkpoint

Run this command and confirm the orchestrator SA is listed as a Token Creator on migration-reader.

```
gcloud iam service-accounts get-iam-policy migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

---

---

# Module 3 - Advanced IAM Concepts

---

## Lab 3.1 - Conditional Access Policies

**Objective:** Add IAM conditions to role bindings to restrict access by time, resource, and expiry.

---

### View Existing Bindings with Conditions

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --format=json
```

### Grant a Role with a Time-Based Condition (Business Hours IST)

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/storage.objectCreator" --condition=title="Migration Window",description="Allow writes only during migration hours 2AM to 6AM IST weekdays",expression="request.time.getHours('Asia/Kolkata') >= 2 && request.time.getHours('Asia/Kolkata') <= 6 && request.time.getDayOfWeek('Asia/Kolkata') >= 1 && request.time.getDayOfWeek('Asia/Kolkata') <= 5"
```

### Grant a Role with a Resource Name Condition

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/storage.objectViewer" --condition=title="Migration Bucket Only",description="Restrict access to migration bucket",expression="resource.name.startsWith('projects/_/buckets/migration-lab-bucket')"
```

### Grant a Role with an Expiry Date Condition

Replace 2025-12-31 with your desired expiry date.

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="user:TEMP_USER@gmail.com" --role="roles/bigquery.dataViewer" --condition=title="Temporary Migration Access",description="Expires after migration is complete",expression="request.time < timestamp('2025-12-31T00:00:00Z')"
```

### View All Bindings with Conditions in JSON

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --format=json --flatten="bindings[].members" --filter="bindings.condition:*"
```

### Remove a Conditional Binding

```
gcloud projects remove-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/storage.objectCreator" --condition=title="Migration Window"
```

### Test Access for a Service Account

```
gcloud projects test-iam-permissions YOUR_PROJECT_ID --permissions="storage.objects.create" --impersonate-service-account=migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### List All Bindings for the Migration Writer SA

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --flatten="bindings[].members" --format="table(bindings.role,bindings.members,bindings.condition.title)" --filter="bindings.members:migration-writer"
```

---

### Checkpoint

Run this command and confirm the condition title appears in the output for migration-writer.

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --flatten="bindings[].members" --format="table(bindings.role,bindings.condition.title)" --filter="bindings.members:migration-writer"
```

---

---

## Lab 3.2 - IAM Recommender and Security Insights

**Objective:** Retrieve and review IAM recommendations and policy insights using gcloud.

---

### List All IAM Recommendations for the Project

```
gcloud recommender recommendations list --project=YOUR_PROJECT_ID --location=global --recommender=google.iam.policy.Recommender
```

### List IAM Recommendations in Table Format

```
gcloud recommender recommendations list --project=YOUR_PROJECT_ID --location=global --recommender=google.iam.policy.Recommender --format="table(name.basename(),stateInfo.state,description)"
```

### Describe a Specific Recommendation

Replace RECOMMENDATION_ID with the ID from the list output above.

```
gcloud recommender recommendations describe RECOMMENDATION_ID --project=YOUR_PROJECT_ID --location=global --recommender=google.iam.policy.Recommender
```

### Mark a Recommendation as Dismissed

```
gcloud recommender recommendations dismiss RECOMMENDATION_ID --project=YOUR_PROJECT_ID --location=global --recommender=google.iam.policy.Recommender --etag=ETAG_FROM_DESCRIBE
```

### List IAM Policy Insights

```
gcloud recommender insights list --project=YOUR_PROJECT_ID --location=global --insight-type=google.iam.policy.Insight
```

### List IAM Policy Insights in Table Format

```
gcloud recommender insights list --project=YOUR_PROJECT_ID --location=global --insight-type=google.iam.policy.Insight --format="table(name.basename(),stateInfo.state,description)"
```

### Describe a Specific Insight

Replace INSIGHT_ID with the ID from the list output above.

```
gcloud recommender insights describe INSIGHT_ID --project=YOUR_PROJECT_ID --location=global --insight-type=google.iam.policy.Insight
```

### List Service Account Key Insights

```
gcloud recommender insights list --project=YOUR_PROJECT_ID --location=global --insight-type=google.iam.serviceAccount.Insight
```

### List Security Findings for the Project

```
gcloud scc findings list YOUR_PROJECT_ID --filter="category:IAM"
```

---

### Checkpoint

Run this command and confirm it executes without error. New projects will return an empty list until 90 days of usage data is collected.

```
gcloud recommender recommendations list --project=YOUR_PROJECT_ID --location=global --recommender=google.iam.policy.Recommender --format="table(name.basename(),stateInfo.state)"
```

---

---

## Lab 3.3 - Organization Policies and Constraints

**Objective:** View, set, and manage organization policies for the migration project.

---

### List All Organization Policy Constraints Available

```
gcloud resource-manager org-policies list-available-constraints --project=YOUR_PROJECT_ID
```

### List Constraints Related to IAM

```
gcloud resource-manager org-policies list-available-constraints --project=YOUR_PROJECT_ID --filter="name:iam"
```

### List Constraints Related to Storage

```
gcloud resource-manager org-policies list-available-constraints --project=YOUR_PROJECT_ID --filter="name:storage"
```

### View the Current Policy for Uniform Bucket Access

```
gcloud resource-manager org-policies describe constraints/storage.uniformBucketLevelAccess --project=YOUR_PROJECT_ID
```

### Enforce Uniform Bucket Level Access

```
gcloud resource-manager org-policies enable-enforce constraints/storage.uniformBucketLevelAccess --project=YOUR_PROJECT_ID
```

### Verify the Policy is Enforced

```
gcloud resource-manager org-policies describe constraints/storage.uniformBucketLevelAccess --project=YOUR_PROJECT_ID
```

### View the Current Resource Location Policy

```
gcloud resource-manager org-policies describe constraints/gcp.resourceLocations --project=YOUR_PROJECT_ID
```

### View the Service Account Key Creation Policy

```
gcloud resource-manager org-policies describe constraints/iam.disableServiceAccountKeyCreation --project=YOUR_PROJECT_ID
```

### Enforce - Disable Service Account Key Creation

```
gcloud resource-manager org-policies enable-enforce constraints/iam.disableServiceAccountKeyCreation --project=YOUR_PROJECT_ID
```

### Verify the Key Creation Policy

```
gcloud resource-manager org-policies describe constraints/iam.disableServiceAccountKeyCreation --project=YOUR_PROJECT_ID
```

### Disable Enforcement - Allow Key Creation Again

```
gcloud resource-manager org-policies disable-enforce constraints/iam.disableServiceAccountKeyCreation --project=YOUR_PROJECT_ID
```

### List All Active Policies on the Project

```
gcloud resource-manager org-policies list --project=YOUR_PROJECT_ID
```

### List Active Policies in Table Format

```
gcloud resource-manager org-policies list --project=YOUR_PROJECT_ID --format="table(constraint,updateTime)"
```

### Delete a Custom Org Policy (Revert to Inherited)

```
gcloud resource-manager org-policies delete constraints/storage.uniformBucketLevelAccess --project=YOUR_PROJECT_ID
```

---

### Checkpoint

Run this command and confirm at least one active policy appears.

```
gcloud resource-manager org-policies list --project=YOUR_PROJECT_ID --format="table(constraint,updateTime)"
```

---

---

# Module 4 - Migration Project Full IAM Setup

---

## Lab 4.1 - End-to-End IAM Configuration

**Objective:** Apply all previous concepts to configure the complete IAM setup for the migration project.

---

### Step 1 - Create All Migration Service Accounts

```
gcloud iam service-accounts create migration-reader --display-name="Migration Reader" --project=YOUR_PROJECT_ID
```

```
gcloud iam service-accounts create migration-writer --display-name="Migration Writer" --project=YOUR_PROJECT_ID
```

```
gcloud iam service-accounts create migration-orchestrator --display-name="Migration Orchestrator" --project=YOUR_PROJECT_ID
```

```
gcloud iam service-accounts create migration-audit --display-name="Migration Audit" --project=YOUR_PROJECT_ID
```

```
gcloud iam service-accounts create migration-validator --display-name="Migration Validator" --project=YOUR_PROJECT_ID
```

### Step 2 - Create the Security Reviewer Custom Role

```
gcloud iam roles create MigrationSecurityReviewer --project=YOUR_PROJECT_ID --title="Migration Security Reviewer" --description="Read-only security and IAM audit access" --stage="GA" --permissions="iam.roles.get,iam.roles.list,iam.serviceAccounts.get,iam.serviceAccounts.list,iam.serviceAccounts.getIamPolicy,resourcemanager.projects.getIamPolicy,logging.logEntries.list,logging.logs.list,storage.buckets.getIamPolicy,storage.buckets.list,bigquery.datasets.getIamPolicy,bigquery.datasets.get,recommender.iamPolicyRecommendations.get,recommender.iamPolicyRecommendations.list"
```

### Step 3 - Grant Roles to Migration Reader

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/bigquery.dataViewer"
```

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/storage.objectViewer"
```

### Step 4 - Grant Roles to Migration Writer

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/bigquery.dataEditor"
```

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/storage.objectCreator"
```

### Step 5 - Grant Roles to Migration Orchestrator

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/dataflow.admin"
```

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/iam.serviceAccountUser"
```

### Step 6 - Grant Roles to Migration Audit

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-audit@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/logging.logWriter"
```

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-audit@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/monitoring.metricWriter"
```

### Step 7 - Grant Roles to Migration Validator

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-validator@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/bigquery.dataViewer"
```

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-validator@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/storage.objectViewer"
```

### Step 8 - Configure Impersonation - Orchestrator to Reader

```
gcloud iam service-accounts add-iam-policy-binding migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com --member="serviceAccount:migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/iam.serviceAccountTokenCreator"
```

### Step 9 - Configure Impersonation - Orchestrator to Writer

```
gcloud iam service-accounts add-iam-policy-binding migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com --member="serviceAccount:migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/iam.serviceAccountTokenCreator"
```

### Step 10 - Configure Impersonation - Orchestrator to Audit

```
gcloud iam service-accounts add-iam-policy-binding migration-audit@YOUR_PROJECT_ID.iam.gserviceaccount.com --member="serviceAccount:migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/iam.serviceAccountTokenCreator"
```

### Step 11 - Apply Migration Window Condition to Writer

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/bigquery.dataEditor" --condition=title="Migration Window",description="Writes allowed only during migration hours",expression="request.time.getHours('Asia/Kolkata') >= 2 && request.time.getHours('Asia/Kolkata') <= 6 && request.time.getDayOfWeek('Asia/Kolkata') >= 1 && request.time.getDayOfWeek('Asia/Kolkata') <= 5"
```

### Step 12 - Enforce Uniform Bucket Level Access

```
gcloud resource-manager org-policies enable-enforce constraints/storage.uniformBucketLevelAccess --project=YOUR_PROJECT_ID
```

### Step 13 - Assign Security Reviewer Role to Security Team

```
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="user:SECURITY_EMAIL@yourdomain.com" --role="projects/YOUR_PROJECT_ID/roles/MigrationSecurityReviewer"
```

---

### Validation Commands

Run each command below and verify the expected output.

### Validate - List All Migration Service Accounts

Expected output: 5 migration service accounts

```
gcloud iam service-accounts list --project=YOUR_PROJECT_ID --filter="email:migration-" --format="table(displayName,email)"
```

### Validate - List All Custom Roles

Expected output: migrationAnalyst and MigrationSecurityReviewer

```
gcloud iam roles list --project=YOUR_PROJECT_ID --format="table(name.basename(),title,stage)"
```

### Validate - Check Reader Has Only Read Permissions

Expected output: storage.objectViewer and bigquery.dataViewer only

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --flatten="bindings[].members" --format="table(bindings.role,bindings.members)" --filter="bindings.members:migration-reader"
```

### Validate - Check Writer Has Conditional Bindings

Expected output: bindings show condition title Migration Window

```
gcloud projects get-iam-policy YOUR_PROJECT_ID --flatten="bindings[].members" --format="table(bindings.role,bindings.condition.title)" --filter="bindings.members:migration-writer"
```

### Validate - Check Impersonation Is Set on Reader

Expected output: orchestrator SA listed as Token Creator

```
gcloud iam service-accounts get-iam-policy migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com --format="table(bindings.role,bindings.members)"
```

### Validate - Check Active Org Policies

Expected output: uniformBucketLevelAccess listed

```
gcloud resource-manager org-policies list --project=YOUR_PROJECT_ID --format="table(constraint,updateTime)"
```

### Validate - Confirm Reader Can Access Storage

Expected output: storage.objects.get listed as allowed

```
gcloud projects test-iam-permissions YOUR_PROJECT_ID --permissions="storage.objects.get,storage.objects.list" --impersonate-service-account=migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### Validate - Confirm Reader Cannot Write to Storage

Expected output: storage.objects.create NOT listed as allowed

```
gcloud projects test-iam-permissions YOUR_PROJECT_ID --permissions="storage.objects.create,storage.objects.delete" --impersonate-service-account=migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

---

### Final Cleanup Commands

Run these only after the lab is complete if you want to remove all created resources.

### Delete All Migration Service Accounts

```
gcloud iam service-accounts delete migration-reader@YOUR_PROJECT_ID.iam.gserviceaccount.com --quiet
```

```
gcloud iam service-accounts delete migration-writer@YOUR_PROJECT_ID.iam.gserviceaccount.com --quiet
```

```
gcloud iam service-accounts delete migration-orchestrator@YOUR_PROJECT_ID.iam.gserviceaccount.com --quiet
```

```
gcloud iam service-accounts delete migration-audit@YOUR_PROJECT_ID.iam.gserviceaccount.com --quiet
```

```
gcloud iam service-accounts delete migration-validator@YOUR_PROJECT_ID.iam.gserviceaccount.com --quiet
```

### Delete Custom Roles

```
gcloud iam roles delete migrationAnalyst --project=YOUR_PROJECT_ID
```

```
gcloud iam roles delete MigrationSecurityReviewer --project=YOUR_PROJECT_ID
```

### Delete the Test Storage Bucket

```
gcloud storage buckets delete gs://migration-lab-bucket-YOUR_PROJECT_ID --quiet
```

### Delete the Secret

```
gcloud secrets delete migration-orchestrator-sa-key --project=YOUR_PROJECT_ID --quiet
```

---

## Command Reference Summary

| Task | Command |
|------|---------|
| View IAM policy | `gcloud projects get-iam-policy PROJECT` |
| Grant a role | `gcloud projects add-iam-policy-binding PROJECT --member=... --role=...` |
| Remove a role | `gcloud projects remove-iam-policy-binding PROJECT --member=... --role=...` |
| Test permissions | `gcloud projects test-iam-permissions PROJECT --permissions=...` |
| List roles | `gcloud iam roles list` |
| Create custom role | `gcloud iam roles create NAME --project=PROJECT --permissions=...` |
| Update custom role | `gcloud iam roles update NAME --project=PROJECT --add-permissions=...` |
| Create service account | `gcloud iam service-accounts create NAME --project=PROJECT` |
| List service accounts | `gcloud iam service-accounts list --project=PROJECT` |
| Create SA key | `gcloud iam service-accounts keys create FILE --iam-account=SA_EMAIL` |
| Delete SA key | `gcloud iam service-accounts keys delete KEY_ID --iam-account=SA_EMAIL` |
| Grant impersonation | `gcloud iam service-accounts add-iam-policy-binding SA --member=... --role=roles/iam.serviceAccountTokenCreator` |
| Impersonate SA | `gcloud ... --impersonate-service-account=SA_EMAIL` |
| List org policies | `gcloud resource-manager org-policies list --project=PROJECT` |
| Enforce org policy | `gcloud resource-manager org-policies enable-enforce CONSTRAINT --project=PROJECT` |
| List recommendations | `gcloud recommender recommendations list --project=PROJECT --location=global --recommender=google.iam.policy.Recommender` |

---

*GCP IAM Fundamentals - gcloud Commands Lab*
*Migration Project Track*
