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

#### Ingress controls

1. Open your service, click **Edit & Deploy New Revision**
2. Click the **Networking** tab
3. Under **Ingress**, choose:
   - **All** — accessible from the public internet
   - **Internal** — accessible only from within your VPC or other Cloud Run services
   - **Internal and Cloud Load Balancing** — internal plus via a load balancer
4. Click **Deploy**

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

## Module 2 — Integration Patterns

### 2.1 Event-Driven Architectures with Pub/Sub

#### Create a Pub/Sub topic

1. Go to **Navigation menu > Pub/Sub > Topics**
2. Click **Create Topic**
3. Topic ID: `order-events`
4. Leave **Add a default subscription** unchecked
5. Click **Create**

#### Create a service account for Pub/Sub to invoke Cloud Run

1. Go to **IAM & Admin > Service Accounts**
2. Click **Create Service Account**
3. Name: `pubsub-invoker`, Description: `Allows Pub/Sub to invoke Cloud Run services`
4. Click **Create and Continue**
5. Skip the project-level role (the role is granted per service instead)
6. Click **Done**

#### Grant the service account Cloud Run Invoker permission

1. Go to **Cloud Run**, open the target service (e.g. `notification-service`)
2. Click the **Security** tab in the right-hand panel
3. Click **Add Principal**
4. New principal: `pubsub-invoker@YOUR_PROJECT_ID.iam.gserviceaccount.com`
5. Role: **Cloud Run > Cloud Run Invoker**
6. Click **Save**

#### Create a Pub/Sub push subscription

1. Go to **Pub/Sub > Subscriptions**
2. Click **Create Subscription**

| Field | Value |
|---|---|
| Subscription ID | `order-events-notif-sub` |
| Cloud Pub/Sub topic | `order-events` |
| Delivery type | Push |
| Endpoint URL | `https://YOUR-NOTIFICATION-SERVICE-URL/pubsub` |
| Enable authentication | Toggle ON |
| Service account | `pubsub-invoker@YOUR_PROJECT_ID.iam.gserviceaccount.com` |
| Acknowledgement deadline | 60 seconds |
| Retry policy | Retry after exponential backoff |
| Minimum backoff | 10 seconds |
| Maximum backoff | 300 seconds |

3. Click **Create**

#### Publish a test message manually

1. Go to **Pub/Sub > Topics**, click `order-events`
2. Click the **Messages** tab
3. Click **Publish Message**
4. Paste in the **Message body**:
   ```json
   {"order_id": 1, "customer_email": "test@example.com", "amount": 49.99}
   ```
5. Click **Publish**
6. Go to **Cloud Run > notification-service > Logs** to confirm the message was received

#### Add a dead-letter topic for failed messages

1. Go to **Pub/Sub > Subscriptions**, open `order-events-notif-sub`
2. Click **Edit**
3. Scroll to **Dead lettering**, toggle **Enable dead lettering** ON
4. Click **Create a topic**, name it `order-events-dlq`
5. Set **Maximum delivery attempts** to `5`
6. Click **Update**

#### Create an Eventarc trigger (Cloud Storage example)

1. Go to **Cloud Run**, open your service, click the **Triggers** tab
2. Click **Add Trigger > Eventarc Trigger**

| Field | Value |
|---|---|
| Trigger name | `gcs-upload-trigger` |
| Event provider | Cloud Storage |
| Event type | `google.cloud.storage.object.v1.finalized` |
| Bucket | Select or create your bucket |
| Service account | Your Cloud Run service account |

3. Click **Save**

---

### 2.2 Database Connections and Connection Pooling

#### Create a Cloud SQL (PostgreSQL) instance

1. Go to **Navigation menu > SQL**
2. Click **Create Instance**, then choose **PostgreSQL**

| Field | Value |
|---|---|
| Instance ID | `workshop-postgres` |
| Password | Set a strong password and save it |
| Database version | PostgreSQL 15 |
| Region | `us-central1` |
| Zonal availability | Single zone |
| Machine type | Shared core — 1 vCPU, 0.614 GB |
| Storage | 10 GB SSD |
| Public IP | Uncheck |
| Private IP | Check |

3. Click **Create Instance** (takes 3–5 minutes)

#### Create a database and user

