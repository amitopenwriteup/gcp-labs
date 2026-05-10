# Cloud Run Workshop — GCP Console UI Guide

**Duration:** ~3 hours  
**Level:** Intermediate  
**Prerequisites:** A Google Cloud project with billing enabled, a GitHub account, basic understanding of containers and REST APIs

> All deployment steps in this workshop are completed entirely through the **Google Cloud Console** at [https://console.cloud.google.com](https://console.cloud.google.com). No terminal or CLI required.

---

## Table of Contents

1. [Module 1 — Cloud Run Fundamentals](#module-1--cloud-run-fundamentals)
   - [Serverless Container Platform Overview](#11-serverless-container-platform-overview)
   - [Service Configuration and Deployment](#12-service-configuration-and-deployment)
   - [Automatic Scaling and Concurrency](#13-automatic-scaling-and-concurrency)
2. [Module 2 — Integration Patterns](#module-2--integration-patterns)
   - [Event-Driven Architectures with Pub/Sub](#21-event-driven-architectures-with-pubsub)
   - [Database Connections and Connection Pooling](#22-database-connections-and-connection-pooling)
   - [Secrets Management and Environment Variables](#23-secrets-management-and-environment-variables)
3. [Module 3 — Hands-on Lab: Deploying Microservices to Cloud Run](#module-3--hands-on-lab-deploying-microservices-to-cloud-run)
4. [Cleanup](#cleanup)
5. [Quick Reference](#quick-reference--where-to-find-everything-in-the-gcp-console)

---

## Module 1 — Cloud Run Fundamentals

### 1.1 Serverless Container Platform Overview

Cloud Run is a fully managed serverless platform on Google Cloud that runs stateless containers. You bring a container image or source code; Google handles everything else — routing, scaling, patching, TLS, and availability.

**Key concepts to understand before starting:**

| Concept | What it means |
|---|---|
| Service | A long-running HTTP endpoint backed by your container |
| Revision | An immutable snapshot created on every deploy |
| Instance | A running copy of your container handling requests |
| Concurrency | How many requests one instance handles simultaneously |
| Scale to zero | Instances shut down when there is no traffic; you pay nothing |

**Cloud Run vs other Google Cloud compute options:**

| Feature | Cloud Run | Cloud Functions | GKE |
|---|---|---|---|
| Unit of deployment | Container | Function | Pod / Deployment |
| Max request timeout | 60 minutes | 9 minutes | Unlimited |
| Scales to zero | Yes | Yes | No (by default) |
| Custom runtime | Any language | Limited runtimes | Any |
| Infrastructure control | Low | None | High |

---

### 1.2 Service Configuration and Deployment

#### Enable required APIs

1. In the GCP Console, click the **Navigation menu** (three horizontal lines, top-left)
2. Go to **APIs & Services > Library**
3. Search for and enable each of the following:
   - `Cloud Run Admin API`
   - `Cloud SQL Admin API`
   - `Secret Manager API`
   - `Cloud Pub/Sub API`

#### Navigate to Cloud Run

1. Click the **Navigation menu**
2. Scroll to the **Serverless** section
3. Click **Cloud Run**
4. You will land on the **Services** list page

#### Deploy your first service using a sample container

1. Click **Create Service** (blue button, top of the page)
2. Under the container source, select **Test with a sample container**
   - This uses a ready-made Hello World image provided by Google
   - In the lab you will deploy from your own GitHub repository instead
3. Under **Service name**, enter `hello-world-service`
4. Under **Region**, choose `us-central1 (Iowa)`
5. Under **Authentication**, select **Allow unauthenticated invocations**
6. Click **Container, Volumes, Networking, Security** to expand advanced settings

**Container tab — key fields:**

| Field | Recommended value | Notes |
|---|---|---|
| Container port | 8080 | Must match the port your app listens on |
| Memory | 512 MiB | Increase for memory-intensive workloads |
| CPU | 1 | Increase for CPU-bound tasks |
| Request timeout | 300 seconds | Maximum time allowed per request |
| Maximum concurrent requests | 80 | Simultaneous requests handled per instance |

7. Click **Create**
8. Wait for the green checkmark (30–60 seconds)
9. Click the **URL** at the top of the service detail page to confirm it is running

#### Understanding revisions

Every time you change any setting and save, Cloud Run creates a new **Revision**. Previous revisions are preserved and can receive traffic at any time.

To view all revisions:

1. Open your service from the Cloud Run list
2. Click the **Revisions** tab
3. Each row shows the revision name, creation time, traffic split, and active instance count

#### Canary deployment — splitting traffic between revisions

1. Open your service
2. Click **Edit & Deploy New Revision**
3. Change any setting (for example, increase Memory to 1 GiB)
4. At the bottom, under **This revision**, select **Do not serve this revision immediately**
5. Click **Deploy**
6. Go to the **Revisions** tab
7. Click **Manage Traffic**
8. Set the new revision to `10%` and the previous revision to `90%`
9. Click **Save**
10. To promote to 100%, repeat and set the new revision to `100%`



---

### 1.3 Automatic Scaling and Concurrency

#### Configuring instance limits and concurrency

1. Open your service, click **Edit & Deploy New Revision**
2. Expand **Container, Volumes, Networking, Security**
3. Under the **Container** tab, scroll to **Capacity**

| Setting | Recommended value | Effect |
|---|---|---|
| Minimum instances | 0 | Scales to zero; set to 1+ to eliminate cold starts |
| Maximum instances | 10 | Hard cap on scale-out; raise for high-traffic services |
| Maximum concurrent requests | 80 | Requests per instance; lower for CPU-heavy workloads |
| Request timeout | 300 seconds | Requests exceeding this are cancelled |

4. Click **Deploy**

#### CPU allocation mode

In the **Container** tab, find **CPU allocation** and choose:

- **CPU is only allocated during request processing** — cheapest option; best for standard HTTP APIs
- **CPU is always allocated** — required for WebSockets, background processing, or streaming responses

#### Startup CPU boost

1. In the **Container** tab, check **Enable CPU boost during startup**
2. This temporarily gives the instance extra CPU during startup to reduce cold start latency
3. Click **Deploy**

#### Observing autoscaling in the Console

1. Open your service
2. Click the **Metrics** tab
3. Observe the **Instance count** chart
4. Change the **Timeframe** to the last 1 hour
5. Send traffic to your service URL; the chart updates to show instances scaling up, then back down after traffic stops

**Concurrency recommendations by workload:**

| Workload | Recommended concurrency | Reason |
|---|---|---|
| ML inference / image processing | 1 | CPU-bound; one request saturates the CPU |
| REST API with database calls | 10–80 | I/O-bound; instance waits while DB responds |
| Lightweight proxy or fan-out | 200–1000 | Minimal CPU per request |

---



