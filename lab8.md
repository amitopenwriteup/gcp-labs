# GCP Load Balancing Lab Guide
### ALB · NLB · VM-Based Nginx App (UI Only)

> **Prerequisites:** Active GCP project with billing enabled, Owner/Editor IAM role.
> All steps use **Google Cloud Console (UI only)** — no `gcloud` CLI until the final section.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Lab 1 — Application Load Balancer (ALB) for APIs](#lab-1--application-load-balancer-alb-for-apis)
3. [Lab 2 — Network Load Balancer (NLB)](#lab-2--network-load-balancer-nlb)
4. [Lab 3 — ALB for VM-Based Nginx App](#lab-3--alb-for-vm-based-nginx-app)
5. [Essential CLI Commands (Reference)](#essential-cli-commands-reference)
6. [Cleanup Steps](#cleanup-steps)

---

## Architecture Overview

```
Internet Traffic
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│               Cloud Load Balancer (Frontend)            │
│          Forwarding Rule → Target Proxy → URL Map       │
└───────────────────────┬─────────────────────────────────┘
                        │
          ┌─────────────┴─────────────┐
          ▼                           ▼
  ┌───────────────┐         ┌───────────────────┐
  │  ALB (L7)     │         │  NLB (L4)         │
  │  HTTP/HTTPS   │         │  TCP/UDP           │
  │  URL routing  │         │  IP:Port routing   │
  └───────┬───────┘         └────────┬──────────┘
          │                          │
          ▼                          ▼
  ┌───────────────┐         ┌────────────────────┐
  │ Backend Svc   │         │  Backend Svc        │
  │ (Instance Grp)│         │  (Instance Grp)     │
  └───────────────┘         └────────────────────┘
```

| Feature            | ALB (L7)                  | NLB (L4)              |
|--------------------|---------------------------|-----------------------|
| OSI Layer          | Layer 7 (Application)     | Layer 4 (Transport)   |
| Protocol           | HTTP / HTTPS              | TCP / UDP / SSL       |
| Routing Logic      | URL path, headers, host   | IP address & port     |
| Use Case           | REST APIs, web apps       | High-throughput, low-latency |
| SSL Termination    | Yes (at LB)               | Yes (passthrough/proxy)|
| Health Checks      | HTTP/HTTPS                | TCP / HTTP            |

---

## Lab 1 — Application Load Balancer (ALB) for APIs

### Step 1.1 — Create a VPC Network

1. Go to **VPC Network → VPC Networks**
2. Click **Create VPC Network**
3. Fill in:
   - **Name:** `api-lb-vpc`
   - **Subnet creation mode:** Custom
   - Click **Add Subnet**
     - **Name:** `api-subnet`
     - **Region:** `us-central1`
     - **IP range:** `10.10.0.0/24`
4. Click **Create**

---

### Step 1.2 — Create Instance Template (API Backend)

1. Go to **Compute Engine → Instance Templates**
2. Click **Create Instance Template**
3. Configure:
   - **Name:** `api-instance-template`
   - **Machine type:** `e2-micro`
   - **Boot disk:** Debian GNU/Linux 11
4. Expand **Advanced Options → Management**
5. Paste in **Startup script:**

```bash
#!/bin/bash
sudo apt-get update -y
sudo apt-get install -y python3
cat > /tmp/api_server.py << 'EOF'
from http.server import HTTPServer, BaseHTTPRequestHandler
import json, socket

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'application/json')
        self.end_headers()
        response = {
            "message": "Hello from GCP ALB API",
            "host": socket.gethostname(),
            "path": self.path
        }
        self.wfile.write(json.dumps(response).encode())

HTTPServer(('0.0.0.0', 8080), Handler).serve_forever()
EOF
python3 /tmp/api_server.py &
```

6. Under **Networking**, add network tag: `api-backend`
7. Click **Create**

---

### Step 1.3 — Create Managed Instance Group (MIG)

1. Go to **Compute Engine → Instance Groups**
2. Click **Create Instance Group**
3. Select **Managed instance group (stateless)**
4. Configure:
   - **Name:** `api-instance-group`
   - **Instance template:** `api-instance-template`
   - **Location:** Single zone → `us-central1-a`
   - **Number of instances:** `2`
   - **Autoscaling:** On
     - Min instances: `2`, Max: `4`
     - CPU target: `60%`
5. Click **Create**

---

### Step 1.4 — Create Firewall Rules

1. Go to **VPC Network → Firewall**
2. Click **Create Firewall Rule**

**Rule 1 — Allow health checks:**
- **Name:** `allow-health-checks`
- **Network:** `api-lb-vpc`
- **Targets:** Specified target tags → `api-backend`
- **Source IP ranges:** `130.211.0.0/22`, `35.191.0.0/16`
- **Protocols/ports:** TCP → `8080`
- Click **Create**

**Rule 2 — Allow HTTP traffic:**
- **Name:** `allow-http-api`
- **Targets:** Specified target tags → `api-backend`
- **Source IP ranges:** `0.0.0.0/0`
- **Protocols/ports:** TCP → `8080`
- Click **Create**

---

### Step 1.5 — Create Backend Service

1. Go to **Network Services → Load Balancing**
2. Click **Create Load Balancer**
3. Select **Application Load Balancer (HTTP/HTTPS)** → **Start Configuration**
4. Select:
   - **Internet facing or internal:** From Internet to my VMs
   - **Global or Regional:** Global (Classic)
5. **Name:** `api-alb`

**Configure Backend:**
6. Click **Backend configuration → Create a backend service**
   - **Name:** `api-backend-service`
   - **Backend type:** Instance group
   - Click **Add Backend:**
     - Instance group: `api-instance-group`
     - Port numbers: `8080`
     - Balancing mode: `Utilization`
   - **Health Check:** Create health check
     - Name: `api-health-check`
     - Protocol: `HTTP`
     - Port: `8080`
     - Request path: `/`
   - Click **Create**

---

### Step 1.6 — Configure URL Map (Routing Rules)

7. Click **Host and path rules**
   - **Mode:** Advanced host and path rule
   - Add rule:
     - **Hosts:** `*`
     - **Paths:** `/api/*` → Backend: `api-backend-service`
     - **Default:** `api-backend-service`
8. Click **Done**

---

### Step 1.7 — Configure Frontend

9. Click **Frontend configuration**
   - **Name:** `api-frontend`
   - **Protocol:** HTTP
   - **IP version:** IPv4
   - **IP address:** Create IP address → Name: `api-lb-ip`
   - **Port:** `80`
10. Click **Done → Review and Finalize → Create**

> ⏳ Wait 3–5 minutes for the load balancer to provision.

**Test the ALB:**
- Navigate to **Network Services → Load Balancing**
- Copy the frontend IP and open in browser: `http://<FRONTEND_IP>/api/health`

---

## Lab 2 — Network Load Balancer (NLB)

### Step 2.1 — Create Instance Template (TCP Backend)

1. Go to **Compute Engine → Instance Templates**
2. Click **Create Instance Template**
3. Configure:
   - **Name:** `nlb-instance-template`
   - **Machine type:** `e2-micro`
   - **Boot disk:** Debian GNU/Linux 11
4. **Startup script:**

```bash
#!/bin/bash
sudo apt-get update -y
sudo apt-get install -y netcat-openbsd
while true; do
  echo -e "HTTP/1.1 200 OK\r\nContent-Length: 28\r\n\r\nHello from NLB Backend!" | nc -l -p 80 -q 1
done &
```

5. Add network tag: `nlb-backend`
6. Click **Create**

---

### Step 2.2 — Create Managed Instance Group for NLB

1. Go to **Compute Engine → Instance Groups → Create Instance Group**
2. Configure:
   - **Name:** `nlb-instance-group`
   - **Template:** `nlb-instance-template`
   - **Location:** Single zone → `us-central1-a`
   - **Instances:** `1`
3. Click **Create**

---

### Step 2.3 — Create Firewall Rule for NLB

1. **VPC Network → Firewall → Create Firewall Rule**
   - **Name:** `allow-nlb-traffic`
   - **Targets:** Tag → `nlb-backend`
   - **Source IP ranges:** `0.0.0.0/0`
   - **Protocols/ports:** TCP → `80`
2. Click **Create**

---

### Step 2.4 — Create Network Load Balancer

1. **Network Services → Load Balancing → Create Load Balancer**
2. Select **Network Load Balancer (TCP/UDP/SSL)** → **Start Configuration**
3. Select:
   - **Internet facing or internal:** From Internet
   - **Region:** `us-central1`
4. **Name:** `api-nlb`

**Backend Configuration:**
5. Click **Backend configuration**
   - **Backend type:** Target pool
   - **Name:** `nlb-target-pool`
   - **Region:** `us-central1`
   - **Health check:** Create health check
     - Name: `nlb-health-check`
     - Protocol: `TCP`
     - Port: `80`
   - **Instances:** Add all instances from `nlb-instance-group`
6. Click **Done**

**Frontend Configuration:**
7. Click **Frontend configuration**
   - **Name:** `nlb-frontend`
   - **Protocol:** `TCP`
   - **IP:** Create static IP → `nlb-lb-ip`
   - **Port:** `80`
8. Click **Done → Review → Create**

---

## Lab 3 — ALB for VM-Based Nginx App

> This lab sets up a production-style ALB in front of Nginx VMs — the most common real-world pattern.

### Step 3.1 — Create Nginx Instance Template

1. **Compute Engine → Instance Templates → Create Instance Template**
2. Configure:
   - **Name:** `nginx-instance-template`
   - **Machine type:** `e2-small`
   - **Boot disk:** Debian GNU/Linux 11 (20 GB)
3. **Startup script:**

```bash
#!/bin/bash
apt-get update -y
apt-get install -y nginx

# Custom index page showing hostname
cat > /var/www/html/index.html << EOF
<!DOCTYPE html>
<html>
<head><title>GCP ALB - Nginx</title></head>
<body style="font-family:sans-serif; padding:40px; background:#f0f4f8;">
  <h1>&#x2705; Nginx Backend Server</h1>
  <p><strong>Hostname:</strong> $(hostname)</p>
  <p><strong>Zone:</strong> $(curl -s http://metadata.google.internal/computeMetadata/v1/instance/zone -H 'Metadata-Flavor: Google' | cut -d/ -f4)</p>
  <p>Served via <strong>GCP Application Load Balancer</strong></p>
</body>
</html>
EOF

# Health check endpoint
mkdir -p /var/www/html/health
echo "OK" > /var/www/html/health/index.html

# Nginx config
cat > /etc/nginx/sites-available/default << 'NGINX'
server {
    listen 80;
    server_name _;

    location /health {
        alias /var/www/html/health/;
        access_log off;
    }

    location / {
        root /var/www/html;
        index index.html;
    }
}
NGINX

systemctl restart nginx
systemctl enable nginx
```

4. **Networking tab:**
   - Network: `default` (or your VPC)
   - Network tag: `nginx-lb-backend`
5. Click **Create**

---

### Step 3.2 — Create Regional Managed Instance Groups (Multi-Zone)

**Create MIG — Zone A:**
1. **Compute Engine → Instance Groups → Create Instance Group**
   - **Name:** `nginx-mig-us-central1-a`
   - **Template:** `nginx-instance-template`
   - **Location:** Single zone → `us-central1-a`
   - **Instances:** `2`
   - **Autoscaling:** On (Min: 2, Max: 5, CPU: 70%)
2. Click **Create**

**Create MIG — Zone B (repeat for HA):**
1. Repeat above with:
   - **Name:** `nginx-mig-us-central1-b`
   - **Zone:** `us-central1-b`
   - **Instances:** `2`

---

### Step 3.3 — Firewall Rules for Nginx

1. **VPC Network → Firewall → Create Firewall Rule**

**Rule 1 — Google health check sources:**
- **Name:** `nginx-allow-health-check`
- **Targets:** Tag → `nginx-lb-backend`
- **Source IP ranges:** `130.211.0.0/22`, `35.191.0.0/16`
- **Protocol/Port:** TCP → `80`

**Rule 2 — HTTP from internet:**
- **Name:** `nginx-allow-http`
- **Targets:** Tag → `nginx-lb-backend`
- **Source IP ranges:** `0.0.0.0/0`
- **Protocol/Port:** TCP → `80`

---

### Step 3.4 — Reserve a Static External IP

1. **VPC Network → External IP Addresses → Reserve Static Address**
   - **Name:** `nginx-alb-ip`
   - **Type:** Global
   - **IP version:** IPv4
2. Note the IP address for DNS or testing

---

### Step 3.5 — Create the Application Load Balancer (Nginx)

1. **Network Services → Load Balancing → Create Load Balancer**
2. Select **Application Load Balancer (HTTP/HTTPS)** → **Start Configuration**
3. Configure:
   - **Internet facing:** Yes
   - **Global:** Yes (Classic)
4. **Name:** `nginx-alb`

**Backend Service:**
5. Click **Backend configuration → Create a backend service**
   - **Name:** `nginx-backend-service`
   - **Protocol:** HTTP
   - **Timeout:** 30s
   - Click **Add Backend:**
     - Instance group: `nginx-mig-us-central1-a` | Port: `80` | Balancing: Utilization
   - Click **Add Backend** again:
     - Instance group: `nginx-mig-us-central1-b` | Port: `80` | Balancing: Utilization
   - **Enable Cloud CDN:** ✅ (optional but recommended)
   - **Health check:** Create new
     - **Name:** `nginx-health-check`
     - **Protocol:** HTTP
     - **Port:** `80`
     - **Request path:** `/health`
     - **Check interval:** 10s | **Unhealthy threshold:** 3
6. Click **Create**

**URL Map / Host & Path Rules:**
7. Click **Host and path rules**
   - Leave default → all traffic → `nginx-backend-service`

**HTTPS / SSL (Optional but Recommended):**
8. Click **Frontend configuration**
   - **Protocol:** HTTPS
   - **IP address:** `nginx-alb-ip` (reserved earlier)
   - **Certificate:** Create Google-managed certificate
     - Enter your domain name
   - **Port:** `443`
9. Add another frontend for HTTP → HTTPS redirect:
   - **Protocol:** HTTP | **Port:** `80`
   - Enable **Redirect to HTTPS**

10. Click **Done → Review and Finalize → Create**

---

### Step 3.6 — Verify the Setup

| Check | Where to Look |
|-------|---------------|
| LB Status | Network Services → Load Balancing → Status: ✅ |
| Backend Health | Click LB → Backend → Health: Healthy |
| Instance Count | Compute Engine → Instance Groups |
| Access Test | Open `http://<nginx-alb-ip>` in browser |
| Refresh Check | Refresh page — hostname should rotate between backends |

---

### Step 3.7 — Configure Autoscaling Policy (UI)

1. **Compute Engine → Instance Groups → `nginx-mig-us-central1-a`**
2. Click **Edit**
3. Under **Autoscaling:**
   - **Autoscaling mode:** On
   - **Min instances:** `2`
   - **Max instances:** `6`
   - **Autoscaling signal:** CPU utilization → `70%`
   - Click **Add signal** → HTTP load balancing utilization → `0.8`
4. Click **Save**

---

## Essential CLI Commands (Reference)

> Use these for scripting, automation, or verification from Cloud Shell.

```bash
# ── AUTH & PROJECT ────────────────────────────────────────────
gcloud auth login
gcloud config set project PROJECT_ID
gcloud config list

# ── LIST LOAD BALANCERS ───────────────────────────────────────
gcloud compute forwarding-rules list
gcloud compute target-http-proxies list
gcloud compute url-maps list
gcloud compute backend-services list --global

# ── CHECK BACKEND HEALTH ──────────────────────────────────────
gcloud compute backend-services get-health api-backend-service \
  --global

gcloud compute backend-services get-health nginx-backend-service \
  --global

# ── VIEW INSTANCE GROUPS ──────────────────────────────────────
gcloud compute instance-groups managed list
gcloud compute instance-groups managed describe nginx-mig-us-central1-a \
  --zone=us-central1-a

# ── RESIZE MIG MANUALLY ───────────────────────────────────────
gcloud compute instance-groups managed resize nginx-mig-us-central1-a \
  --size=3 --zone=us-central1-a

# ── ROLLING UPDATE (new template) ─────────────────────────────
gcloud compute instance-groups managed rolling-action start-update \
  nginx-mig-us-central1-a \
  --version=template=nginx-instance-template \
  --zone=us-central1-a

# ── CHECK FORWARDING RULE IP ──────────────────────────────────
gcloud compute forwarding-rules describe nginx-alb-forwarding-rule \
  --global --format="value(IPAddress)"

# ── VIEW FIREWALL RULES ───────────────────────────────────────
gcloud compute firewall-rules list --filter="name~nginx OR name~api"

# ── SSH INTO A BACKEND VM ─────────────────────────────────────
gcloud compute ssh INSTANCE_NAME --zone=us-central1-a

# ── HEALTH CHECK STATUS ───────────────────────────────────────
gcloud compute health-checks list
gcloud compute health-checks describe nginx-health-check

# ── VIEW AUTOSCALING POLICY ───────────────────────────────────
gcloud compute instance-groups managed describe nginx-mig-us-central1-a \
  --zone=us-central1-a \
  --format="yaml(autoscaler)"

# ── TEST LB WITH CURL ─────────────────────────────────────────
LB_IP=$(gcloud compute forwarding-rules describe api-alb-forwarding-rule \
  --global --format="value(IPAddress)")
curl -s http://$LB_IP/api/health | python3 -m json.tool

# LOOP TEST (observe load distribution)
for i in {1..10}; do
  curl -s http://$LB_IP/ | grep -i hostname
done

# ── LOGS (LB access logs in Cloud Logging) ───────────────────
gcloud logging read \
  'resource.type="http_load_balancer"' \
  --limit=20 \
  --format="table(timestamp,httpRequest.requestUrl,httpRequest.status)"
```

---

## Cleanup Steps

> ⚠️ **Delete resources in order** to avoid dependency errors. Estimated cost savings: stops all billing for these resources.

### Phase 1 — Delete Load Balancers (UI)

1. **Network Services → Load Balancing**
2. Select all load balancers: `api-alb`, `api-nlb`, `nginx-alb`
3. Click **Delete** → Check **Delete all associated backends**
4. Click **Delete** to confirm

> This removes: Forwarding Rules, Target Proxies, URL Maps, Backend Services

---

### Phase 2 — Delete Health Checks

1. **Compute Engine → Health Checks** *(or Network Services → Health Checks)*
2. Select: `api-health-check`, `nlb-health-check`, `nginx-health-check`
3. Click **Delete**

---

### Phase 3 — Delete Instance Groups

1. **Compute Engine → Instance Groups**
2. Select all MIGs:
   - `api-instance-group`
   - `nlb-instance-group`
   - `nginx-mig-us-central1-a`
   - `nginx-mig-us-central1-b`
3. Click **Delete** → Confirm

> ⏳ This terminates all VM instances — wait until all show Deleted status.

---

### Phase 4 — Delete Instance Templates

1. **Compute Engine → Instance Templates**
2. Select:
   - `api-instance-template`
   - `nlb-instance-template`
   - `nginx-instance-template`
3. Click **Delete**

---

### Phase 5 — Release Static IP Addresses

1. **VPC Network → External IP Addresses**
2. Find reserved IPs:
   - `api-lb-ip`
   - `nlb-lb-ip`
   - `nginx-alb-ip`
3. Click **Release static address** for each

---

### Phase 6 — Delete Firewall Rules

1. **VPC Network → Firewall**
2. Select all lab firewall rules:
   - `allow-health-checks`
   - `allow-http-api`
   - `allow-nlb-traffic`
   - `nginx-allow-health-check`
   - `nginx-allow-http`
3. Click **Delete**

---

### Phase 7 — Delete VPC (if created custom)

1. **VPC Network → VPC Networks**
2. Click on `api-lb-vpc`
3. Scroll down → **Delete VPC Network**

> Note: Subnets are deleted automatically with the VPC.

---

### CLI Cleanup (Alternative — All at Once)

```bash
# Set your project
PROJECT_ID="your-project-id"
gcloud config set project $PROJECT_ID
ZONE_A="us-central1-a"
ZONE_B="us-central1-b"
REGION="us-central1"

# Delete Forwarding Rules
gcloud compute forwarding-rules delete api-alb-forwarding-rule --global -q
gcloud compute forwarding-rules delete api-nlb-forwarding-rule --region=$REGION -q
gcloud compute forwarding-rules delete nginx-alb-forwarding-rule --global -q

# Delete Target Proxies
gcloud compute target-http-proxies delete api-alb-target-proxy --global -q
gcloud compute target-http-proxies delete nginx-alb-target-proxy --global -q

# Delete URL Maps
gcloud compute url-maps delete api-alb --global -q
gcloud compute url-maps delete nginx-alb --global -q

# Delete Backend Services
gcloud compute backend-services delete api-backend-service --global -q
gcloud compute backend-services delete nginx-backend-service --global -q

# Delete Health Checks
gcloud compute health-checks delete api-health-check -q
gcloud compute health-checks delete nlb-health-check -q
gcloud compute health-checks delete nginx-health-check -q

# Delete Instance Groups
gcloud compute instance-groups managed delete api-instance-group \
  --zone=$ZONE_A -q
gcloud compute instance-groups managed delete nlb-instance-group \
  --zone=$ZONE_A -q
gcloud compute instance-groups managed delete nginx-mig-us-central1-a \
  --zone=$ZONE_A -q
gcloud compute instance-groups managed delete nginx-mig-us-central1-b \
  --zone=$ZONE_B -q

# Delete Instance Templates
gcloud compute instance-templates delete api-instance-template -q
gcloud compute instance-templates delete nlb-instance-template -q
gcloud compute instance-templates delete nginx-instance-template -q

# Release Static IPs
gcloud compute addresses delete api-lb-ip --global -q
gcloud compute addresses delete nlb-lb-ip --region=$REGION -q
gcloud compute addresses delete nginx-alb-ip --global -q

# Delete Firewall Rules
gcloud compute firewall-rules delete allow-health-checks -q
gcloud compute firewall-rules delete allow-http-api -q
gcloud compute firewall-rules delete allow-nlb-traffic -q
gcloud compute firewall-rules delete nginx-allow-health-check -q
gcloud compute firewall-rules delete nginx-allow-http -q

# Delete VPC (if custom)
gcloud compute networks subnets delete api-subnet --region=$REGION -q
gcloud compute networks delete api-lb-vpc -q

echo "✅ All lab resources deleted successfully!"
```

---

### Cleanup Verification

```bash
# Confirm everything is gone
gcloud compute forwarding-rules list
gcloud compute backend-services list --global
gcloud compute instance-groups managed list
gcloud compute addresses list
gcloud compute firewall-rules list --filter="name~nginx OR name~api OR name~nlb"
```

All commands should return **empty lists** — your project is clean. ✅

---

## Quick Reference Summary

| Resource Type     | Lab 1 (ALB-API) | Lab 2 (NLB) | Lab 3 (Nginx ALB) |
|-------------------|-----------------|-------------|-------------------|
| Instance Template | api-instance-template | nlb-instance-template | nginx-instance-template |
| Instance Group    | api-instance-group | nlb-instance-group | nginx-mig-* (x2) |
| Backend Service   | api-backend-service | Target Pool | nginx-backend-service |
| Health Check      | api-health-check (HTTP:8080) | nlb-health-check (TCP:80) | nginx-health-check (HTTP:/health) |
| Frontend Port     | 80 | 80 (TCP) | 80 / 443 |
| Network Tag       | api-backend | nlb-backend | nginx-lb-backend |

---

*Guide Version 1.0 — GCP Console UI Steps · No gcloud CLI required for setup*
