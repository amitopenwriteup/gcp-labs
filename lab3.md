# GCP IAM Lab — Roles, Custom Roles, and Service Accounts

## Use Case

A data pipeline reads raw files from Cloud Storage and loads them into BigQuery. You need a dedicated service account for this pipeline with exactly the permissions it needs — no more, no less. You will inspect an existing predefined role, build a custom role, create a service account, and assign the role to it.

Each task is shown twice: once using the Google Cloud Console (web UI) and once using the gcloud CLI.

---

## Prerequisites

- A GCP project with billing enabled
- Owner or Editor access to the project
- gcloud CLI installed and authenticated (`gcloud auth login`)
- Your project ID set: `gcloud config set project YOUR_PROJECT_ID`

Replace `YOUR_PROJECT_ID` throughout this lab with your actual project ID.

---

## Part 1 — Inspect a Predefined Role

Before creating anything, understand what permissions a predefined role contains and decide whether it fits your use case.

The pipeline needs to read from Cloud Storage and load data into BigQuery. The closest predefined role for BigQuery loading is `roles/bigquery.dataEditor`. Inspect it first.

---

### Console UI

1. Open [https://console.cloud.google.com](https://console.cloud.google.com) and select your project.
2. Click the Navigation menu (top-left) and go to **IAM & Admin > Roles**.
3. In the filter bar at the top of the roles list, type `BigQuery Data Editor`.
4. Click the **BigQuery Data Editor** row to open its detail page.
5. Scroll through the **Included permissions** list. Note permissions such as:
   - `bigquery.datasets.get`
   - `bigquery.tables.create`
   - `bigquery.tables.updateData`
   - `bigquery.tables.delete` — the pipeline does not need this
6. Click the back arrow to return to the roles list.
7. Repeat for **Storage Object Viewer** — filter and click it to review its permissions:
   - `storage.objects.get`
   - `storage.objects.list`

**Checkpoint:** You have identified that `roles/bigquery.dataEditor` includes `bigquery.tables.delete`, which the pipeline should not have. You will build a custom role in Part 2 without it. `roles/storage.objectViewer` fits the storage read requirement as-is.

---

### gcloud CLI

List all predefined roles:

```bash
gcloud iam roles list --filter="name:roles/bigquery"
```

Describe the BigQuery Data Editor role to see its permissions:

```bash
gcloud iam roles describe roles/bigquery.dataEditor
```

Look for `includedPermissions` in the output. Confirm `bigquery.tables.delete` is present — this confirms the predefined role is broader than needed.

Describe the Storage Object Viewer role:

```bash
gcloud iam roles describe roles/storage.objectViewer
```

Confirm the output contains only read permissions: `storage.objects.get` and `storage.objects.list`.

---

## Part 2 — Create a Custom Role

The pipeline needs BigQuery write access but must not be able to delete tables. Build a custom role with only the required permissions.

**Permissions to include:**

| Permission | Why |
|---|---|
| `bigquery.datasets.get` | Read dataset metadata |
| `bigquery.tables.get` | Read table schema |
| `bigquery.tables.create` | Create new tables |
| `bigquery.tables.list` | List tables in a dataset |
| `bigquery.tables.updateData` | Insert and update rows |

---

### Console UI

1. Go to **IAM & Admin > Roles**.
2. Click **+ CREATE ROLE** at the top of the page.
3. Fill in the role details:

   | Field | Value |
   |---|---|
   | Title | `Pipeline Data Loader` |
   | Description | `Allows loading data into BigQuery without delete access` |
   | ID | `pipelineDataLoader` |
   | Role launch stage | `General Availability` |

4. Click **+ ADD PERMISSIONS**.
5. In the filter box, type `bigquery.datasets.get` and check the box next to it.
6. Clear the filter and type `bigquery.tables.get` — check it.
7. Repeat for each remaining permission:
   - `bigquery.tables.create`
   - `bigquery.tables.list`
   - `bigquery.tables.updateData`
8. Click **ADD**.
9. Click **CREATE**.

**Checkpoint:** The role `pipelineDataLoader` appears in the Roles list with a **CUSTOM** badge. Open it and confirm exactly 5 permissions are listed and `bigquery.tables.delete` is not among them.

---
## Lab2 :Command line
### gcloud CLI

Create a YAML file defining the custom role:

```bash
cat > pipeline-data-loader-role.yaml << 'EOF'
title: "Pipeline Data Loader"
description: "Allows loading data into BigQuery without delete access"
stage: "GA"
includedPermissions:
  - bigquery.datasets.get
  - bigquery.tables.get
  - bigquery.tables.create
  - bigquery.tables.list
  - bigquery.tables.updateData
EOF
```

Create the role in your project:

```bash
gcloud iam roles create pipelineDataLoader \
  --project=YOUR_PROJECT_ID \
  --file=pipeline-data-loader-role.yaml
```

Verify the role was created and review its permissions:

```bash
gcloud iam roles describe pipelineDataLoader \
  --project=YOUR_PROJECT_ID
```

**Checkpoint:** The output shows `stage: GA` and lists all 5 permissions. The `name` field shows `projects/YOUR_PROJECT_ID/roles/pipelineDataLoader`.

---



### gcloud CLI

Create the service account:

```bash
gcloud iam service-accounts create data-pipeline-loader \
  --display-name="Data Pipeline Loader" \
  --description="Service account for the BigQuery data loading pipeline" \
  --project=YOUR_PROJECT_ID
```

Verify it was created:

```bash
gcloud iam service-accounts list \
  --project=YOUR_PROJECT_ID \
  --filter="email:data-pipeline-loader"
```

**Checkpoint:** The output shows the service account with email `data-pipeline-loader@YOUR_PROJECT_ID.iam.gserviceaccount.com` and `DISABLED: False`.

---

## Part 4 — Assign Roles to the Service Account

Assign two roles to the service account:

- Your custom role `pipelineDataLoader` — for BigQuery write access
- The predefined `roles/storage.objectViewer` — for Cloud Storage read access

---


---

### gcloud CLI

Assign the custom role:

```bash
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:data-pipeline-loader@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="projects/YOUR_PROJECT_ID/roles/pipelineDataLoader"
```

Assign the predefined Storage Object Viewer role:

```bash
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:data-pipeline-loader@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
```

Verify both bindings exist by checking the project's IAM policy and filtering for the service account:

```bash
gcloud projects get-iam-policy YOUR_PROJECT_ID \
  --flatten="bindings[].members" \
  --format="table(bindings.role, bindings.members)" \
  --filter="bindings.members:data-pipeline-loader"
```

**Checkpoint:** The output shows two rows — one for `projects/YOUR_PROJECT_ID/roles/pipelineDataLoader` and one for `roles/storage.objectViewer`, both bound to the service account.

Verify that `bigquery.tables.delete` is not reachable by checking whether any role bound to the SA includes it:

```bash
gcloud iam roles describe pipelineDataLoader \
  --project=YOUR_PROJECT_ID \
  --format="value(includedPermissions)" | grep "tables.delete"
```

**Checkpoint:** No output is returned, confirming delete permission is absent from the custom role.

---

## Summary

| Step | What you did | Console path | CLI command |
|---|---|---|---|
| Inspect predefined role | Reviewed permissions of `bigquery.dataEditor` and `storage.objectViewer` | IAM & Admin > Roles | `gcloud iam roles describe` |
| Create custom role | Built `pipelineDataLoader` with 5 targeted permissions | IAM & Admin > Roles > + CREATE ROLE | `gcloud iam roles create --file` |
| Create service account | Created `data-pipeline-loader` SA for the pipeline identity | IAM & Admin > Service Accounts | `gcloud iam service-accounts create` |
| Assign roles | Bound both roles to the SA at project level | IAM & Admin > IAM > + GRANT ACCESS | `gcloud projects add-iam-policy-binding` |

**Key principles applied:**

- Inspecting predefined roles before use reveals whether they are over-permissioned for your workload.
- Custom roles let you grant exactly the permissions a workload needs and nothing else.
- One service account per workload keeps permissions scoped and auditable.
- Verifying access with Policy Troubleshooter (UI) or `get-iam-policy` (CLI) confirms the binding is correct before deploying.

---

## Cleanup

Remove the role bindings:

```bash
gcloud projects remove-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:data-pipeline-loader@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="projects/YOUR_PROJECT_ID/roles/pipelineDataLoader"

gcloud projects remove-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:data-pipeline-loader@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
```

Delete the service account:

```bash
gcloud iam service-accounts delete \
  data-pipeline-loader@YOUR_PROJECT_ID.iam.gserviceaccount.com \
  --project=YOUR_PROJECT_ID
```

Delete the custom role:

```bash
gcloud iam roles delete pipelineDataLoader \
  --project=YOUR_PROJECT_ID
```