1. Click on `workshop-postgres` once it is ready
2. In the left sidebar, click **Databases**
3. Click **Create Database**, name it `orders`, click **Create**
4. In the left sidebar, click **Users**
5. Click **Add User Account**, select **Built-in authentication**
6. Username: `workshop_user`, Password: `your-db-password`
7. Click **Add**

#### Connect Cloud Run to Cloud SQL

1. Go to **Cloud Run**, open your service
2. Click **Edit & Deploy New Revision**
3. Click the **Container** tab, scroll to **Cloud SQL connections**
4. Click **Add Connection**, select `workshop-postgres`
5. Click **Done**, then **Deploy**

> Cloud Run injects a Unix socket at `/cloudsql/PROJECT:REGION:INSTANCE/.s.PGSQL.5432`. Your application uses this path to connect — no IP address, firewall rule, or SSL management needed.

#### Grant Cloud SQL access to your service account

1. Go to **IAM & Admin > IAM**
2. Find the service account used by your Cloud Run service
3. Click the pencil (Edit) icon
4. Click **Add Another Role**, search for and select `Cloud SQL Client`
5. Click **Save**

#### Connection pool sizing reference

> These values go in your application code. Use the formula below when configuring your database connection.

```
pool_size per instance = max_db_connections / max_cloud_run_instances

Example:
  Cloud SQL max connections : 100
  Cloud Run max instances   : 10
  pool_size                 : 10 per instance
  max_overflow              : 2
  pool_recycle              : 1800 seconds
  pool_pre_ping             : True
```

---

### 2.3 Secrets Management and Environment Variables

#### Create a secret in Secret Manager

1. Go to **Navigation menu > Security > Secret Manager**
2. Click **Create Secret**

| Field | Value |
|---|---|
| Name | `db-password` |
| Secret value | Your Cloud SQL password |
| Replication policy | Automatic |

3. Click **Create Secret**

#### Grant your service account access to the secret

1. Click on `db-password`, then the **Permissions** tab
2. Click **Grant Access**
3. New principal: `cloudrun-workshop-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com`
4. Role: **Secret Manager Secret Accessor**
5. Click **Save**

#### Mount the secret in Cloud Run

1. Go to **Cloud Run**, open your service
2. Click **Edit & Deploy New Revision**
3. In the **Container** tab, scroll to **Secrets**
4. Click **Reference a Secret**

| Field | Value |
|---|---|
| Secret | `db-password` |
| Reference method | Exposed as environment variable |
| Environment variable name | `DB_PASS` |
| Version | latest |

5. Click **Done**, then **Deploy**

> To use file-based mounting instead: choose **Mounted as volume**, set the path to `/secrets/db-password`. Your app reads it with `open("/secrets/db-password").read()`. Volume-mounted secrets update automatically on rotation; environment variable secrets require a new revision.

#### Add non-sensitive environment variables

1. In **Edit & Deploy New Revision**, scroll to **Variables & Secrets**
2. Under **Environment variables**, click **Add Variable** for each:

| Name | Value |
|---|---|
| APP_ENV | production |
| LOG_LEVEL | info |
| MAX_RETRIES | 3 |

3. Click **Deploy**

#### Rotate a secret

1. Go to **Security > Secret Manager**, click `db-password`
2. Click **New Version**, enter the new value, click **Add New Version**
3. Click the **Versions** tab, find version 1, click its three-dot menu, select **Disable**

> Services with the secret mounted as a volume pick up the new value within ~60 seconds automatically. Services using it as an environment variable need a new revision deployed.

---

## Module 3 — Hands-on Lab: Deploying Microservices to Cloud Run

### Lab Architecture

```
Client
  |
  v
[Order Service]  --writes to-->  Cloud SQL (PostgreSQL)
  |
  +--publishes to-->  Pub/Sub Topic: order-events
                              |
                              v
                    [Notification Service]
```

**Order Service** — REST API that accepts orders, saves them to Cloud SQL, and publishes an event to Pub/Sub.

**Notification Service** — Pub/Sub consumer that receives events and logs a notification (simulating email delivery).

---

### 3.1 Prepare Source Code in GitHub

Both services are deployed using Cloud Run's **Deploy from source** feature, which reads source code from a connected GitHub repository and handles the container build automatically in the background.

#### Create the GitHub repositories

