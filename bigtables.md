# Bigtable Lab Guide — Ride-Sharing App Schema Design

**Duration:** ~90 minutes  
**Level:** Intermediate  
**Prerequisites:** A GCP project with billing enabled, Owner or Editor role

---

## Lab overview

In this lab you will use the **Google Cloud Console** to:

1. Create a Bigtable instance
2. Create tables for a ride-sharing application
3. Design row keys and column families
4. Use Key Visualizer to detect hotspots
5. Review cluster monitoring metrics

---

## Before you begin

Open the GCP Console and make sure your project is selected in the top bar:

```
https://console.cloud.google.com
```

Enable the Bigtable API if prompted, or run this in Cloud Shell:

```bash
gcloud services enable bigtable.googleapis.com \
  bigtableadmin.googleapis.com \
  --project=YOUR_PROJECT_ID
```

---

## Step 1 — Create a Bigtable instance

### 1.1 Navigate to Bigtable

1. In the GCP Console left menu, click **Navigation menu (☰) → Bigtable**
2. Or go directly: `https://console.cloud.google.com/bigtable`
3. Click **+ Create instance**

### 1.2 Configure the instance

Fill in the form with these values:

| Field | Value |
|---|---|
| Instance name | `rideshare-instance` |
| Instance ID | `rideshare-instance` |
| Storage type | **SSD** (recommended for low latency) |
| Cluster ID | `rideshare-cluster-1` |
| Region | `us-central1` |
| Zone | `us-central1-b` |
| Nodes | `3` |

> **Why SSD?** SSD clusters deliver single-digit millisecond read latency. Use HDD only for large analytical workloads where cost matters more than latency.

> **Why 3 nodes?** Each node handles ~10,000 QPS. Three nodes gives headroom to scale and is the minimum recommended for production.

### 1.3 Create the instance

Click **Create** at the bottom of the form. The instance will be ready in ~2 minutes. You will see it listed with a green status indicator.

**Checkpoint:** Confirm you see `rideshare-instance` with status **Ready** in the Bigtable instances list.

---

## Step 2 — Create tables

### 2.1 Open your instance

1. Click **rideshare-instance** in the instances list
2. In the left panel, click **Tables**
3. Click **+ Create table**

### 2.2 Create the `users` table

| Field | Value |
|---|---|
| Table ID | `users` |

Click **Create**. Repeat for the next two tables.

### 2.3 Create the `trips` table

| Field | Value |
|---|---|
| Table ID | `trips` |

### 2.4 Create the `driver_locations` table

| Field | Value |
|---|---|
| Table ID | `driver_locations` |

**Checkpoint:** You should now see three tables listed:

```
users
trips
driver_locations
```

---

## Step 3 — Design the schema

Bigtable schema design happens primarily in code (the Python client) and in the **Column families** section of the Console. Follow both sub-steps below.

### 3.1 Design and implement row keys using Cloud Shell

The row key is your **only index** in Bigtable — rows are sorted lexicographically by it. There is no UI panel to configure row keys; you define them in code when writing rows. Follow the steps below to implement and verify row keys in Cloud Shell.

**Row key formats for this lab:**

| Table | Row key format | Rationale |
|---|---|---|
| `users` | `user#<user_id>` | Simple lookup by user ID |
| `trips` | `trip#<reverse_timestamp>#<trip_id>` | Most-recent trips appear first in scans |
| `driver_locations` | `driver#<driver_id>#<reverse_timestamp>` | Latest location per driver appears first |

> **Anti-pattern to avoid:** Do not use sequential numeric IDs (`trip-000001`, `trip-000002`) as the first component of a row key. All new writes go to the same tablet, creating a hotspot. You will see this in Step 4.

---

#### 3.1.1 — Open Cloud Shell

Click the **`>_`** icon in the top-right of the GCP Console toolbar. A terminal opens at the bottom of your browser.

#### 3.1.2 — Install the Bigtable library

```bash
pip install google-cloud-bigtable --quiet
```

#### 3.1.3 — Create the Python file

```bash
nano bigtable_lab.py
```

#### 3.1.4 — Paste this script

Copy the entire block below and paste it into the nano editor. Replace `YOUR_PROJECT_ID` with your actual GCP project ID (visible in the top bar of the Console).

```python
import time
from google.cloud import bigtable

PROJECT_ID  = "YOUR_PROJECT_ID"
INSTANCE_ID = "rideshare-instance"

client   = bigtable.Client(project=PROJECT_ID, admin=True)
instance = client.instance(INSTANCE_ID)

MAX_INT64 = 2**63 - 1

def reverse_ts():
    """
    Subtracts current time in ms from MAX_INT64.
    Result: the most recent row always has the SMALLEST key,
    so it sorts to the TOP in a prefix scan — newest first.
    """
    return str(MAX_INT64 - int(time.time() * 1000)).zfill(19)


# ── Write a user ──────────────────────────
table   = instance.table("users")
row_key = b"user#user-042"
row     = table.direct_row(row_key)
row.set_cell("profile", "name",   "Priya Sharma")
row.set_cell("profile", "email",  "priya@example.com")
row.set_cell("profile", "rating", "4.8")
row.commit()
print("Written:", row_key.decode())


# ── Write a trip ──────────────────────────
table = instance.table("trips")
key   = f"trip#{reverse_ts()}#trip-abc-001".encode()
row   = table.direct_row(key)
row.set_cell("details", "driver_id",    "driver-007")
row.set_cell("details", "passenger_id", "user-042")
row.set_cell("details", "origin",       "28.6139,77.2090")
row.set_cell("details", "destination",  "28.5355,77.3910")
row.set_cell("payment",  "amount",      "285.50")
row.set_cell("payment",  "method",      "UPI")
row.commit()
print("Written:", key.decode())


# ── Write a driver location ───────────────
table = instance.table("driver_locations")
key   = f"driver#driver-007#{reverse_ts()}".encode()
row   = table.direct_row(key)
row.set_cell("loc", "lat",     "28.6145")
row.set_cell("loc", "lng",     "77.2110")
row.set_cell("loc", "speed",   "32")
row.set_cell("loc", "heading", "NE")
row.commit()
print("Written:", key.decode())


# ── Read back latest 5 trips ──────────────
# Note: consume_all() fetches the full response stream before iterating.
# Without it, the for loop may return empty even though rows exist.
print("\n--- Latest trips ---")
table = instance.table("trips")
rows  = table.read_rows(start_key=b"trip#", end_key=b"trip$", limit=5)
rows.consume_all()

for row_key, row in rows.rows.items():
    key    = row_key.decode()
    origin = row.cells["details"][b"origin"][0].value.decode()
    amount = row.cells["payment"][b"amount"][0].value.decode()
    print(f"  {key}  |  origin: {origin}  |  ₹{amount}")
```

#### 3.1.5 — Save and exit nano

- Press `Ctrl + O` then `Enter` to save
- Press `Ctrl + X` to exit

#### 3.1.6 — Run the script

```bash
python bigtable_lab.py
```

#### 3.1.7 — Expected output

```
Written: user#user-042
Written: trip#9223370258508124749#trip-abc-001
Written: driver#driver-007#9223370258508124535

--- Latest trips ---
  trip#9223370258508124749#trip-abc-001  |  origin: 28.6139,77.2090  |  ₹285.50
```

The large number in the trip row key (`9223370258...`) is the reverse timestamp. Run the script again and the new row will have a slightly smaller number — so it sorts above the previous one, keeping newest trips at the top.

gle.com/bigtable/pricing)
