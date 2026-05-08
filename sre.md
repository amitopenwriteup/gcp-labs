# SRE Dashboards with GCP Compute Engine, Cloud Monitoring & Cloud Logging

## 2-Hour Hands-On Workshop

---

**Duration:** 2 hours  
**Level:** Intermediate  
**Prerequisites:** GCP account with Compute Engine, Cloud Monitoring, and Cloud Logging access (free trial eligible)

---

## Learning Objectives

- Configure Cloud Monitoring for Compute Engine instances
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

> ✅ **GCP assigns these roles automatically** to the default Compute Engine service account — no manual IAM role creation needed (unlike AWS IAM instance profiles).

---

## Part 3: Launch Compute Engine VMs

### Step 1: Navigate to Compute Engine

- Search for **Compute Engine** in the top bar
- Click **VM instances**
- Click **Create Instance**

### Step 2: Configure First VM

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

Click **Create**.

### Step 3: Create Second VM

Repeat the same configuration with:
- **Name:** `sre-workshop-web-02`
- **Zone:** `us-central1-b`

### Step 4: Add Labels (GCP equivalent of Tags)

- Select `sre-workshop-web-01` → click **Edit**
- Scroll to **Labels** → click **Add label**

| Key | Value |
|-----|-------|
| `environment` | `workshop` |
| `application` | `webserver` |
| `team` | `sre` |

- Click **Save**. Repeat for `sre-workshop-web-02`.

> ✅ Both VMs now have permissions to send metrics and logs to Cloud Monitoring and Cloud Logging via the attached service account.

---

## Part 4: Install Ops Agent & Configure Logging

The **Ops Agent** is GCP's unified agent that collects both **metrics** (replacing Stackdriver Monitoring agent) and **logs** (replacing Stackdriver Logging agent).

### Step 1: Connect to VM via SSH

- In **Compute Engine** → **VM instances**
- Click **SSH** button next to `sre-workshop-web-01`
- A browser-based terminal opens

### Step 2: Install the Ops Agent

```bash
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install
```

Verify installation:

```bash
sudo systemctl status google-cloud-ops-agent
```

### Step 3: Create Ops Agent Configuration

```bash
sudo nano /etc/google-cloud-ops-agent/config.yaml
```

Paste this configuration (collects metrics AND Nginx logs):

```yaml
logging:
  receivers:
    nginx_access:
      type: files
      include_paths:
        - /var/log/nginx/access.log
    nginx_error:
      type: files
      include_paths:
        - /var/log/nginx/error.log
    syslog:
      type: files
      include_paths:
        - /var/log/syslog
  processors:
    parse_nginx_access:
      type: parse_nginx_combined
  pipelines:
    nginx_access_pipeline:
      receivers: [nginx_access]
      processors: [parse_nginx_access]
    nginx_error_pipeline:
      receivers: [nginx_error]
    syslog_pipeline:
      receivers: [syslog]

metrics:
  receivers:
    hostmetrics:
      type: hostmetrics
      collection_interval: 60s
  processors:
    metrics_filter:
      type: exclude_metrics
      metrics_pattern: []
  pipelines:
    default_pipeline:
      receivers: [hostmetrics]
      processors: [metrics_filter]
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

> Repeat **Parts 4 and 5** on `sre-workshop-web-02` by SSH-ing into it.

---

## Part 6: Explore Cloud Monitoring Metrics

### Step 1: Open Cloud Monitoring

- Search for **Monitoring** in the top bar
- Click **Cloud Monitoring**
- The **Overview** page shows your project health

### Step 2: Explore Metrics Explorer

- Click **Metrics Explorer** in the left menu
- In the **Select a metric** field, search for:

```
compute.googleapis.com/instance/cpu/utilization
```

- Select it — you'll see CPU for both VMs
- Filter by label: `instance_name = sre-workshop-web-01`

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

We'll create a log-based metric first:

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
- **Metric:** `logging.googleapis.com/user/nginx_500_errors`
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
3. Click **Continue** → **Create**

Back in Dashboard:
- Click **Add Widget** → **Scorecard**
- **Metric:** `monitoring.googleapis.com/uptime_check/check_passed`
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

- Drag widgets to your preferred layout
- Suggested layout:
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
- Click **Select a metric**:
  ```
  compute.googleapis.com/instance/cpu/utilization
  ```
- **Condition type:** Threshold
- **Threshold:** `0.8` (80%)
- **Duration:** 5 minutes
- **Condition name:** `High CPU Utilization`
- Click **Next** → Select your email channel
- **Alert name:** `SRE - High CPU Alert`
- Click **Create Policy**

---

### Alert 2: High Memory

- Create another policy:
- **Metric:** `agent.googleapis.com/memory/percent_used`
- **Filter:** `state = used`
- **Threshold:** `85`
- **Duration:** 5 minutes
- **Alert name:** `SRE - High Memory Alert`

---

### Alert 3: Log-Based 5xx Error Spike

- **Metric:** `logging.googleapis.com/user/nginx_500_errors`
- **Threshold:** `10` (errors in 5 min window)
- **Alert name:** `SRE - HTTP 5xx Error Spike`

---

## Part 9: Cloud Logging Queries

### Step 1: Open Logs Explorer

- Go to **Cloud Logging** → **Logs Explorer**
- Set time range: **Last 1 hour**

---

### Query 1: All Nginx Access Logs

```
resource.type="gce_instance"
log_id("nginx_access")
```

---

### Query 2: HTTP 4xx and 5xx Errors

```
resource.type="gce_instance"
log_id("nginx_access")
httpRequest.status>=400
```

---

### Query 3: 5xx Errors Only

```
resource.type="gce_instance"
log_id("nginx_access")
httpRequest.status>=500
```

---

### Query 4: Nginx Error Logs

```
resource.type="gce_instance"
log_id("nginx_error")
severity>=ERROR
```

---

### Query 5: System Logs (Syslog)

```
resource.type="gce_instance"
log_id("syslog")
severity>=WARNING
```

---

### Step 2: Use Log Analytics (SQL-Style Queries)

- Click **Log Analytics** in the left menu
- Run aggregation queries using standard SQL:

**Count errors by hour:**
```sql
SELECT
  TIMESTAMP_TRUNC(timestamp, HOUR) AS hour,
  COUNT(*) AS error_count
