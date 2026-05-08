# Deploy a Microservice to Cloud Run — Simple UI Guide

## What You Will Build

A single Python microservice deployed on **Cloud Run** using the GCP Console UI — no CLI required.

```
Your Code (GitHub)  →  Cloud Build  →  Artifact Registry  →  Cloud Run
```

---

## Prerequisites

- GCP account with a project created
- GitHub account
- Basic Python knowledge

---

## Step 1: Enable Required APIs

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. In the search bar, type **"APIs & Services"** and click it
3. Click **"+ Enable APIs and Services"**
4. Search and enable each of these one by one:

| API | Search For |
|---|---|
| Cloud Run API | `Cloud Run` |
| Cloud Build API | `Cloud Build` |
| Artifact Registry API | `Artifact Registry` |

---

## Step 2: Create the Service Account

1. In the search bar, type **"Service Accounts"** and open it
2. Click **"+ Create Service Account"**
3. Fill in:
   - **Name:** `my-run-sa`
   - **Description:** Service account for Cloud Run
4. Click **"Create and Continue"**
5. Add these roles one by one using the dropdown:
   - `Cloud Run Admin`
   - `Storage Object Viewer`
   - `Artifact Registry Writer`
6. Click **"Done"**

---

## Step 3: Create the App Code

Create a new GitHub repository and add these 3 files:

### `app.py`

```python
import os
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/")
def home():
    return jsonify({"message": "Hello from Cloud Run!", "status": "ok"})

@app.route("/health")
def health():
    return jsonify({"status": "healthy"})

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    app.run(host="0.0.0.0", port=port)
```

### `requirements.txt`

```
flask==3.0.0
gunicorn==21.2.0
```

### `Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```

---

## Step 4: Create Artifact Registry Repository

1. Search for **"Artifact Registry"** in the GCP Console
2. Click **"+ Create Repository"**
3. Fill in:
   - **Name:** `my-app-repo`
   - **Format:** Docker
   - **Region:** `us-central1`
4. Click **"Create"**

---

## Step 5: Set Up Cloud Build Trigger

1. Search for **"Cloud Build"** → Click **"Triggers"**
2. Click **"+ Create Trigger"**
3. Fill in the details:

| Field | Value |
|---|---|
| Name | `deploy-my-app` |
| Event | Push to a branch |
| Source | Connect your GitHub repo |
| Branch | `^main$` |
| Build config | Inline |

4. In the **Inline Build Config** box, paste:

```yaml
steps:
  # Build the Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-app-repo/my-app:$BUILD_ID'
      - '.'

  # Push image to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-app-repo/my-app:$BUILD_ID'

  # Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'gcloud'
    args:
      - 'run'
      - 'deploy'
      - 'my-app'
      - '--image=us-central1-docker.pkg.dev/$PROJECT_ID/my-app-repo/my-app:$BUILD_ID'
      - '--region=us-central1'
      - '--platform=managed'
      - '--allow-unauthenticated'
```

5. Under **Service Account**, select `my-run-sa`
6. Click **"Save"**

---

## Step 6: Run the Build

1. On the Triggers page, find `deploy-my-app`
2. Click **"Run"** → **"Run Trigger"**
3. Click the build link to watch live progress

You will see 3 green checkmarks when done:

```
✅ Step 1 — Docker Build
✅ Step 2 — Push to Artifact Registry
✅ Step 3 — Deploy to Cloud Run
```

---

## Step 7: View Your Live Service

1. Search for **"Cloud Run"** in the console
2. Click on **`my-app`**
3. Copy the **URL** shown at the top (e.g. `https://my-app-xxxx-uc.a.run.app`)
4. Open it in your browser — you should see:

```json
{ "message": "Hello from Cloud Run!", "status": "ok" }
```

---

## Step 8: Make a Change and Redeploy

1. Edit `app.py` — change the message to anything
2. Commit and push to `main` branch on GitHub
3. Cloud Build triggers **automatically**
4. New version is live within ~2 minutes

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Build fails at Docker step | Check Artifact Registry repo name matches exactly |
| `403` when deploying | Add `Cloud Run Admin` role to your service account |
| URL returns error | Click the service in Cloud Run → check **Logs** tab |
| Trigger not firing | Confirm branch is `main` and GitHub is connected |

---

*Simple Cloud Run Deploy — GCP Console UI | Python + Docker*
