# GCP Load Balancer Creation Guide
### ALB · NLB — Frontend, Backend & Health Check Only

> **Assumption:** Managed Instance Groups (MIGs) and Instance Groups (IGs) are already created.  
> This guide covers **only** Load Balancer creation — frontend, backend service, and health check configuration.

---

## Table of Contents

1. [Application Load Balancer (ALB)](#1-application-load-balancer-alb)
2. [Network Load Balancer (NLB)](#2-network-load-balancer-nlb)
3. [Verification Checklist](#3-verification-checklist)

---

## 1. Application Load Balancer (ALB)

### Step 1 — Start Load Balancer Creation

1. Go to **Navigation Menu → Network Services → Load Balancing**
2. Click **+ Create Load Balancer**
3. Select **Application Load Balancer (HTTP/HTTPS)**
4. Click **Start Configuration**
5. Select:
   - **Internet facing:** From Internet to my VMs
   - **Scope:** Global (Classic)
6. **Name:** `api-alb`

---

### Step 2 — Frontend Configuration

1. Click **Frontend configuration**

| Field | Value |
|-------|-------|
| Name | `api-frontend` |
| Protocol | `HTTP` |
| IP version | `IPv4` |
| IP address | Create new → Name: `api-lb-ip` |
| Port | `80` |

2. Click **Done**

---

### Step 3 — Backend Service

1. Click **Backend configuration → Create a backend service**

| Field | Value |
|-------|-------|
| Name | `api-backend-service` |
| Backend type | `Instance group` |
| Protocol | `HTTP` |
| Timeout | `30s` |

2. Click **Add Backend:**

| Field | Value |
|-------|-------|
| Instance group | *(select your existing MIG/IG)* |
| Port numbers | `80` |
| Balancing mode | `Utilization` |

---

### Step 4 — Health Check

1. Within the backend service form, click **Create Health Check**

| Field | Value |
|-------|-------|
| Name | `api-health-check` |
| Protocol | `HTTP` |
| Port | `80` |
| Request path | `/` |
| Check interval | `10s` |
| Healthy threshold | `2` |
| Unhealthy threshold | `3` |

2. Click **Save** to save the health check
3. Click **Create** to save the backend service

---

### Step 5 — Host and Path Rules (URL Map)

1. Click **Host and path rules**
2. Configure routing:

| Mode | Setting |
|------|---------|
| Advanced host and path rule | Enabled |
| Hosts | `*` |
| Paths | `/api/*` → `api-backend-service` |
| Default action | `api-backend-service` |

3. Click **Done**

---

### Step 6 — Finalize ALB

1. Click **Review and Finalize**
2. Verify frontend and backend summary
3. Click **Create**

> ⏳ Wait **3–5 minutes** for provisioning to complete.

**Test:**
```
http://<FRONTEND_IP>/

---

---

## 2. Network Load Balancer (NLB)

### Step 1 — Start Load Balancer Creation

1. Go to **Navigation Menu → Network Services → Load Balancing**
2. Click **+ Create Load Balancer**
3. Select **Network Load Balancer (TCP/UDP/SSL)**
4. Click **Start Configuration**

---

### Step 2 — Load Balancer Type Wizard

Progress through the wizard screens:

| Screen | Selection |
|--------|-----------|
| Type | **Proxy load balancer** (TCP/TLS, advanced traffic control) |
| Facing | **Public facing (external)** |
| Deployment | **Best for regional workloads** (single region) |
| Generation | **Global external Application Load Balancer** (`EXTERNAL_MANAGED`) ✅ Recommended |

Click **Next** on each screen, then **Configure** on the final screen.

---

### Step 3 — Basic Configuration

- **Name:** `api-nlb`

---

### Step 4 — Frontend Configuration

1. Click **Frontend configuration**

| Field | Value |
|-------|-------|
| Name | `nlb-frontend` |
| Protocol | `TCP` |
| Network Service Tier | `Premium` |
| IP address | `Ephemeral` *(or reserve a static IP)* |
| Port | `80` |

2. Click **Done**

---

### Step 5 — Backend Service

1. Click **Backend configuration → Create a backend service**

| Field | Value |
|-------|-------|
| Name | `nlb-backend-service` |
| Backend type | `Instance group` |
| Region | `us-central1` *(or your region)* |

2. Click **Add Backend:**

| Field | Value |
|-------|-------|
| Instance group | *(select your existing MIG/IG)* |
| Port numbers | `80` |

---

### Step 6 — Health Check

1. Within the backend service form, click **Create Health Check**

| Field | Value |
|-------|-------|
| Name | `nlb-health-check` |
| Protocol | `TCP` |
| Port | `80` |

2. Click **Save** to save the health check
3. Click **Save** to save the backend service

---

### Step 7 — Finalize NLB

1. Click **Review and Finalize**
2. Verify frontend and backend summary
3. Click **Create**

> ⏳ Wait **2–3 minutes** for provisioning.

**Test:**
```bash
curl http://<FRONTEND_IP>
```

---

---

## 3. Verification Checklist

| Check | Where to Look | Expected |
|-------|---------------|----------|
| LB Status | Network Services → Load Balancing | ✅ Green |
| Backend Health | Click LB → Backend tab | Healthy |
| Health Check | Compute Engine → Health Checks | Passing |
| Frontend IP | Load Balancer → Frontend tab | IP assigned |
| HTTP Response | `curl http://<IP>/` | `200 OK` |

### Quick CLI Verification

```bash
# List all load balancers
gcloud compute forwarding-rules list

# Check ALB backend health
gcloud compute backend-services get-health api-backend-service --global

# Check NLB backend health
gcloud compute backend-services get-health nlb-backend-service --global

# Test ALB endpoint
LB_IP=$(gcloud compute forwarding-rules describe api-frontend-karthik --global \
  --format="value(IPAddress)")
curl -s http://$LB_IP/ 

# Test NLB endpoint
NLB_IP=$(gcloud compute forwarding-rules describe api-nlb-forwarding-rule \
  --region=us-central1 --format="value(IPAddress)")
curl -s http://$NLB_IP
```

---

## Quick Reference Summary

| Component | ALB | NLB |
|-----------|-----|-----|
| LB Type | Application (HTTP/HTTPS) | Network (TCP/UDP/SSL) |
| Scope | Global Classic | Regional |
| Frontend Protocol | HTTP | TCP |
| Frontend Port | `80` | `80` |
| Static IP Name | `api-lb-ip` | Ephemeral |
| Backend Service | `api-backend-service` | `nlb-backend-service` |
| Health Check Protocol | HTTP | TCP |
| Health Check Port | `8080` | `80` |
| Health Check Path | `/` | — |

---

*Guide Version 1.1 — MIG/IG already provisioned · Correct UI order: Frontend → Backend → Health Check*
