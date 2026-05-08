# SRE Dashboards with GCP Compute Engine, Cloud Monitoring & Cloud Logging

## 2-Hour Hands-On Workshop (Single VM Edition)

---

**Duration:** 2 hours  
**Level:** Intermediate  
**Prerequisites:** GCP account with Compute Engine, Cloud Monitoring, and Cloud Logging access (free trial eligible)

---

## Learning Objectives

- Configure Cloud Monitoring for a Compute Engine instance
- Set up Cloud Logging for application log collection
- Create SRE dashboards with metrics and log-based widgets
- Write Cloud Logging queries (Log Analytics / MQL)
- Set up alerting policies and notification channels
- Implement SRE metrics — SLIs, SLOs, and Error Budgets

---

## Part 1: SRE Concepts Overview

### The Four Golden Signals

| Signal | GCP Metric Example |
|--------|--------------------|
| **Latency** | Request/response time from Nginx logs |
| **Traffic** | `compute.googleapis.com/instance/network/received_bytes_count` |
| **Errors** | HTTP 5xx count from log-based metrics |
| **Saturation** | CPU utilization, memory, disk usage |

### SRE Targets

- **SLI (Service Level Indicator):** Measurable metric — e.g., % of requests < 200ms  
- **SLO (Service Level Objective):** Target — e.g., 99.9% availability (43.8 min downtime/month)  
- **Error Budget:** Remaining allowable downtime = `100% - SLO`

---

## Part 2: Enable APIs & IAM Setup

### Step 1: Open Google Cloud Console