FROM `YOUR_PROJECT_ID._Default._Default`
WHERE log_id = "nginx_access"
  AND http_request.status >= 500
  AND timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
GROUP BY hour
ORDER BY hour DESC
```

**Top 10 requested URLs:**
```sql
SELECT
  http_request.request_url AS url,
  COUNT(*) AS request_count
FROM `YOUR_PROJECT_ID._Default._Default`
WHERE log_id = "nginx_access"
  AND timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
GROUP BY url
ORDER BY request_count DESC
LIMIT 10
```

**Unique IPs by hour:**
```sql
SELECT
  TIMESTAMP_TRUNC(timestamp, HOUR) AS hour,
  COUNT(DISTINCT http_request.remote_ip) AS unique_ips
FROM `YOUR_PROJECT_ID._Default._Default`
WHERE log_id = "nginx_access"
  AND timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 6 HOUR)
GROUP BY hour
ORDER BY hour DESC
```

> **Replace** `YOUR_PROJECT_ID` with your actual GCP project ID (visible in the project selector at the top).

---

## Part 10: Load Testing & Validation

### Step 1: Generate CPU Load

SSH into `sre-workshop-web-01`:

```bash
sudo apt-get install stress -y
stress --cpu 2 --timeout 300s &
```

### Step 2: Generate Web Traffic with Errors

```bash
cat > /tmp/generate_traffic.sh << 'EOF'
#!/bin/bash
for i in {1..1000}; do
  # Normal requests
  curl -s http://localhost/ > /dev/null
  # 404 errors
  curl -s http://localhost/nonexistent-page > /dev/null
  # Random delay
  sleep 0.1
done
EOF
chmod +x /tmp/generate_traffic.sh
/tmp/generate_traffic.sh &
```

### Step 3: Monitor in Real Time

- Return to your **SRE Production Dashboard - GCP**
- Observe CPU spike in the CPU widget
- Watch the live logs panel update
- Check the 5xx error widget (404s won't trigger the 5xx metric, that's expected)

### Step 4: Verify Alerting

- Navigate to **Monitoring** → **Alerting**
- You should see an **incident** created for High CPU
- Check your email for the notification

### Step 5: Stop Load Test

```bash
pkill stress
pkill -f generate_traffic.sh
```

---

## Part 11: SLO Tracking

### Step 1: Create an SLO

- Go to **Monitoring** → **SLOs** → **Create SLO**
- **Service:** Select your uptime check service or create a custom service
- **SLI type:** Availability
- **Request-based SLI metric:** Use uptime check pass rate
- **SLO target:** `99.9%`
- **Compliance period:** Rolling 30 days
- Click **Create SLO**

### Step 2: Add SLO Widget to Dashboard

- Open **SRE Production Dashboard - GCP**
- Click **Add Widget** → **Error Budget**
- Select your SLO
- Title: `Availability SLO - 99.9%`
- Click **Apply** → **Save**

> ✅ You now have an error budget widget showing remaining downtime allowance for the month.

---

## Troubleshooting Guide

### Issue: Metrics Not Appearing in Cloud Monitoring

**Solutions:**
```bash
# Check Ops Agent status
sudo systemctl status google-cloud-ops-agent

# Check agent logs
sudo journalctl -u google-cloud-ops-agent -n 100

