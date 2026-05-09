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

> **Troubleshooting — `--- Latest trips ---` appears but no rows print:**
> This means the rows were written successfully but the read stream was not consumed before iterating. Make sure you have `rows.consume_all()` followed by `for row_key, row in rows.rows.items()` exactly as shown in step 3.1.4. A plain `for row in rows` loop can silently return empty.

#### 3.1.8 — Verify in the GCP Console

1. Go to **Bigtable → rideshare-instance → Tables**
2. Click **trips**
3. Click **Read rows**
4. Confirm you see your row key `trip#9223...#trip-abc-001`

**Checkpoint:** Row key `trip#9223...` is visible in the Console table browser. ✓

### 3.2 Add column families via the Console

Column families group related columns and carry garbage collection (GC) policies.

#### For the `users` table

1. Click **users** in the tables list
2. Click **+ Add column family**
3. Enter:

| Field | Value |
|---|---|
| Column family name | `profile` |
| GC policy | **Max versions** → `1` |

4. Click **Save**

#### For the `trips` table

1. Click **trips** → **+ Add column family**
2. Add the `details` family:

| Field | Value |
|---|---|
| Column family name | `details` |
| GC policy | **Max age** → `90 days` |

3. Click **Save**, then add a second family:

| Field | Value |
|---|---|
| Column family name | `payment` |
| GC policy | **Max versions** → `1` |

4. Click **Save**

#### For the `driver_locations` table

1. Click **driver_locations** → **+ Add column family**
2. Add the `loc` family:

| Field | Value |
|---|---|
| Column family name | `loc` |
| GC policy | **Max age** → `1 day` |

3. Click **Save**

**Checkpoint — full column family list:**

| Table | Column family | Qualifiers | GC policy |
|---|---|---|---|
| users | profile | name, email, phone, rating | Max versions: 1 |
| trips | details | driver_id, passenger_id, origin, destination | Max age: 90 days |
| trips | payment | amount, method, status | Max versions: 1 |
| driver_locations | loc | lat, lng, speed, heading | Max age: 1 day |

---

## Step 4 — Key Visualizer (hotspot detection)

Key Visualizer requires at least 30 GB of data or 10,000 rows. For this lab you will **read** the visualizer and interpret a simulated hotspot.

### 4.1 Open Key Visualizer

1. In your instance page, click **Key Visualizer** in the left panel
2. Select the `trips` table
3. Select a time range of the last hour

### 4.2 Read the heatmap

The x-axis is time. The y-axis is the row key space (top = smallest key, bottom = largest key).

| Colour | Meaning |
|---|---|
| White / light green | Low activity — healthy |
| Orange | Moderate activity |
| Red / bright spot | Hotspot — one tablet is overloaded |

### 4.3 Identify hotspot patterns

**Sequential key hotspot (bad):**

If you used keys like `trip-000001`, `trip-000002` ... all new writes go to the bottom of the heatmap (the largest key), creating a permanent red band.

**Fix:** Switch to reverse timestamp prefix:

```
Before: trip-000001, trip-000002, trip-000003
After:  trip#9999862800000#abc-001
        trip#9999862799000#abc-002
```

Writes are now distributed across all tablets as different `trip#` ranges fill naturally.

**Checkpoint:** In Key Visualizer, confirm your `trips` table shows an even spread of reads/writes, with no persistent red band at one end of the key space.

---

## Step 5 — Cluster monitoring

### 5.1 Open the Monitoring dashboard

1. In your instance page, click **Monitoring** in the left panel
2. You will see charts for:
   - Read/write latency (p50, p95, p99)
   - Requests per second
   - CPU utilisation per node
   - Disk storage used

### 5.2 Key metrics to watch

| Metric | Healthy range | Action if breached |
|---|---|---|
| Read latency p99 | < 10ms (SSD) | Check row key design, increase nodes |
| CPU utilisation | < 70% | Add nodes — Bigtable scales linearly |
| Disk usage | < 70% | Add nodes or review GC policies |
| Error rate | 0% | Check IAM permissions and network |

### 5.3 Set an alerting policy

1. Click **Alerting** in the left menu → **+ Create policy**
2. Select metric: **Bigtable → Instance → Read latency (p99)**
3. Set threshold: `10ms`
4. Set notification: your email
5. Click **Save**

Repeat for **CPU utilisation > 70%**.

**Checkpoint:** Confirm the alerting policy appears in the policies list with status **No incidents**.

---

## Lab summary

You have completed all five steps. Here is what you built:

| Resource | Detail |
|---|---|
| Instance | `rideshare-instance` · us-central1-b · 3 nodes · SSD |
| Tables | users, trips, driver_locations |
| Column families | profile, details, payment, loc |
| Row key pattern | Reverse timestamp — hotspot-free |
| GC policies | 1 version / 90 days / 1 day (per family) |
| Monitoring | Latency + CPU alerting policies configured |

---

## Clean up

To avoid ongoing charges, delete the instance when the lab is done:

1. Go to **Bigtable → Instances**
2. Click **rideshare-instance**
3. Click **Delete instance**
4. Type the instance ID to confirm

Or via CLI:

```bash
gcloud bigtable instances delete rideshare-instance \
  --project=YOUR_PROJECT_ID
```

---

## Next steps

- [Bigtable schema design best practices](https://cloud.google.com/bigtable/docs/schema-design)
- [Key Visualizer guide](https://cloud.google.com/bigtable/docs/keyvis-overview)
- [Python client library reference](https://googleapis.dev/python/bigtable/latest/index.html)
- [Bigtable pricing](https://cloud.google.com/bigtable/pricing)
