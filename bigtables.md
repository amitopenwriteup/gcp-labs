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