- Go to [https://console.cloud.google.com](https://console.cloud.google.com)
- Select or create a **project** (e.g., `sre-workshop`)

### Step 2: Enable Required APIs

In the top search bar, search for and enable each API:

```
Cloud Monitoring API
Cloud Logging API
Compute Engine API
```

Or use Cloud Shell (click the `>_` icon in the top right):

```bash
gcloud services enable \
  monitoring.googleapis.com \
  logging.googleapis.com \
  compute.googleapis.com
```

### Step 3: Verify Default Service Account Permissions

The default Compute Engine service account already has **Logs Writer** and **Monitoring Metric Writer** roles. Verify this:

- Navigate to **IAM & Admin** → **IAM**
- Find the service account ending in `-compute@developer.gserviceaccount.com`
- Confirm it has `roles/logging.logWriter` and `roles/monitoring.metricWriter`

> ✅ GCP assigns these roles automatically to the default Compute Engine service account — no manual IAM role creation needed.

---

## Part 3: Launch a Single Compute Engine VM

### Step 1: Navigate to Compute Engine

- Search for **Compute Engine** in the top bar
- Click **VM instances**
- Click **Create Instance**

### Step 2: Configure the VM

| Field | Value |
|-------|-------|
| **Name** | `sre-workshop-web-01` |
| **Region** | `us-central1` |
| **Zone** | `us-central1-a` |
| **Machine type** | `e2-micro` (free tier eligible) |
| **Boot disk OS** | Debian 12 (Bookworm) |
| **Boot disk size** | 10 GB |
| **Firewall** | ✅ Allow HTTP traffic, ✅ Allow HTTPS traffic |

Under **Identity and API access**:
- Service account: **Compute Engine default service account**
- Access scopes: **Allow full access to all Cloud APIs**

### Step 3: Enable Ops Agent During VM Creation *(Recommended)*

> ✅ **This is the easiest way** — GCP can install the Ops Agent automatically when the VM is created, so you skip manual installation entirely.

While still on the **Create an instance** page:

1. Scroll down to the **Observability** section (below the firewall settings)
2. You will see:

   ```
   Ops Agent (Recommended) ⓘ
   Observe your instance and application through collection of logs and metrics.
   ☑ Install Ops Agent for Monitoring and Logging
   ```

3. **Check the box** — `☑ Install Ops Agent for Monitoring and Logging`
4. Click **Create**

> ✅ The Ops Agent will be installed and started automatically when the VM boots. No SSH commands needed for installation.

### Step 4: Add Labels

- Select `sre-workshop-web-01` → click **Edit**
- Scroll to **Labels** → click **Add label**

| Key | Value |
|-----|-------|
| `environment` | `workshop` |
| `application` | `webserver` |
| `team` | `sre` |

- Click **Save**

---

## Part 4: Verify Ops Agent & Configure Nginx Logging

### Step 1: Connect to VM via SSH

- In **Compute Engine** → **VM instances**
- Click **SSH** button next to `sre-workshop-web-01`

### Step 2: Verify the Ops Agent is Running

Since the agent was installed automatically at VM creation, just verify its status:

```bash
sudo systemctl status google-cloud-ops-agent
```

You should see `active (running)`. If not, wait 1–2 minutes and retry, or install manually:

```bash
# Manual install fallback (only if the checkbox method failed)
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install
```

### Step 3: Configure Ops Agent to Collect Nginx Logs

The default Ops Agent config collects system metrics and syslog. Add Nginx log collection:

```bash
sudo tee /etc/google-cloud-ops-agent/config.yaml > /dev/null << 'EOF'
logging:
  receivers:
    syslog:
      type: files
      include_paths:
        - /var/log/syslog
  service:
    pipelines:
      syslog_pipeline:
        receivers: [syslog]
metrics:
  receivers:
    hostmetrics:
      type: hostmetrics
      collection_interval: 60s
  service:
    pipelines:
      default_pipeline:
        receivers: [hostmetrics]
EOF
```

Press `Ctrl+X`, then `Y`, then `Enter` to save.

### Step 4: Restart the Ops Agent

```bash
sudo systemctl restart google-cloud-ops-agent
sudo systemctl status google-cloud-ops-agent
```

### Step 5: Verify Metrics Are Flowing

Wait 2–3 minutes, then check:

```bash
sudo journalctl -u google-cloud-ops-agent -n 50
```

Look for lines like `Successfully sent X metrics`.

---

## Part 5: Install & Configure Nginx

### Step 1: Install Nginx

```bash
sudo apt-get update -y
sudo apt-get install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Step 2: Verify Nginx is Running

```bash
sudo systemctl status nginx
curl -s http://localhost/ | head -5
```

### Step 3: Create a Custom Application Page

```bash
sudo bash -c 'cat > /var/www/html/index.html << EOF
<!DOCTYPE html>
<html>
<head><title>SRE Workshop - GCP</title></head>
<body>
  <h1>SRE Workshop - GCP Monitoring</h1>
  <p>Instance: sre-workshop-web-01</p>
  <p>Monitored by Google Cloud Ops Agent</p>
</body>
</html>
EOF'
```

### Step 4: Verify Logs Are Being Collected

```bash
sudo tail -f /var/log/nginx/access.log
```

Open another browser tab and hit your VM's **External IP** on port 80 (from the VM instances page). You should see log entries appear.

---

## Part 6: Explore Cloud Monitoring Metrics

### Step 1: Open Cloud Monitoring

- Search for **Monitoring** in the top bar
- The **Overview** page shows your project health

### Step 2: Explore Metrics Explorer

- Click **Metrics Explorer** in the left menu
- In the **Select a metric** field, search for:

```
compute.googleapis.com/instance/cpu/utilization
```

- Select it — you'll see CPU for your VM

### Step 3: Key GCP Metrics to Know

| Metric | Path |
|--------|------|
| CPU utilization | `compute.googleapis.com/instance/cpu/utilization` |
| Memory usage | `agent.googleapis.com/memory/percent_used` |
| Disk usage % | `agent.googleapis.com/disk/percent_used` |
| Network bytes in | `compute.googleapis.com/instance/network/received_bytes_count` |
| Network bytes out | `compute.googleapis.com/instance/network/sent_bytes_count` |
| Uptime check | `monitoring.googleapis.com/uptime_check/check_passed` |

---

## Part 7: Create SRE Dashboard

### Step 1: Navigate to Dashboards

- Click **Dashboards** in the left menu → **Create Dashboard**
- Name: `SRE Production Dashboard - GCP`
- Click **Confirm**

---

### Widget 1: CPU Utilization (Line Chart)

- Click **Add Widget** → **Line Chart**
- **Metric:** `compute.googleapis.com/instance/cpu/utilization`
- **Group by:** `instance_name`
- **Aggregation:** `mean`
- Title: `CPU Utilization (%)`
- Click **Apply**

---

### Widget 2: Memory Usage (Line Chart)

- Click **Add Widget** → **Line Chart**
- **Metric:** `agent.googleapis.com/memory/percent_used`
- **Filter:** `state = used`
- **Group by:** `instance_name`
- Title: `Memory Usage (%)`
- Click **Apply**

---

### Widget 3: Disk Usage (Scorecard)

- Click **Add Widget** → **Scorecard**
- **Metric:** `agent.googleapis.com/disk/percent_used`
- **Aggregation:** `mean`
- Title: `Disk Usage (%)`
- Click **Apply**

---

### Widget 4: Network Traffic (Stacked Area)

- Click **Add Widget** → **Stacked Area**
- **Metric:** `compute.googleapis.com/instance/network/received_bytes_count`
- **Group by:** `instance_name`
- Title: `Inbound Network Traffic`
- Click **Apply**

---

### Widget 5: Log-Based Error Count (Line Chart)

First, create a log-based metric:

1. Go to **Cloud Logging** → **Log-based Metrics** (left menu)
2. Click **Create Metric**
3. Configure:
   - **Metric type:** Counter
   - **Name:** `nginx_500_errors`
   - **Description:** `HTTP 500 errors from Nginx`
4. In the **Filter** box:
   ```
   resource.type="gce_instance"
   log_id("nginx_access")
   httpRequest.status>=500
   ```
5. Click **Create Metric**

Back in Dashboard:
- Click **Add Widget** → **Line Chart**
- **Metric:** `VM Instance - logging/user/nginx_500_errors`
- Title: `HTTP 5xx Error Rate`
- Click **Apply**

---

### Widget 6: Uptime Check Status

First, create an uptime check:

1. Go to **Monitoring** → **Uptime checks** → **Create Uptime Check**
2. Configure:
   - **Title:** `Web Server Uptime`
   - **Protocol:** HTTP
   - **Resource type:** Instance
   - **Applies to:** Select `sre-workshop-web-01`
   - **Path:** `/`
   - **Check frequency:** 1 minute
#other values keep it default
3. Click **Continue** → **Create**

Back in Dashboard:
- Click **Add Widget** → **Scorecard**
- **Metric:** `check for uptime`
- Title: `Uptime Check Status`
- Click **Apply**

---

### Widget 7: Logs Panel (Live Logs)

- Click **Add Widget** → **Logs Panel**
- **Filter:**
  ```
  resource.type="gce_instance"
  log_id("nginx_access")
  ```
- Title: `Nginx Access Logs - Live`
- Click **Apply**

---

### Step 2: Arrange and Save Dashboard

- Drag widgets to your preferred layout:
  - Row 1: Uptime status (full width)
  - Row 2: CPU | Memory | Disk
  - Row 3: Network traffic | 5xx errors
  - Row 4: Live logs (full width)
- Click **Save** (top right)

> ✅ You now have a full SRE dashboard combining infrastructure metrics, error tracking, uptime, and live logs!

---

## Part 8: Set Up Alerting Policies

### Step 1: Create Notification Channel

- Click **Alerting** → **Edit notification channels**
- Under **Email**, click **Add new**
- Enter your email address
- Click **Save**

---

### Alert 1: High CPU

- Click **Alerting** → **Create Policy**
- **Metric:** `compute.googleapis.com/instance/cpu/utilization`
- **Condition type:** Threshold
- **Threshold:** `0.8` (80%)
- **Duration:** 5 minutes
- **Condition name:** `High CPU Utilization`
- Click **Next** → Select your email channel
- **Alert name:** `SRE - High CPU Alert`
- Click **Create Policy**

---

### Alert 2: High Memory

- **Metric:** `agent.googleapis.com/memory/percent_used`
- **Filter:** `state = used`
- **Threshold:** `85`
- **Duration:** 5 minutes
- **Alert name:** `SRE - High Memory Alert`

---

### Alert 3: Log-Based 5xx Error Spike

- Try yourself

---

## Part 9: Cloud Logging Queries

### Step 1: Open Logs Explorer

- Go to  → **Logs Explorer**
- Set time range: **Last 1 hour**

---

### Query 1: All Nginx Access Logs

```
resource.type="gce_instance"
log_id("nginx_access")
```

### Query 2: HTTP 4xx and 5xx Errors

```
resource.type="gce_instance"
log_id("nginx_access")
httpRequest.status>=400
```

### Query 3: 5xx Errors Only

```
resource.type="gce_instance"
log_id("nginx_access")
httpRequest.status>=500
```

### Query 4: Nginx Error Logs

```
resource.type="gce_instance"
log_id("nginx_error")
severity>=ERROR
```

### Query 5: System Logs (Syslog)

```
resource.type="gce_instance"
log_id("syslog")
severity>=WARNING
```

---



---

##  SLO Tracking
>  GCP SLOs require a **Service** to be defined first. Compute Engine VMs are not auto-detected — you must create a Custom service manually before an SLO can be attached.

### Step 1: Define a Custom Service

- Go to **Monitoring** → **SLOs**
- Click **+ Define service** ( top of page)
- Under **Select service type**, click **Custom**
- Fill in:
  - **Service ID:** `sre-workshop-web`
  - **Display name:** `SRE Workshop Web Server`
- Click **Submit**

### Step 2: Create an SLO on the Service

- You will be taken to the service detail page
- Click **+ Create SLO**
- Configure:
  - **SLI type:** `Availability` or go for none if not selected
  - **Request-based or Window-based:** `Request-based`
  - Click **Continue**
- Set the metric:
  - **SLI metric:** Use uptime check pass rate
    - Select serach for `uptime`
  - Click **Continue**
- Set the SLO target:
  - **Goal:** `99.9`%
  - **Compliance period:** Rolling `30` days
  - Click **Continue**
- Review and click **Create SLO**

### Step 3: Add SLO Widget to Dashboard

- Open **SRE Production Dashboard - GCP**
- Click **Add Widget** → **SLO**
- Select the SLO you just created (`SRE Workshop Web Server`)
- Title: `Availability SLO - 99.9%`
- Click **Apply** → **Save**




---

