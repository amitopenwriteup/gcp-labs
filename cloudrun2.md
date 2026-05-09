# Module 2 — Hands-on Lab: Deploying Microservices to Cloud Run

## Lab Architecture

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

## 3.1 Prepare Source Code in GitHub

Both services are deployed using Cloud Run's **Deploy from source** feature, which reads source code from a connected GitHub repository and handles the container build automatically in the background.

### Create the GitHub repositories

1. Log in to https://github.com
2. Create a new public repository named `order-service`
3. Create a second public repository named `notification-service`

### Order Service files

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

### Notification Service files

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

## 3.2 Create the Cloud SQL Database

> **Do this before deploying any Cloud Run service** — the Order Service needs the database connection details as environment variables.

### 3.2.1 Create the Cloud SQL Instance

1. In the GCP Console, go to **Navigation menu → Databases → SQL**
2. Click **+ Create Instance**
3. Choose **PostgreSQL**, then click **Choose PostgreSQL**
4. Fill in the form:

| Field | Value |
|---|---|
| Instance ID | `workshop-postgres` |
| Password | Choose a strong password — save it, you'll need it later |
| Database version | **PostgreSQL 15** |
| Region | `us-central1` |
| Zonal availability | **Single zone** (sufficient for this lab) |

5. Expand **Machine configuration**:

| Field | Value |
|---|---|
| Machine type | **Lightweight** — 1 vCPU, 3.75 GB RAM |

6. Expand **Storage**:

| Field | Value |
|---|---|
| Storage type | **SSD** |
| Storage capacity | **10 GB** |

7. Click **Create Instance** — this takes 3–5 minutes.

**Checkpoint:** You see `workshop-postgres` with a green status indicator in the SQL instances list. ✓

---

### 3.2.2 Create the Database

1. Click **workshop-postgres** in the instances list
2. In the left panel, click **Databases**
3. Click **+ Create Database**
4. Enter:

| Field | Value |
|---|---|
| Database name | `orders` |

5. Click **Create**

**Checkpoint:** `orders` appears in the databases list. ✓

---

### 3.2.3 Create the Database User

1. In the left panel, click **Users**
2. Click **+ Add User Account**
3. Select **Built-in authentication**
4. Fill in:

| Field | Value |
|---|---|
| Username | `workshop_user` |
| Password | Choose a strong password — save it for use in Secret Manager |

5. Click **Add**

**Checkpoint:** `workshop_user` appears in the users list. ✓

---

### 3.2.4 Note the Connection Name

1. In the left panel, click **Overview**
2. Scroll down to the **Connect to this instance** section
3. Copy the **Connection name** — it looks like:

```
YOUR_PROJECT_ID:us-central1:workshop-postgres
```

You will need this value when configuring the `DB_HOST` environment variable in Cloud Run:

```
/cloudsql/YOUR_PROJECT_ID:us-central1:workshop-postgres
```

---

## 3.3 Create the Shared Service Account

1. Go to **IAM & Admin → Service Accounts**
2. Click **Create Service Account**
3. Name: `cloudrun-workshop-sa`, Description: `Workshop service account`
4. Click **Create and Continue**
5. Add the following roles (click **Add Another Role** after each):
   - Cloud SQL Client
   - Secret Manager Secret Accessor
   - Pub/Sub Publisher
   - Pub/Sub Subscriber
6. Click **Done**

---

## 3.4 Store the Database Password in Secret Manager

1. Go to **Security → Secret Manager**
2. Click **Create Secret**
3. Name: `db-password`, value: the password you set for `workshop_user` in Cloud SQL
4. Click **Create Secret**
5. Open the secret, go to **Permissions**, click **Grant Access**
   - Principal: `cloudrun-workshop-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com`
   - Role: **Secret Manager Secret Accessor**
6. Click **Save**

---

## 3.5 Create the Pub/Sub Topic

1. Go to **Pub/Sub → Topics**
2. Click **Create Topic**, Topic ID: `order-events`
3. Click **Create**

---

## 3.6 Deploy the Order Service from GitHub

1. Go to **Cloud Run**, click **Create Service**
2. Select **Continuously deploy from a repository (source or function)**
3. Click **Set up with Cloud Build**
4. Click **Connect a new repository**, choose **GitHub** as the provider
5. Authenticate with your GitHub account when prompted
6. Select the `order-service` repository, click **Next**
7. Build configuration: select **Dockerfile**, location `/Dockerfile`
8. Click **Save**
9. Configure the service:

| Field | Value |
|---|---|
| Service name | `order-service` |
| Region | `us-central1` |
| Authentication | Allow unauthenticated invocations |

10. Expand **Container, Volumes, Networking, Security**

**Container tab:**

| Field | Value |
|---|---|
| Port | `8080` |
| Memory | `512 MiB` |
| CPU | `1` |
| Minimum instances | `0` |
| Maximum instances | `10` |
| Concurrency | `80` |

**Variables & Secrets — environment variables:**