1. Log in to [https://github.com](https://github.com)
2. Create a new public repository named `order-service`
3. Create a second public repository named `notification-service`

#### Order Service files

In the `order-service` repository, create each file using GitHub's web editor (**Add file > Create new file**):

**`app.py`**

```python
import os, json, base64
from flask import Flask, request, jsonify
from google.cloud import pubsub_v1
import sqlalchemy

app = Flask(__name__)
publisher = pubsub_v1.PublisherClient()

def create_pool():
    db_socket = os.environ["DB_HOST"]
    return sqlalchemy.create_engine(
        sqlalchemy.engine.url.URL.create(
            drivername="postgresql+pg8000",
            username=os.environ["DB_USER"],
            password=os.environ["DB_PASS"],
            database=os.environ["DB_NAME"],
            query={"unix_sock": f"{db_socket}/.s.PGSQL.5432"},
        ),
        pool_size=5, max_overflow=2,
        pool_recycle=1800, pool_pre_ping=True
    )

pool = None

@app.before_first_request
def startup():
    global pool
    pool = create_pool()
    with pool.connect() as conn:
        conn.execute(sqlalchemy.text("""
            CREATE TABLE IF NOT EXISTS orders (
                id SERIAL PRIMARY KEY,
                customer_email TEXT NOT NULL,
                amount NUMERIC(10,2) NOT NULL,
                status TEXT DEFAULT 'placed',
                created_at TIMESTAMP DEFAULT NOW()
            )
        """))

@app.route('/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    if not data or 'customer_email' not in data or 'amount' not in data:
        return jsonify({'error': 'customer_email and amount are required'}), 400
    with pool.connect() as conn:
        result = conn.execute(
            sqlalchemy.text(
                "INSERT INTO orders (customer_email, amount) "
                "VALUES (:email, :amount) RETURNING id"
            ),
            {"email": data["customer_email"], "amount": data["amount"]}
        )
        order_id = result.fetchone()[0]
    project = os.environ.get("PROJECT_ID", "")
    topic = f"projects/{project}/topics/order-events"
    event = json.dumps({
        "order_id": order_id,
        "customer_email": data["customer_email"],
        "amount": data["amount"]
    })
    publisher.publish(topic, event.encode(), event_type="order_placed")
    return jsonify({"order_id": order_id, "status": "placed"}), 201

@app.route('/health')
def health():
    return jsonify({"status": "ok"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 8080)))
```

**`requirements.txt`**

```
flask==3.0.0
gunicorn==21.2.0
sqlalchemy==2.0.23
pg8000==1.30.3
google-cloud-pubsub==2.18.4
```

**`Dockerfile`**

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "1", "--threads", "8", "app:app"]
```

#### Notification Service files

In the `notification-service` repository, create:

**`app.py`**

```python
import os, json, base64
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/pubsub', methods=['POST'])
def handle_pubsub():
    envelope = request.get_json()
    if not envelope or 'message' not in envelope:
        return jsonify({'error': 'invalid payload'}), 400
    raw = base64.b64decode(envelope['message']['data']).decode('utf-8')
    order = json.loads(raw)
    print(
        f"[NOTIFICATION] Sending confirmation to "
        f"{order['customer_email']} for order #{order['order_id']} "
        f"(${order['amount']})"
    )
    return jsonify({'status': 'ok'}), 200

@app.route('/health')
def health():
    return jsonify({"status": "ok"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 8080)))
```

**`requirements.txt`**

```
flask==3.0.0
gunicorn==21.2.0
```

**`Dockerfile`**

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "1", "--threads", "4", "app:app"]
```

---

### 3.2 Create the Shared Service Account

1. Go to **IAM & Admin > Service Accounts**
2. Click **Create Service Account**
3. Name: `cloudrun-workshop-sa`, Description: `Workshop service account`
4. Click **Create and Continue**
5. Add the following roles (click **Add Another Role** after each):
   - `Cloud SQL Client`
   - `Secret Manager Secret Accessor`
   - `Pub/Sub Publisher`
   - `Pub/Sub Subscriber`
6. Click **Done**

---

### 3.3 Store the Database Password in Secret Manager

1. Go to **Security > Secret Manager**
2. Click **Create Secret**
3. Name: `db-password`, value: the password you set for `workshop_user` in Cloud SQL
4. Click **Create Secret**
5. Open the secret, go to **Permissions**, click **Grant Access**
6. Principal: `cloudrun-workshop-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com`
7. Role: **Secret Manager Secret Accessor**
8. Click **Save**

