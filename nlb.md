# GCP Network Load Balancer (NLB) — Cloud Console Creation Guide

> **Assumption:** Managed Instance Groups (MIGs) or Instance Groups (IGs) are already created.

---

## Step 1 — Navigate to Load Balancing

1. Go to **Navigation Menu → Network Services → Load Balancing**
2. Click **+ Create Load Balancer**
3. Select **Network Load Balancer (TCP/UDP/SSL)**
4. Click **Start Configuration**

---

## Step 2 — Load Balancer Type Wizard

Progress through 4 wizard screens, clicking **Next** on each:

| Screen | Selection |
|--------|-----------|
| Type | **Proxy load balancer** (TCP/TLS, advanced traffic control) |
| Facing | **Public facing (external)** |
| Deployment | **Best for regional workloads** (single region) |
| Generation | **Global external Application Load Balancer** (`EXTERNAL_MANAGED`) ✅ Recommended |

Click **Configure** on the final screen.

---

## Step 3 — Basic Configuration

| Field | Value |
|-------|-------|
| Name | `api-nlb` |

---

## Step 4 — Frontend Configuration

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

## Step 5 — Backend Service

1. Click **Backend configuration → Create a backend service**

| Field | Value |
|-------|-------|
| Name | `nlb-backend-service` |
| Backend type | `Instance group` |
| Region | `us-central1` *(or your region)* |

2. Click **Add Backend**:

| Field | Value |
|-------|-------|
| Instance group | *(select your existing MIG / IG)* |
| Port numbers | `80` |

---

## Step 6 — Health Check

1. Inside the backend service form, click **Create Health Check**

| Field | Value |
|-------|-------|
| Name | `nlb-health-check` |
| Protocol | `TCP` |
| Port | `80` |

2. Click **Save** to save the health check
3. Click **Save** to save the backend service

---

## Step 7 — Review & Finalize

1. Click **Review and Finalize**
2. Verify the frontend and backend summary
3. Click **Create**

>  Wait **2–3 minutes** for provisioning to complete.

**Test the endpoint:**
```bash
curl http://<FRONTEND_IP>
```

---

## Quick Reference Summary

| Component | Value |
|-----------|-------|
| LB Name | `api-nlb` |
| LB Type | Network (TCP/UDP/SSL) — Proxy |
| Scope | Regional |
| Frontend Name | `nlb-frontend` |
| Frontend Protocol | TCP |
| Frontend Port | `80` |
| IP Address | Ephemeral |
| Backend Service | `nlb-backend-service` |
| Health Check Name | `nlb-health-check` |
| Health Check Protocol | TCP |
| Health Check Port | `80` |

---

## CLI Verification (Post-Creation)

```bash
# Check NLB backend health
gcloud compute backend-services get-health nlb-backend-service --global

# Get frontend IP and test
NLB_IP=$(gcloud compute forwarding-rules describe api-nlb-forwarding-rule \
  --region=us-central1 --format="value(IPAddress)")
curl -s http://$NLB_IP
```

---

*Guide Version 1.0 — NLB Cloud Console UI Steps*