# Restart agent
sudo systemctl restart google-cloud-ops-agent
```

- Verify service account has `roles/monitoring.metricWriter`
- Wait 3–5 minutes for first metric data to appear

---

### Issue: Logs Not Appearing in Cloud Logging

**Solutions:**
```bash
# Check if Nginx is writing logs
sudo tail -f /var/log/nginx/access.log

# Verify agent config
sudo cat /etc/google-cloud-ops-agent/config.yaml

# Check agent errors
sudo journalctl -u google-cloud-ops-agent | grep -i error
```

- Verify service account has `roles/logging.logWriter`
- Confirm Nginx log paths match the config

---

### Issue: Log Analytics Query Fails

- Replace `YOUR_PROJECT_ID` with your actual project ID
- Ensure **Log Analytics** is enabled for the `_Default` log bucket:
  - Go to **Logging** → **Log Storage** → **_Default** bucket → **Edit** → Enable Log Analytics

---

## Workshop Summary & Best Practices

### Dashboard Organization (GCP)

- Combine infrastructure metrics (Ops Agent) and log-based metrics for full observability
- Use **Log Panels** for live tail views directly in dashboards
- Group related widgets — uptime at top, then metrics, then logs
- Enable **Auto Refresh** (top right of dashboard) for production monitoring

### Log Analysis Best Practices

- Use **structured logging** in apps (JSON format) for automatic field parsing
- Create **log-based metrics** for important patterns (errors, latency buckets)
- Use **Log Analytics** for aggregate SQL queries; use **Logs Explorer** for ad-hoc filtering
- Set appropriate **retention periods** per log bucket (default: 30 days for `_Default`)

### Alerting Strategy

- Set thresholds from historical baseline data
- Create **multi-condition alerts** (CPU AND memory) to reduce false positives
- Use **alert policies with multiple channels** (email + PagerDuty/Slack via webhooks)
- Test alerts regularly using load testing

### SRE Metrics Priority (GCP)

1. **Availability** — Uptime check pass rate
2. **Latency** — Response time from Nginx logs
3. **Error Rate** — HTTP 5xx log-based metric
4. **Saturation** — CPU, memory, disk from Ops Agent

---

## Cleanup Instructions

> ⚠️ **Complete cleanup to avoid charges.**

### Delete Alerting Policies
```
Monitoring → Alerting → Select all policies → Delete
```

### Delete Uptime Checks
```
Monitoring → Uptime checks → Select all → Delete
```

### Delete Dashboards
```
Monitoring → Dashboards → Select SRE Production Dashboard → Delete
```

### Delete Log-Based Metrics
```
Cloud Logging → Log-based Metrics → Select nginx_500_errors → Delete
```

### Delete VM Instances
```
Compute Engine → VM instances → Select sre-workshop-web-01 and web-02
→ Delete
```

### Delete SLOs (if created)
```
Monitoring → SLOs → Select → Delete
```

---

## Additional Resources

| Resource | Link |
|----------|------|
| Cloud Monitoring Documentation | https://cloud.google.com/monitoring/docs |
| Cloud Logging Documentation | https://cloud.google.com/logging/docs |
| Ops Agent Documentation | https://cloud.google.com/stackdriver/docs/solutions/agents/ops-agent |
| Log Analytics (SQL) | https://cloud.google.com/logging/docs/analyze/query-and-view |
| SLO Documentation | https://cloud.google.com/stackdriver/docs/solutions/slo-monitoring |
| Compute Engine Monitoring | https://cloud.google.com/compute/docs/instances/monitor-instances |
| GCP Free Tier | https://cloud.google.com/free |

---

## AWS → GCP Service Mapping Reference

| AWS Service | GCP Equivalent |
|-------------|----------------|
| EC2 | Compute Engine |
| CloudWatch Metrics | Cloud Monitoring |
| CloudWatch Logs | Cloud Logging |
| CloudWatch Agent | Ops Agent |
| CloudWatch Alarms | Alerting Policies |
| CloudWatch Dashboards | Cloud Monitoring Dashboards |
| CloudWatch Logs Insights | Log Analytics (SQL) / Logs Explorer |
| IAM Instance Profile | Service Account (attached to VM) |
| CloudWatchAgentServerPolicy | `roles/monitoring.metricWriter` + `roles/logging.logWriter` |
| SNS Topics | Notification Channels |
| CloudWatch Metric Filters | Log-Based Metrics |
| CloudWatch SLOs | Cloud Monitoring SLOs |

---

*Lab designed as a GCP equivalent of the AWS EC2 + CloudWatch + Logs SRE Workshop. Reduced from 5 hours to 2 hours by using GCP's integrated Ops Agent (single install for metrics + logs), automatic IAM via service accounts, and pre-built log parsers.*
