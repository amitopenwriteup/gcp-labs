# Hands-On Lab: SRE-Based SLO Dashboard on GCP
### EduRamp Learning Services -- Site Reliability Engineering Series
### Console / UI-Based Lab (No CLI Required)

---

## Lab Overview

| Field | Details |
|---|---|
| **Lab Title** | Comprehensive Monitoring & SLO Dashboard for GCP VM Web Services |
| **Duration** | 3-4 Hours |
| **Difficulty** | Intermediate to Advanced |
| **Track** | Site Reliability Engineering (SRE) |
| **Platform** | Google Cloud Platform Console (UI) |
| **Services Used** | Compute Engine, Cloud Monitoring, Cloud Logging, Cloud Trace, Error Reporting, Cloud Profiler |

---

## Learning Objectives

By the end of this lab, you will be able to:

- Launch and configure a **Compute Engine VM** with a running web service using the GCP Console
- Install and configure the **Ops Agent** through the VM console
- Define meaningful **SLIs** and **SLOs** through the Cloud Monitoring UI
- Build a complete **SRE Dashboard** with burn rate widgets using drag-and-drop
- Configure **alert policies** tied to SLO error budget burn rates via the Alerting UI
- Route logs using the **Cloud Logging** Log Router UI
- Create **log-based metrics** and view **Error Reporting** and **Cloud Trace** from the console
- Analyze performance using **Cloud Profiler** from the console

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Architecture Overview](#architecture-overview)
3. [Part 1 - Create the VM via Console](#part-1--create-the-vm-via-console)
4. [Part 2 - Install Ops Agent via Console](#part-2--install-ops-agent-via-console)
5. [Part 3 - Enable APIs via Console](#part-3--enable-apis-via-console)
6. [Part 4 - Deploy the Web Service via SSH Browser](#part-4--deploy-the-web-service-via-ssh-browser)
7. [Part 5 - Create Custom Metrics via Monitoring UI](#part-5--create-custom-metrics-via-monitoring-ui)
8. [Part 6 - Define SLIs and SLOs in Cloud Monitoring UI](#part-6--define-slis-and-slos-in-cloud-monitoring-ui)
9. [Part 7 - Build the SRE SLO Dashboard (UI)](#part-7--build-the-sre-slo-dashboard-ui)
10. [Part 8 - Alert Policies & Notification Channels (UI)](#part-8--alert-policies--notification-channels-ui)
11. [Part 9 - Cloud Logging & Log Router (UI)](#part-9--cloud-logging--log-router-ui)
12. [Part 10 - Log-Based Metrics (UI)](#part-10--log-based-metrics-ui)
13. [Part 11 - Error Reporting & Cloud Trace (UI)](#part-11--error-reporting--cloud-trace-ui)
14. [Part 12 - Cloud Profiler (UI)](#part-12--cloud-profiler-ui)
15. [Part 13 - Verify End-to-End Observability](#part-13--verify-end-to-end-observability)
16. [Cleanup via Console](#cleanup-via-console)
17. [Lab Summary & Key Takeaways](#lab-summary--key-takeaways)

---

## Prerequisites

### What You Need

- A Google Account with access to **Google Cloud Console**
- A GCP Project with **Billing enabled**
- A modern web browser (Chrome recommended)
- No local tools or CLI required everything is done in the browser

### Console Access

Open your browser and go to:
```
https://console.cloud.google.com
```

Make sure your correct **Project** is selected in the top project selector dropdown.

---

## Architecture Overview

```

 GCP PROJECT (Console UI)


 Compute Engine VM Cloud Operations Suite

 Python Flask
 Web Service logs Cloud Monitoring
 port: 8080 - SLO Dashboard
 - Alert Policies
 - Custom Metrics

 Ops Agent
 (metrics + Cloud Logging
 logs) - Log Router
 - Log-based Metrics


 Cloud Trace
 Error Reporting
 Cloud Profiler



```

---

## Part 1 - Create the VM via Console

### Step 1.1 Navigate to Compute Engine

1. In the GCP Console, click the ** Navigation Menu** (top-left hamburger icon)
2. Hover over **Compute Engine**
3. Click **VM Instances**
4. If prompted, click **Enable** to enable the Compute Engine API
5. Wait for the API to enable (takes about 1 minute)

### Step 1.2 Create a New VM Instance

1. Click the **+ CREATE INSTANCE** button at the top of the page
2. Fill in the following fields:

**Basic Configuration:**

| Field | Value |
|---|---|
| **Name** | `sre-lab-vm` |
| **Region** | `us-central1` |
| **Zone** | `us-central1-a` |
| **Machine Family** | General Purpose |
| **Series** | E2 |
| **Machine Type** | `e2-medium` (2 vCPU, 4 GB RAM) |

**Boot Disk:**

3. Under **Boot Disk**, click **CHANGE**
4. Set the following:

| Field | Value |
|---|---|
| **Operating System** | Debian |
| **Version** | Debian GNU/Linux 12 (bookworm) |
| **Boot Disk Type** | Balanced persistent disk |
| **Size** | 20 GB |

5. Click **SELECT**

**Identity and API Access:**

6. Scroll down to **Identity and API access**
7. Under **Access Scopes**, select **Allow full access to all Cloud APIs**

> This grants the VM permission to write metrics, logs, and traces to GCP required for this lab.

**Firewall:**

8. Scroll to the **Firewall** section
9. Check **Allow HTTP traffic**
10. Check **Allow HTTPS traffic**

**Network Tags (for custom firewall rule):**

11. Click **ADVANCED OPTIONS** at the bottom of the form
12. Expand the **Networking** section
13. Under **Network Tags**, type `web-service` and press **Enter**

### Step 1.3 Create Firewall Rule for Port 8080

1. In the Navigation Menu, go to **VPC Network Firewall**
2. Click **+ CREATE FIREWALL RULE**
3. Fill in:

| Field | Value |
|---|---|
| **Name** | `allow-web-service-8080` |
| **Network** | default |
| **Direction of traffic** | Ingress |
| **Action on match** | Allow |
| **Targets** | Specified target tags |
| **Target tags** | `web-service` |
| **Source filter** | IPv4 ranges |
| **Source IPv4 ranges** | `0.0.0.0/0` |
| **Protocols and ports** | Specified protocols and ports TCP `8080` |

4. Click **CREATE**

### Step 1.4 Finish VM Creation

1. Return to **Compute Engine VM Instances**
2. Click **CREATE** at the bottom of the instance creation form
3. Wait for the green checkmark next to `sre-lab-vm` (about 1 minute)

---

## Part 2 - Install Ops Agent via Console

The **Ops Agent** is Google's unified agent for collecting infrastructure metrics and logs from your VM.

### Step 2.1 Open the VM Detail Page

1. Go to **Compute Engine VM Instances**
2. Click on the name **`sre-lab-vm`** to open its detail page
3. Scroll down to the **Observability** section
4. You will see: **"Ops Agent not detected"**

### Step 2.2 Trigger the Installation from the UI

1. Click **INSTALL OPS AGENT** button in the Observability section
2. A panel slides in from the right titled **"Install Ops Agent"**
3. Click the blue **INSTALL** button
4. The console will automatically open an SSH session and run the installation script behind the scenes
5. A progress bar appears wait for the message: **"Ops Agent installed successfully"** (23 minutes)
6. Click **DONE**

> Alternatively, you can reach this from **Cloud Monitoring Settings Agent** and selecting the VM.

### Step 2.3 Verify Ops Agent is Working

1. On the VM detail page, click the **Monitoring** tab (near the top)
2. Live graphs should now appear for:
 - CPU Utilization
 - Memory Usage
 - Disk Read/Write
 - Network traffic

> If these graphs are populated, the Ops Agent is collecting and sending data successfully.

---

## Part 3 - Enable APIs via Console

### Step 3.1 Navigate to API Library

1. In the Navigation Menu, click **APIs & Services Library**

### Step 3.2 Enable Each Required API

Search and enable each API below, one at a time:

| API Name | Search Term | Purpose |
|---|---|---|
| Cloud Monitoring API | `Cloud Monitoring` | Metrics, dashboards, and SLOs |
| Cloud Logging API | `Cloud Logging` | Log collection and routing |
| Cloud Trace API | `Cloud Trace` | Distributed request tracing |
| Error Reporting API | `Error Reporting` | Automatic exception grouping |
| Cloud Profiler API | `Cloud Profiler` | CPU and memory profiling |

**For each API:**

1. Type the search term in the search bar
2. Click the matching API card in the results
3. Click the **ENABLE** button
4. Wait for the green confirmation screen
5. Click the browser Back button and search the next API

---

## Part 4 - Deploy the Web Service via SSH Browser

### Step 4.1 Open the Browser-Based SSH Terminal

1. Go to **Compute Engine VM Instances**
2. In the row for `sre-lab-vm`, click the **SSH** button on the right
3. A new browser window opens with a terminal wait for the connection to establish

### Step 4.2 Install Python and Dependencies

In the SSH terminal, paste and run:

```bash
sudo apt-get update -y && sudo apt-get install -y python3 python3-pip
```

Then install Python libraries:

```bash
pip3 install flask google-cloud-logging google-cloud-monitoring \
 opentelemetry-sdk opentelemetry-exporter-gcp-trace \
 google-cloud-profiler requests --break-system-packages
```

Wait for all packages to finish installing (approximately 2 minutes).

### Step 4.3 Create the Application File

In the terminal, type:

```bash
nano ~/app.py
```

This opens the nano text editor. Paste the complete code below:

```python
import time
import random
import logging
import os
from flask import Flask, jsonify

import google.cloud.logging
from opentelemetry import trace
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Start Cloud Profiler
try:
 import googlecloudprofiler
 googlecloudprofiler.start(
 service='sre-web-service',
 service_version='1.0.0',
 verbose=3
 )
except Exception as e:
 print(f"Profiler not started: {e}")

# Cloud Logging setup
log_client = google.cloud.logging.Client()
log_client.setup_logging()
logger = logging.getLogger("sre-web-service")

# Cloud Trace setup
tracer_provider = TracerProvider()
tracer_provider.add_span_processor(
 BatchSpanProcessor(CloudTraceSpanExporter())
)
trace.set_tracer_provider(tracer_provider)
tracer = trace.get_tracer(__name__)

app = Flask(__name__)

# In-memory counters
request_count = 0
error_count = 0
total_latency = 0.0

@app.route('/health')
def health():
 return jsonify({"status": "ok", "service": "sre-web-service"}), 200

@app.route('/api/data')
def get_data():
 global request_count, error_count, total_latency
 start = time.time()
 request_count += 1

 with tracer.start_as_current_span("handle_get_data"):
 # Simulate 5% error rate for SLO testing
 if random.random() < 0.05:
 error_count += 1
 logger.error("Simulated error on /api/data",
 extra={"json_fields": {
 "endpoint": "/api/data",
 "error_type": "SimulatedInternalError"
 }})
 raise Exception("Simulated Internal Error")

 # Simulate variable latency between 50ms and 400ms
 time.sleep(random.uniform(0.05, 0.4))
 latency = time.time() - start
 total_latency += latency

 logger.info("Request OK", extra={"json_fields": {
 "endpoint": "/api/data",
 "latency_ms": round(latency * 1000, 2),
 "request_number": request_count
 }})
 return jsonify({
 "data": "payload",
 "latency_ms": round(latency * 1000, 2)
 }), 200

@app.route('/metrics')
def metrics():
 avg_latency = (total_latency / request_count * 1000) if request_count > 0 else 0
 error_rate = (error_count / request_count) if request_count > 0 else 0
 return jsonify({
 "total_requests": request_count,
 "total_errors": error_count,
 "error_rate": round(error_rate, 4),
 "avg_latency_ms": round(avg_latency, 2)
 })

@app.errorhandler(Exception)
def handle_exception(e):
 return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
 logger.info("SRE Web Service starting on port 8080")
 app.run(host='0.0.0.0', port=8080, debug=False)
```

Press **Ctrl+X** then **Y** then **Enter** to save and exit.

### Step 4.4 Start the Web Service

```bash
nohup python3 ~/app.py > ~/app.log 2>&1 &
echo "Service started PID: $!"
```

### Step 4.5 Test the Service

```bash
curl http://localhost:8080/health
```

Expected response:
```json
{"service": "sre-web-service", "status": "ok"}
```

### Step 4.6 Start the Traffic Generator

Click **SSH** again to open a **second terminal** for the same VM, then run:

```bash
# Continuous traffic loop keep this running throughout the lab
while true; do
 curl -s http://localhost:8080/api/data > /dev/null
 sleep 2
done
```

> This loop sends a request every 2 seconds, generating live metrics, logs, and traces. Keep this terminal open.

---

## Part 5 - Create Custom Metrics via Monitoring UI

### Step 5.1 Open Metrics Explorer

1. In the Navigation Menu, go to **Monitoring**
2. In the left sidebar, click **Metrics Explorer**

### Step 5.2 Explore Built-In VM Metrics

1. Click the **Select a metric** dropdown
2. In the resource type field, type `gce_instance`
3. Expand **VM Instance CPU**
4. Select `compute.googleapis.com/instance/cpu/utilization`
5. Click **Apply**
6. A live CPU utilization chart for `sre-lab-vm` appears

### Step 5.3 Create Custom Metric Descriptors

1. In the Navigation Menu, go to **Monitoring Metrics Management**
2. Click **+ CREATE METRIC DESCRIPTOR**

**First metric Error Rate:**

| Field | Value |
|---|---|
| **Metric type** | Custom |
| **Metric identifier** | `web_service/error_rate` |
| **Display name** | `Web Service Error Rate` |
| **Description** | `Fraction of requests returning errors` |
| **Unit** | `1` |
| **Metric kind** | Gauge |
| **Value type** | Double |

3. Under **Labels**, click **+ Add Label**:
 - Label key: `endpoint`
 - Value type: String
 - Description: `API endpoint path`
4. Click **CREATE**

**Second metric Average Latency:**

5. Click **+ CREATE METRIC DESCRIPTOR** again

| Field | Value |
|---|---|
| **Metric identifier** | `web_service/avg_latency_ms` |
| **Display name** | `Web Service Average Latency ms` |
| **Metric kind** | Gauge |
| **Value type** | Double |
| **Unit** | `ms` |

6. Add the same `endpoint` label
7. Click **CREATE**

---

## Part 6 - Define SLIs and SLOs in Cloud Monitoring UI

### SRE Concepts Reference

| Term | Definition | This Lab |
|---|---|---|
| **SLI** | Quantitative service behavior measure | % successful requests under 300ms |
| **SLO** | Reliability target over a time window | 99.5% availability over 30 days |
| **Error Budget** | Allowed failures = 100% SLO% | 0.5% = ~216 minutes/month |
| **Burn Rate** | Rate of error budget consumption | 14x = budget gone in ~2 days |

### Step 6.1 Create a Service

1. In the Navigation Menu, go to **Monitoring Services**
2. Click **+ Define Service** (top right)
3. Fill in:

| Field | Value |
|---|---|
| **Service ID** | `sre-web-service` |
| **Display Name** | `SRE Web Service` |
| **Description** | `Production Flask web service on GCE VM` |

4. Under **Telemetry**, select **Custom**
5. Click **SUBMIT**

### Step 6.2 Create the Availability SLO

1. Click on **`SRE Web Service`** in the services list
2. Click the **SLOs** tab
3. Click **+ CREATE SLO**

**Step A Choose SLI Metric Type:**

4. Select **Request-based**
5. Under Performance Metric, choose **Availability**
6. Click **CONTINUE**

**Step B Define Good / Bad Requests:**

7. Under **Good service filter**, enter:
 ```
 metric.type="custom.googleapis.com/web_service/error_rate"
 metric.labels.endpoint="/api/data"
 ```
8. Define a **good request** as: error_rate value **less than 0.01**
9. Click **CONTINUE**

**Step C Set the SLO Target:**

| Field | Value |
|---|---|
| **Performance Goal** | `99.5` % |
| **Compliance Period** | Rolling window |
| **Period Length** | 30 days |

10. Click **CONTINUE**
11. Name the SLO: `Availability SLO 99.5% over 30 days`
12. Click **CREATE SLO**

### Step 6.3 Create the Latency SLO

1. Click **+ CREATE SLO** again
2. Select **Request-based Latency**
3. Set latency threshold: **300 ms**
4. Good requests = those completing within 300ms
5. Click **CONTINUE**

| Field | Value |
|---|---|
| **Performance Goal** | `95` % |
| **Period** | Rolling 30 days |

6. Name: `Latency SLO 95% requests under 300ms`
7. Click **CREATE SLO**

### Step 6.4 Review the Error Budget

1. Click on the **Availability SLO** you created
2. The detail panel shows:
 - **Current SLI value** (live percentage)
 - **Error budget remaining** (in % and in minutes)
 - **Compliance chart** over the 30-day window
3. Note the **SLO ID** from the URL bar you need it for alerts in Part 8

---

## Part 7 - Build the SRE SLO Dashboard (UI)

### Step 7.1 Create a New Dashboard

1. Go to **Monitoring Dashboards**
2. Click **+ CREATE DASHBOARD**
3. In the dialog, enter name: `SRE Dashboard Web Service`
4. Click **CONFIRM**

The canvas opens with an empty grid and a widget toolbar on the left.

### Step 7.2 Widget 1: Error Budget Remaining

1. Click **+ ADD WIDGET**
2. Select **Line Chart**
3. Click **Select a metric**
4. Search for: `error_budget_minutes_remaining`
5. Select: `monitoring.googleapis.com/slo/error_budget_minutes_remaining`
6. In filters, add: `service_name = sre-web-service`
7. Widget title: `Error Budget Remaining (minutes)`
8. Click **APPLY**
9. Drag the widget to span the full top row

### Step 7.3 Widget 2: Availability SLI vs SLO Target

1. Click **+ ADD WIDGET Line Chart**
2. Select metric: `monitoring.googleapis.com/slo/sli_value`
3. Filter: `service_name = sre-web-service`, `slo_name = Availability SLO`
4. Click **Add threshold line**:
 - Value: `0.995`
 - Color: Red
 - Label: `SLO Target (99.5%)`
5. Title: `Availability SLI vs SLO Target`
6. Click **APPLY**

### Step 7.4 Widget 3: Error Rate Gauge

1. Click **+ ADD WIDGET Gauge**
2. Select metric: `custom.googleapis.com/web_service/error_rate`
3. Set thresholds:
 - Green: 0 to 0.003
 - Yellow: 0.003 to 0.005
 - Red: above 0.005
4. Title: `Live Error Rate`
5. Click **APPLY**

### Step 7.5 Widget 4: Latency Over Time

1. Click **+ ADD WIDGET Line Chart**
2. Select metric: `custom.googleapis.com/web_service/avg_latency_ms`
3. Add a threshold:
 - Value: `300`
 - Color: Orange
 - Label: `300ms SLO Threshold`
4. Title: `Average Response Latency (ms)`
5. Click **APPLY**

### Step 7.6 Widget 5: VM CPU Utilization

1. Click **+ ADD WIDGET Line Chart**
2. Select metric: `compute.googleapis.com/instance/cpu/utilization`
3. Filter: `instance_name = sre-lab-vm`
4. Add thresholds:
 - `0.70` Yellow, label `Warning`
 - `0.90` Red, label `Critical`
5. Title: `CPU Utilization sre-lab-vm`
6. Click **APPLY**

### Step 7.7 Widget 6: VM Memory Usage

1. Click **+ ADD WIDGET Line Chart**
2. Select metric: `agent.googleapis.com/memory/percent_used`
3. Filter: `instance_name = sre-lab-vm`
4. Title: `Memory Usage %`
5. Click **APPLY**

### Step 7.8 Widget 7: Request Throughput

1. Click **+ ADD WIDGET Line Chart**
2. Select metric: `custom.googleapis.com/web_service/request_count`
3. Set aligner to: **ALIGN_RATE** (shows requests per second)
4. Title: `Request Rate (req/s)`
5. Click **APPLY**

### Step 7.9 Widget 8: SLO Compliance Over 30 Days

1. Click **+ ADD WIDGET Line Chart**
2. Select metric: `monitoring.googleapis.com/slo/sli_value`
3. Group by: `slo_name`
4. This overlays both the Availability and Latency SLIs on the same chart
5. Title: `SLI Compliance All SLOs`
6. Click **APPLY**

### Step 7.10 Arrange and Save the Dashboard

Arrange widgets in a logical layout:

```

 Error Budget Remaining (full width)

 Availability SLI Error Rate Gauge Burn Rate

 Latency (ms) Request Rate (req/s)

 CPU Utilization Memory Usage %
-
```

Click **SAVE** in the top-right corner.

> Your SRE Dashboard is now live. It auto-refreshes every 1 minute by default change the interval in the top-right clock icon.

---

## Part 8 - Alert Policies & Notification Channels (UI)

### Step 8.1 Create a Notification Channel

1. Go to **Monitoring Alerting** in the left sidebar
2. Click **EDIT NOTIFICATION CHANNELS** (link at the top of the page)
3. In the Email section, click **+ ADD NEW**
4. Fill in:

| Field | Value |
|---|---|
| **Display Name** | `SRE On-Call Team` |
| **Email Address** | `your-email@domain.com` |

5. Click **SAVE**

> You can also add Slack, PagerDuty, SMS, or custom webhook channels from the same screen.

### Step 8.2 Create the Fast Burn Rate Alert (P1)

This alert fires when the error budget is burning at **14x the normal rate**, meaning the entire 30-day budget could be exhausted within about 2 days.

1. Go to **Monitoring Alerting**
2. Click **+ CREATE POLICY**

**Step A Add a Condition:**

3. Click **SELECT A METRIC**
4. Search for: `error_budget_minutes_remaining`
5. Select: `monitoring.googleapis.com/slo/error_budget_minutes_remaining`
6. Filter:
 - `service_name = sre-web-service`
 - `slo_name = Availability SLO`
7. Click **APPLY**

**Step B Configure the Trigger:**

8. Under **Transform data**:
 - Rolling window: `1 hour`
 - Rolling window function: `mean`
9. Under **Configure alert trigger**:
 - Condition type: **Threshold**
 - Alert fires when: **value is below**
 - Threshold value: `211`

 > Why 211? Total budget for 99.5% SLO over 30 days = 0.5% 43,200 min = 216 min. A 14x burn rate in 1 hour consumes 14 (216/720 hrs) = ~4.2 min. Alert fires when remaining budget drops rapidly toward this threshold.

10. Condition name: `Fast Burn Rate 14x (1-hour window)`
11. Click **NEXT**

**Step C Add Notifications:**

12. Under **Notify on**, keep **"An incident is opened"** selected
13. Click **ADD NOTIFICATION CHANNELS**
14. Select **SRE On-Call Team** and click **OK**

**Step D Add Runbook Documentation:**

15. In the **Documentation** box, paste:

```
## Fast Burn Rate Alert Immediate Action Required

The SLO error budget is burning at 14x normal rate.
At this pace, the entire 30-day budget will be exhausted in approximately 2 days.

STEPS TO INVESTIGATE:
1. Open the SRE Dashboard check current error rate and latency
2. Go to Cloud Logging filter severity=ERROR for the last 30 minutes
3. Check Error Reporting for any new error groups
4. Review Cloud Trace for high-latency or failing requests
5. Check VM CPU and Memory on the dashboard

ESCALATION:
If not resolved within 30 minutes, escalate to the service owner.
```

16. Set **Alert name**: `SLO Fast Burn Rate P1 - Critical`
17. Click **NEXT REVIEW SAVE POLICY**

### Step 8.3 Create the Slow Burn Rate Alert (P2)

1. Click **+ CREATE POLICY** again
2. Same metric: `monitoring.googleapis.com/slo/error_budget_minutes_remaining`
3. Filter to the same SLO

**Condition settings:**

| Field | Value |
|---|---|
| **Rolling window** | 6 hours |
| **Condition** | Value is below threshold |
| **Threshold** | `195` (corresponds to ~2x burn over 6 hours) |
| **Condition name** | `Slow Burn Rate 2x (6-hour window)` |

4. Add notification: **SRE On-Call Team**
5. Add documentation:

```
## Slow Burn Rate Alert Investigate

The error budget is eroding at 2x the normal rate over the last 6 hours.
This will exhaust the budget in approximately 15 days if unchecked.

STEPS:
1. Review recent deployments for regressions
2. Check latency trends in Metrics Explorer
3. Review log-based metrics for error trends
4. If budget drops below 50%, consider throttling traffic
```

6. Alert name: `SLO Slow Burn Rate P2 - High`
7. Click **SAVE POLICY**

### Step 8.4 Burn Rate Alert Reference Table

| Alert Level | Burn Rate | Window | Budget Consumed | Response |
|---|---|---|---|---|
| Critical | 14x | 1 hour | 2% in 1 hr | Page On-Call immediately |
| High | 6x | 6 hours | 5% in 6 hrs | Create ticket + escalate |
| Medium | 3x | 3 days | 10% in 3 days | Investigate proactively |
| Normal | 1x | 30 days | Budget at 0% at window end | No action needed |

---

## Part 9 - Cloud Logging & Log Router (UI)

### Step 9.1 Open Log Explorer

1. In the Navigation Menu, go to **Logging Log Explorer**
2. In the query editor, enter:
 ```
 resource.type="gce_instance"
 ```
3. Click **RUN QUERY**
4. Log entries from your VM appear in real time

### Step 9.2 Filter to Application Logs

1. Update the query to:
 ```
 resource.type="gce_instance"
 jsonPayload.json_fields.endpoint="/api/data"
 ```
2. Click **RUN QUERY**
3. You should see structured JSON log entries with `latency_ms` and `request_number` fields
4. Expand any log entry by clicking the **** arrow to inspect the full JSON payload

### Step 9.3 View Error Logs Only

1. Change the query to:
 ```
 resource.type="gce_instance"
 severity=ERROR
 ```
2. Click **RUN QUERY**
3. These are the error log entries from the 5% error simulation in the application

### Step 9.4 Create a Log Sink to BigQuery

A **Log Sink** copies matching logs to another GCP destination for long-term analysis.

1. In the Navigation Menu, go to **Logging Log Router**
2. Click **+ CREATE SINK**

**Sink Details:**

| Field | Value |
|---|---|
| **Sink name** | `sre-error-logs-sink` |
| **Sink description** | Routes error logs to BigQuery for trend analysis |

3. Click **NEXT**

**Sink Destination:**

4. Under **Select sink service**, choose **BigQuery dataset**
5. Click **CREATE NEW BIGQUERY DATASET**
6. Fill in:
 - Dataset ID: `sre_logs_analysis`
 - Location: `US`
7. Click **CREATE DATASET**
8. Back in the sink form, select the newly created dataset

**Inclusion Filter:**

9. In the **Build inclusion filter** text box, enter:
 ```
 resource.type="gce_instance"
 severity>=ERROR
 ```
10. Click **CREATE SINK**

> All ERROR and CRITICAL logs from `sre-lab-vm` will now be automatically routed to BigQuery.

---

## Part 10 - Log-Based Metrics (UI)

Log-based metrics convert log data into time-series metrics that can be charted and alerted on.

### Step 10.1 Create an Error Counter Metric

1. Go to **Logging Log-based Metrics**
2. Click **+ CREATE METRIC**
3. Select **Counter** as the metric type
4. Fill in:

| Field | Value |
|---|---|
| **Metric name** | `web_service_errors_total` |
| **Description** | `Counts ERROR log entries from the web service` |
| **Unit** | `1` |

**Filter to Build the Metric From:**

5. In the **Build filter** section, enter:
 ```
 resource.type="gce_instance"
 severity=ERROR
 jsonPayload.json_fields.error_type="SimulatedInternalError"
 ```
6. Click **VERIFY PREVIEW** you should see matching log entries in the preview panel below
7. Click **CREATE METRIC**

### Step 10.2 Create a Latency Distribution Metric

1. Click **+ CREATE METRIC**
2. Select **Distribution** as the metric type
3. Fill in:

| Field | Value |
|---|---|
| **Metric name** | `web_service_latency_dist` |
| **Description** | `Latency distribution derived from application logs` |
| **Unit** | `ms` |

**Field Extraction:**

4. Under **Field name**, enter: `jsonPayload.json_fields.latency_ms`
5. Set Type: **Double**

**Histogram Buckets:**

6. Under **Histogram buckets**, select **Explicit**
7. In the **Bounds** field, enter (comma separated): `50, 100, 150, 200, 250, 300, 400, 500, 1000`
8. Click **CREATE METRIC**

### Step 10.3 Add Log-Based Metrics to the SRE Dashboard

1. Go to **Monitoring Dashboards SRE Dashboard Web Service**
2. Click **+ ADD WIDGET Line Chart**
3. In the metric picker, search for: `logging.googleapis.com/user/web_service_errors_total`
4. Select it and click **APPLY**
5. Title: `Error Log Count (Log-Based)`

6. Add one more widget:
 - Metric: `logging.googleapis.com/user/web_service_latency_dist`
 - Aggregation: **percentile(99)**
 - Title: `P99 Latency from Logs (ms)`
7. Click **SAVE**

---

## Part 11 - Error Reporting & Cloud Trace (UI)

### Step 11.1 Explore Error Reporting

1. In the Navigation Menu, go to **Error Reporting**
2. The console automatically groups identical exceptions into error groups
3. You should see: **`Exception: Simulated Internal Error`** this is the intentional 5% error from the app

**Click the error group to explore:**

| Section | What to Examine |
|---|---|
| **Occurrences chart** | Rate of errors over time look for spikes |
| **First seen / Last seen** | When the error was introduced and if it is still active |
| **Stack trace** | Full Python call stack showing where the error was raised |
| **Affected versions** | Which service version is associated with this error |
| **Sample** | A single raw log entry for one occurrence |

4. Click **View logs** this jumps directly to Log Explorer filtered to this error group

### Step 11.2 Configure Error Reporting Email Notifications

1. In Error Reporting, click the ** Settings** icon (top-right area)
2. Click **EMAIL NOTIFICATIONS**
3. Add your email address
4. Set notification frequency: **Immediately for new errors**
5. Click **SAVE**

> You will now be notified automatically any time a previously unseen error type occurs.

### Step 11.3 Explore Cloud Trace

1. In the Navigation Menu, go to **Trace Trace List**
2. Set the time range selector (top-right) to **Last 30 minutes**
3. A list of individual traces appears one per request to `/api/data`

**Open a single trace:**

4. Click any trace in the list
5. The **Waterfall view** opens showing:
 - `handle_get_data` the span created by your route handler
 - Duration bar in milliseconds
 - Start time relative to the request

6. In the right panel, inspect **Labels** which include HTTP method, URL, status code, and project ID

**Sort by Slowest Traces:**

7. Back in the Trace List, click the **Latency** column header to sort descending
8. The slowest requests appear at the top any over 300ms would represent latency SLO violations
9. Click one of those slow traces to identify which part of the code caused the delay

**View Latency Distribution:**

10. Go to **Trace Overview**
11. The **Latency Distribution** histogram shows:
 - X-axis: Response time buckets (ms)
 - Y-axis: Percentage of total requests
12. Identify P50 (median), P95, and P99 latency percentiles visually

### Step 11.4 Create a Latency Analysis Report

1. In Cloud Trace, click **Analysis Reports** in the left panel
2. Click **+ NEW REPORT**

| Field | Value |
|---|---|
| **Report name** | `Web Service Latency Analysis` |
| **Service** | `sre-web-service` |
| **Time range** | Last 24 hours |

3. Click **CREATE REPORT**
4. After processing (12 minutes), the report shows latency percentile trends, comparisons between time periods, and regression indicators

---

## Part 12 - Cloud Profiler (UI)

Cloud Profiler collects continuous CPU, wall-clock, heap, and thread-contention profiles from your running service.

### Step 12.1 Open Cloud Profiler

1. In the Navigation Menu, go to **Profiler**
2. In the filter bar at the top, set:
 - **Service**: `sre-web-service`
 - **Version**: `1.0.0`
 - **Profile type**: `CPU time`
3. Allow **510 minutes** of traffic before profile data appears

### Step 12.2 Read the Flame Graph

The flame graph displays the call stack of your application with width proportional to CPU time:

```
Wider bar = More CPU time consumed by that function
Taller stack = Deeper nested function calls


 handle_get_data Entry point (widest = most time)

 sleep() jsonify() logger.info Called functions

 time.time() dict ops encode() Deeper call stack

```

- Click any bar in the flame graph to **zoom in** to that function
- Click the root bar to **zoom back out**

### Step 12.3 Switch Profile Types

Use the **Profile type** dropdown to explore different views:

| Profile Type | What It Reveals |
|---|---|
| **CPU time** | Where the CPU is actively computing |
| **Wall time** | Actual elapsed time including I/O and sleep waits |
| **Heap** | Current memory objects held in RAM |
| **Allocated heap** | Total memory allocated over time (including freed) |
| **Contention** | Thread locking and blocking delays |

> For this Flask app with `time.sleep()`, **Wall time** is most informative as it shows where the request is actually waiting.

### Step 12.4 Filter the Flame Graph

1. In the search box above the flame graph, type: `handle_get_data`
2. The view highlights only frames in the call path of your route handler
3. This helps isolate expensive sub-function calls within your own code

---

## Part 13 - Verify End-to-End Observability

### Complete Verification Checklist

Go through each item and confirm the expected status:

| Component | Where to Check in Console | Expected Status |
|---|---|---|
| VM is running | Compute Engine VM Instances | Green status |
| Web service responding | SSH terminal: `curl localhost:8080/health` | `{"status":"ok"}` |
| Ops Agent active | VM detail Monitoring tab | Live metric graphs |
| Custom metrics visible | Monitoring Metrics Explorer type `web_service` | Data points present |
| Availability SLO created | Monitoring Services SLOs tab | SLI value shown live |
| Latency SLO created | Same location | SLI value shown live |
| Dashboard fully populated | Monitoring Dashboards SRE Dashboard | All 8+ widgets show data |
| Fast burn alert active | Monitoring Alerting Policies | Policy listed, no incidents |
| Slow burn alert active | Same location | Policy listed |
| Application logs visible | Logging Log Explorer | JSON log entries streaming |
| Error logs visible | Log Explorer with `severity=ERROR` | Periodic error entries |
| Log sink active | Logging Log Router | Sink shows as active |
| Error counter metric | Logging Log-based Metrics | `web_service_errors_total` listed |
| Latency distribution metric | Same location | `web_service_latency_dist` listed |
| Error group visible | Error Reporting | `SimulatedInternalError` group |
| Traces visible | Trace Trace List | Entries with ms latency |
| Latency distribution | Trace Overview | Histogram with P50/P95/P99 |
| Profiler flame graph | Profiler | CPU/Wall time graph after 10 min |

### Step 13.1 Simulate a High Error Burst

In the SSH terminal (first window), run a burst of requests:

```bash
for i in $(seq 1 200); do
 curl -s http://localhost:8080/api/data > /dev/null
done
```

Then go to **Monitoring Services SRE Web Service Availability SLO** and watch the error budget decrease in near-real-time over the next few minutes.

### Step 13.2 Watch the Alert Fire

1. Go to **Monitoring Alerting**
2. If the burst was large enough to trigger the burn rate condition, an **Active Incident** appears in red
3. Click the incident to view:
 - Incident start time and duration
 - Affected resource (your VM and service)
 - Linked documentation (the runbook you wrote)
4. Check your email for the notification from **SRE On-Call Team**

### Step 13.3 Acknowledge and Close the Incident

1. In the active incident view, click **ACKNOWLEDGE**
2. Add an acknowledgment note: `Confirmed simulated burst no real production impact`
3. Once the error rate returns to normal, click **CLOSE INCIDENT**

---

## Cleanup via Console

> Complete all cleanup steps to avoid ongoing charges after the lab.

### Step 1 Delete Alert Policies

1. Go to **Monitoring Alerting Policies**
2. Select both policies (checkboxes on the left)
3. Click **DELETE** in the top bar
4. Confirm deletion

### Step 2 Delete SLOs

1. Go to **Monitoring Services SRE Web Service SLOs tab**
2. Click the **** menu next to each SLO **Delete**
3. Confirm deletion of both SLOs

### Step 3 Delete the Service

1. Still in **Monitoring Services**
2. Click **** next to `SRE Web Service`
3. Click **Delete**

### Step 4 Delete the Dashboard

1. Go to **Monitoring Dashboards**
2. Click the **** menu next to `SRE Dashboard Web Service`
3. Click **Delete**

### Step 5 Delete Log-Based Metrics

1. Go to **Logging Log-based Metrics**
2. Check both metrics
3. Click **DELETE**

### Step 6 Delete the Log Sink

1. Go to **Logging Log Router**
2. Click **** next to `sre-error-logs-sink`
3. Click **Delete sink** and confirm

### Step 7 Delete the BigQuery Dataset

1. Go to **BigQuery Explorer** (in Navigation Menu)
2. Expand your project
3. Right-click on `sre_logs_analysis`
4. Click **Delete dataset**
5. Type the dataset name to confirm, then click **DELETE**

### Step 8 Delete the Firewall Rule

1. Go to **VPC Network Firewall**
2. Check `allow-web-service-8080`
3. Click **DELETE** in the top toolbar

### Step 9 Delete the VM

1. Go to **Compute Engine VM Instances**
2. Check the box next to `sre-lab-vm`
3. Click **DELETE** in the toolbar
4. In the dialog, confirm by clicking **DELETE** again

> Deleting the VM also deletes all data on its boot disk. Make sure you have saved any important outputs first.

---

## Lab Summary & Key Takeaways

### What You Built

```

 SRE Observability Stack

 GCE VM Ops Agent Cloud Monitoring
 Flask App

 SRE SLO Dashboard
 - Error Budget Widget
 - SLI Compliance Chart
 - Latency / Error Rate
 - CPU / Memory


 Alerts: Fast Burn (14x P1)
 Slow Burn (2x P2)

 Logs Log Router BQ
 Errors Error Reporting
 Traces Cloud Trace
 CPU/Heap Cloud Profiler

```

### Core SRE Principles Applied

**1. The Four Golden Signals**

Every monitoring widget you built maps to one of the four golden signals that Google SRE recommends tracking for every service:

| Signal | How We Measured It |
|---|---|
| **Latency** | Latency SLO + avg_latency_ms metric + P99 from Cloud Trace |
| **Traffic** | Request Rate widget on dashboard |
| **Errors** | Availability SLO + error_rate metric + Error Reporting groups |
| **Saturation** | CPU Utilization and Memory Usage dashboard widgets |

**2. Why Burn Rate Alerting is Better**

Traditional threshold alerts (e.g., "fire if error rate > 5%") create two problems: they miss slow degradations that erode the budget quietly, and they fire too noisily on brief spikes. Burn rate alerting tells you *how fast* your budget is depleting, giving you proportionate urgency.

**3. Structured Logging Powers Everything Downstream**

By writing application logs in structured JSON format with consistent field names, you were able to build log-based metrics, rich query filters, Error Reporting groupings, and BigQuery analysis all without changing instrumentation later.

**4. The Ops Agent is the Observability Foundation**

One installation of the Ops Agent automatically collects dozens of VM-level metrics (CPU, memory, disk, network) that back the infrastructure widgets on your dashboard no custom SDK integration needed for infrastructure observability.

**5. Cloud Trace + Error Reporting + Profiler = Full APM**

- **Error Reporting** groups exceptions and alerts on new error types automatically
- **Cloud Trace** shows the latency breakdown of individual requests end-to-end
- **Cloud Profiler** reveals CPU hotspots and memory leaks in application code

---

## Further Reading & Resources

| Resource | URL |
|---|---|
| Google SRE Book (free online) | https://sre.google/sre-book/table-of-contents/ |
| Cloud Monitoring SLO Documentation | https://cloud.google.com/monitoring/slos |
| Alerting on SLOs SRE Workbook | https://sre.google/workbook/alerting-on-slos/ |
| Ops Agent Overview | https://cloud.google.com/monitoring/agent/ops-agent |
| Log-Based Metrics Guide | https://cloud.google.com/logging/docs/logs-based-metrics |
| Cloud Trace Documentation | https://cloud.google.com/trace/docs |
| Error Reporting Documentation | https://cloud.google.com/error-reporting/docs |
| Cloud Profiler Documentation | https://cloud.google.com/profiler/docs |

---

*EduRamp Learning Services SRE Track | Lab Version 2.0 (UI Edition) | Google Cloud Platform*