---

### 3.4 Create the Pub/Sub Topic

1. Go to **Pub/Sub > Topics**
2. Click **Create Topic**, Topic ID: `order-events`
3. Click **Create**

---

### 3.5 Deploy the Order Service from GitHub

1. Go to **Cloud Run**, click **Create Service**
2. Select **Continuously deploy from a repository (source or function)**
3. Click **Set up with Cloud Build**
4. Click **Connect a new repository**, choose **GitHub** as the provider
5. Authenticate with your GitHub account when prompted
6. Select the `order-service` repository, click **Next**
7. Build configuration: select **Dockerfile**, location `/Dockerfile`
8. Click **Save**

Configure the service:

| Field | Value |
|---|---|
| Service name | `order-service` |
| Region | `us-central1` |
| Authentication | Allow unauthenticated invocations |

9. Expand **Container, Volumes, Networking, Security**

**Container tab:**

| Field | Value |
|---|---|
| Port | 8080 |
| Memory | 512 MiB |
| CPU | 1 |
| Minimum instances | 0 |
| Maximum instances | 10 |
| Concurrency | 80 |

**Variables & Secrets — environment variables:**

| Name | Value |
|---|---|
| DB_HOST | `/cloudsql/YOUR_PROJECT_ID:us-central1:workshop-postgres` |
| DB_USER | `workshop_user` |
| DB_NAME | `orders` |
| PROJECT_ID | `YOUR_PROJECT_ID` |

**Variables & Secrets — secret reference:**

| Field | Value |
|---|---|
| Secret | `db-password` |
| Reference method | Exposed as environment variable |
| Environment variable name | `DB_PASS` |
| Version | latest |

**Cloud SQL connections:** Click **Add Connection**, select `workshop-postgres`

**Security tab:** Service account: `cloudrun-workshop-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com`

10. Click **Create**
11. Cloud Run triggers a build from your GitHub repository automatically
12. Once the green checkmark appears, copy the service URL

> Every future commit pushed to the `order-service` `main` branch triggers an automatic rebuild and new revision.

---

### 3.6 Deploy the Notification Service from GitHub

1. Go to **Cloud Run**, click **Create Service**
2. Select **Continuously deploy from a repository (source or function)**
3. Click **Set up with Cloud Build**
4. Connect the `notification-service` GitHub repository
5. Build configuration: **Dockerfile**, location `/Dockerfile`, click **Save**

| Field | Value |
|---|---|
| Service name | `notification-service` |
| Region | `us-central1` |
| Authentication | Require authentication |

**Container tab:**

| Field | Value |
|---|---|
| Port | 8080 |
| Memory | 256 MiB |
| Maximum instances | 5 |
| Concurrency | 20 |

**Networking tab:** Ingress: **Internal**

**Security tab:** Service account: `cloudrun-workshop-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com`

6. Click **Create**, wait for the green checkmark, copy the service URL

---

### 3.7 Connect Pub/Sub to the Notification Service

#### Create the Pub/Sub invoker service account

1. Go to **IAM & Admin > Service Accounts**
2. Click **Create Service Account**, name: `pubsub-invoker`
3. Click **Create and Continue**, then **Done**

#### Grant it permission to invoke the Notification Service

1. Go to **Cloud Run**, click `notification-service`
2. Click the **Security** tab, click **Add Principal**
3. Principal: `pubsub-invoker@YOUR_PROJECT_ID.iam.gserviceaccount.com`
4. Role: **Cloud Run Invoker**
5. Click **Save**

#### Create the push subscription

1. Go to **Pub/Sub > Subscriptions**, click **Create Subscription**

| Field | Value |
|---|---|
| Subscription ID | `order-events-notif-sub` |
| Topic | `order-events` |
| Delivery type | Push |
| Endpoint URL | `https://YOUR-NOTIFICATION-SERVICE-URL/pubsub` |
| Enable authentication | Toggle ON |
| Service account | `pubsub-invoker@YOUR_PROJECT_ID.iam.gserviceaccount.com` |
| Acknowledgement deadline | 60 seconds |
| Retry policy | Retry after exponential backoff |
| Minimum backoff | 10 seconds |
| Maximum backoff | 300 seconds |