| Name | Value |
|---|---|
| `DB_HOST` | `/cloudsql/YOUR_PROJECT_ID:us-central1:workshop-postgres` |
| `DB_USER` | `workshop_user` |
| `DB_NAME` | `orders` |
| `PROJECT_ID` | `YOUR_PROJECT_ID` |

**Variables & Secrets — secret reference:**

| Field | Value |
|---|---|
| Secret | `db-password` |
| Reference method | Exposed as environment variable |
| Environment variable name | `DB_PASS` |
| Version | `latest` |

**Cloud SQL connections:** Click **Add Connection**, select `workshop-postgres`

**Security tab:** Service account: `cloudrun-workshop-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com`

11. Click **Create**
12. Once the green checkmark appears, copy the service URL

> Every future commit pushed to the `order-service` main branch triggers an automatic rebuild and new revision.

---

## 3.7 Deploy the Notification Service from GitHub

1. Go to **Cloud Run**, click **Create Service**
2. Select **Continuously deploy from a repository (source or function)**
3. Click **Set up with Cloud Build**
4. Connect the `notification-service` GitHub repository
5. Build configuration: Dockerfile, location `/Dockerfile`, click **Save**

| Field | Value |
|---|---|
| Service name | `notification-service` |
| Region | `us-central1` |
| Authentication | Require authentication |

**Container tab:**

| Field | Value |
|---|---|
| Port | `8080` |
| Memory | `256 MiB` |
| Maximum instances | `5` |
| Concurrency | `20` |

**Networking tab:** Ingress: **Internal**

**Security tab:** Service account: `cloudrun-workshop-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com`

6. Click **Create**, wait for the green checkmark, copy the service URL

---

## 3.8 Connect Pub/Sub to the Notification Service

### Create the Pub/Sub invoker service account

1. Go to **IAM & Admin → Service Accounts**
2. Click **Create Service Account**, name: `pubsub-invoker`
3. Click **Create and Continue**, then **Done**

### Grant it permission to invoke the Notification Service

1. Go to **Cloud Run**, click `notification-service`
2. Click the **Security** tab, click **Add Principal**
   - Principal: `pubsub-invoker@YOUR_PROJECT_ID.iam.gserviceaccount.com`
   - Role: **Cloud Run Invoker**
3. Click **Save**

### Create the push subscription

1. Go to **Pub/Sub → Subscriptions**, click **Create Subscription**

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

## 3.9 Validate the End-to-End Flow

### Test 1 — Trigger the flow by publishing a Pub/Sub message

1. Go to **Pub/Sub → Topics**, click `order-events`
2. Click the **Messages** tab, then **Publish Message**
3. Paste in the message body:
```json
{"order_id": 1, "customer_email": "alice@example.com", "amount": 49.99}
```
4. Click **Publish**
5. Go to **Cloud Run → notification-service → Logs**
6. You should see:
```
[NOTIFICATION] Sending confirmation to alice@example.com for order #1 ($49.99)
```

### Test 2 — Check autoscaling on the Metrics tab

1. Go to **Cloud Run → order-service**, click the **Metrics** tab
2. Observe these charts in real time:
   - **Request count** — requests per second
   - **Request latency** — p50, p95, p99 response times
   - **Instance count** — number of active instances (watch it scale up, then back to zero)
   - **Container instance hours** — your cost indicator
3. Publish several Pub/Sub messages in quick succession and watch the Instance count chart respond

### Test 3 — Verify records in Cloud SQL Studio

1. Go to **SQL → workshop-postgres**
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

### Test 4 — Check Pub/Sub delivery health

1. Go to **Pub/Sub → Subscriptions**, click `order-events-notif-sub`
2. Open the **Activity** tab and review:
   - **Undelivered message count** — should be `0` if all messages were acknowledged
   - **Oldest undelivered message age** — should be very low
   - **Sent message count** — should match the number of messages you published

### Test 5 — Trigger a continuous deployment update

1. Open the `notification-service` GitHub repository
2. Edit `app.py` in the web editor — update the log message, for example:
```python
print(f"[NOTIFICATION] Order #{order['order_id']} confirmed for {order['customer_email']}")
```
3. Commit directly to `main`
4. Go to **Cloud Run → notification-service → Revisions** tab
5. A new revision appears automatically within 1–2 minutes, receiving 100% of traffic

---

## Cleanup

### Delete Cloud Run services

1. Go to **Cloud Run**
2. Select `order-service`, click **Delete**, confirm
3. Repeat for `notification-service`

### Delete Pub/Sub resources

1. Go to **Pub/Sub → Subscriptions**, select `order-events-notif-sub`, click **Delete**
2. Go to **Pub/Sub → Topics**, select `order-events`, click **Delete**

### Delete Cloud SQL instance

1. Go to **SQL**, click `workshop-postgres`
2. Click **Delete**, type the instance ID to confirm, click **Delete**

### Delete secrets

1. Go to **Security → Secret Manager**
2. Select `db-password`, click **Delete**, confirm

### Delete service accounts

1. Go to **IAM & Admin → Service Accounts**
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
