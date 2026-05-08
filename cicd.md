# GCP CI/CD Pipeline with Cloud Build — Step-by-Step Guide

## Overview

This guide walks through setting up a complete **CI/CD pipeline on GCP using Cloud Build**, connected to a GitHub repository. The pipeline installs dependencies, runs tests, builds a Docker image, and pushes it to **Artifact Registry**.

---

## Prerequisites

- Active GCP project with billing enabled
- GitHub account
- Docker image repository already created in **Artifact Registry**
- GCS bucket for storing build logs (e.g., `gs://amit23compow`)

---

## Step 1: Create a Service Account

1. In GCP Console, click **"View All Products"**
2. Navigate to **IAM & Admin → Service Accounts**
3. Click **"Create Service Account"**
4. Provide a name (e.g., `cloudbuild-sa`) and click **"Create and Continue"**
5. Assign the following roles:

| Role | Purpose |
|---|---|
| `Cloud Build Service Account` | Run build jobs |
| `Artifact Registry Writer` | Push Docker images |
| `Storage Object Admin` | Write logs to GCS bucket |
| `Logs Writer` | Write build logs |

6. Click **"Done"**

---

## Step 2: Fork the GitHub Repository

1. Open the repository: [https://github.com/amitopenwriteup/cicdproject](https://github.com/amitopenwriteup/cicdproject)
2. Click **"Fork"** (top-right corner)
3. Select your GitHub account as the destination
4. Click **"Create Fork"**

> Your forked repo will be at: `https://github.com/YOUR_USERNAME/cicdproject`

---

## Step 3: Navigate to Cloud Build

1. In GCP Console, click **"View All Products"**
2. Under the **CI/CD** section, click **"Cloud Build"**
3. Select **"1st Gen"** triggers (legacy trigger interface)

---

## Step 4: Connect Your GitHub Repository

1. Select your **region** from the dropdown
2. Click **"Connect Repository"**
3. Select **GitHub** as the source provider
4. Click **"Continue"**
5. A GitHub OAuth popup will appear — **authenticate** with your GitHub credentials
6. Select your forked repository `cicdproject` from the list
7. Check the authorization checkbox
8. Click **"Connect"**

---

## Step 5: Create a Build Trigger

1. Click **"Triggers"** in the left sidebar
2. Click **"Create Trigger"**
3. Fill in the trigger details:

| Field | Value |
|---|---|
| **Name** | e.g., `cicd-trigger-main` |
| **Event** | Push to a branch |
| **Repository** | Your forked `cicdproject` repo |
| **Branch** | `^main$` (regex for main branch) |
| **Build configuration** | Cloud Build configuration file (`cloudbuild.yaml`) |
| **Service Account** | Select the service account created in Step 1 |

4. Click **"Save"**

---

## Step 6: Run the Trigger

1. On the **Triggers** page, locate your newly created trigger
2. Click **"Run"** next to the trigger name
3. Confirm by clicking **"Run Trigger"** in the popup
4. Cloud Build will start a new build — click on the build ID to monitor progress

---

## Step 7: `cloudbuild.yaml` — Pipeline Configuration

Place this file in the **root of your repository** as `cloudbuild.yaml`.

```yaml
steps:
  # Step 1: Install dependencies in a virtual environment
  - name: 'python:3.9'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        python -m venv venv && \
        . venv/bin/activate && \
        pip install --upgrade pip && \
        pip install -r requirements.txt

  # Step 2: Run unit tests in the same virtual environment
  - name: 'python:3.9'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        . venv/bin/activate && \
        python -m unittest discover tests

  # Step 3: Build Docker image and tag with Build ID
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'us-east1-docker.pkg.dev/upgradlabs-1738334790345/mydockerrgisty/myapp:$BUILD_ID'
      - '.'

  # Step 4: Push Docker image to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'us-east1-docker.pkg.dev/upgradlabs-1738334790345/mydockerrgisty/myapp:$BUILD_ID'

# Store build logs in GCS bucket
logsBucket: "gs://amit23compow"
```

---

## Pipeline Stages Explained

### Stage 1 — Install Dependencies
- Uses the official `python:3.9` Docker image
- Creates an isolated **virtual environment** (`venv`) to avoid package conflicts
- Upgrades `pip` to the latest version
- Installs all packages listed in `requirements.txt`

> ⚠️ Each Cloud Build step runs in a **fresh container**. The `venv` directory is persisted on the build workspace volume between steps.

### Stage 2 — Run Unit Tests
- Re-activates the same `venv` created in Stage 1
- Runs `unittest` auto-discovery on the `tests/` directory
- If any test fails, the build **stops here** and does not proceed to Docker build

### Stage 3 — Build Docker Image
- Uses Google's managed `cloud-builders/docker` image
- Builds the image from the `Dockerfile` in the repo root
- Tags the image using `$BUILD_ID` — a unique ID automatically provided by Cloud Build
- Full image path: `us-east1-docker.pkg.dev/upgradlabs-1738334790345/mydockerrgisty/myapp:$BUILD_ID`

### Stage 4 — Push to Artifact Registry
- Pushes the built and tagged image to **Artifact Registry**
- Ensures the image is stored and versioned by build ID
- Can be pulled by Cloud Run, GKE, or any other GCP service

---

## Required Repository Structure

```
cicdproject/
├── cloudbuild.yaml       ← Pipeline definition
├── Dockerfile            ← Docker build instructions
├── requirements.txt      ← Python dependencies
├── app.py                ← Application entry point
└── tests/
    └── test_app.py       ← Unit tests (discovered automatically)
```

---

## IAM Permissions Checklist

Ensure your service account has all of the following before running the trigger:

- [ ] `roles/cloudbuild.builds.builder`
- [ ] `roles/artifactregistry.writer`
- [ ] `roles/storage.objectAdmin` (for log bucket access)
- [ ] `roles/logging.logWriter`

---

## Troubleshooting

| Error | Likely Cause | Fix |
|---|---|---|
| `Permission denied on Artifact Registry` | SA missing write role | Add `artifactregistry.writer` to service account |
| `venv: command not found` | Python base image issue | Ensure step uses `python:3.9` not `python:3.9-slim` |
| `No module named X` | Stage 2 can't find installed packages | Confirm `venv` is activated with `. venv/bin/activate` |
| `Tests not found` | Wrong test directory | Ensure test files are inside a `tests/` folder |
| `Cannot access log bucket` | GCS bucket permission | Grant `storage.objectAdmin` on the specific bucket |
| `Trigger not firing on push` | Branch regex mismatch | Use `^main$` — not just `main` |
| `Docker push 401 Unauthorized` | Artifact Registry API not enabled | Enable `artifactregistry.googleapis.com` |

---

## Quick Reference: Useful `gcloud` Commands

```bash
# List all triggers
gcloud builds triggers list --region=us-east1

# Manually run a trigger by name
gcloud builds triggers run cicd-trigger-main --region=us-east1 --branch=main

# View recent builds
gcloud builds list --limit=5

# Stream live build logs
gcloud builds log BUILD_ID --stream
```

---

*Guide covers GCP Cloud Build 1st Gen — Python 3.9 + Docker + Artifact Registry pipeline*