2. Click **Create**

---

### 3.8 Validate the End-to-End Flow

#### Test 1 — Trigger the flow by publishing a Pub/Sub message

1. Go to **Pub/Sub > Topics**, click `order-events`
2. Click the **Messages** tab, then **Publish Message**
3. Paste in the message body:
   ```json
   {"order_id": 1, "customer_email": "alice@example.com", "amount": 49.99}
   ```
4. Click **Publish**
5. Go to **Cloud Run > notification-service > Logs**
6. You should see:
   ```
   [NOTIFICATION] Sending confirmation to alice@example.com for order #1 ($49.99)
   ```

#### Test 2 — Check autoscaling on the Metrics tab

1. Go to **Cloud Run > order-service**, click the **Metrics** tab
2. Observe these charts in real time:
   - **Request count** — requests per second
   - **Request latency** — p50, p95, p99 response times
   - **Instance count** — number of active instances (watch it scale up, then back to zero)
   - **Container instance hours** — your cost indicator
3. Publish several Pub/Sub messages in quick succession and watch the Instance count chart respond

#### Test 3 — Verify records in Cloud SQL Studio

1. Go to **SQL > workshop-postgres**
2. Click **Cloud SQL Studio** in the left sidebar
3. Sign in with username `workshop_user` and your password, select database `orders`
4. Run the following query:

```sql
SELECT id, customer_email, amount, status, created_at
FROM orders
ORDER BY created_at DESC
LIMIT 20;
```

You should see one row for each order event processed.

#### Test 4 — Check Pub/Sub delivery health

1. Go to **Pub/Sub > Subscriptions**, click `order-events-notif-sub`
2. Open the **Activity** tab and review:
   - **Undelivered message count** — should be 0 if all messages were acknowledged
   - **Oldest undelivered message age** — should be very low
   - **Sent message count** — should match the number of messages you published

#### Test 5 — Trigger a continuous deployment update

1. Open the `notification-service` GitHub repository
2. Edit `app.py` in the web editor — update the log message, for example:
   ```python
   print(f"[NOTIFICATION] Order #{order['order_id']} confirmed for {order['customer_email']}")
   ```
3. Commit directly to `main`
4. Go to **Cloud Run > notification-service > Revisions** tab
5. A new revision appears automatically within 1–2 minutes, receiving 100% of traffic

---

## Cleanup

#### Delete Cloud Run services

1. Go to **Cloud Run**
2. Select `order-service`, click **Delete**, confirm
3. Repeat for `notification-service`

#### Delete Pub/Sub resources

1. Go to **Pub/Sub > Subscriptions**, select `order-events-notif-sub`, click **Delete**
2. Go to **Pub/Sub > Topics**, select `order-events`, click **Delete**
3. If created: also delete `order-events-dlq`

#### Delete Cloud SQL instance

1. Go to **SQL**, click `workshop-postgres`
2. Click **Delete**, type the instance ID to confirm, click **Delete**

#### Delete secrets

1. Go to **Security > Secret Manager**
2. Select `db-password`, click **Delete**, confirm

#### Delete service accounts

1. Go to **IAM & Admin > Service Accounts**
2. Delete `cloudrun-workshop-sa` and `pubsub-invoker`

---

## Quick Reference — Where to Find Everything in the GCP Console

| Resource | Console path |
|---|---|
| Cloud Run services | Navigation menu > Serverless > Cloud Run |
| Cloud SQL | Navigation menu > Databases > SQL |
| Cloud SQL Studio | SQL > select instance > Cloud SQL Studio |
| Pub/Sub Topics | Navigation menu > Analytics > Pub/Sub > Topics |
| Pub/Sub Subscriptions | Navigation menu > Analytics > Pub/Sub > Subscriptions |
| Secret Manager | Navigation menu > Security > Secret Manager |
| IAM Permissions | Navigation menu > IAM & Admin > IAM |
| Service Accounts | Navigation menu > IAM & Admin > Service Accounts |
| APIs & Services | Navigation menu > APIs & Services > Library |
| Logs Explorer | Navigation menu > Logging > Logs Explorer |
| Eventarc Triggers | Navigation menu > Integration Services > Eventarc |

---

*Cloud Run Workshop (GCP Console UI Edition) — End of document*
