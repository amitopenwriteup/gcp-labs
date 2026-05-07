# Google Cloud VPC Networking — Comprehensive Lab Guide

> **Skill Level:** Associate / Professional Cloud Engineer Prep
> **Estimated Time:** 3–4 hours
> **Prerequisites:** Active GCP project with billing enabled, Owner or Editor IAM role

---

## Table of Contents

1. [Module 1 — VPC Network Fundamentals](#module-1)
2. [Module 2 — Firewall Rules & Security](#module-2)
3. [Module 3 — Advanced Networking](#module-3)
4. [Validation & Cleanup](#validation--cleanup)
5. [Knowledge Check](#knowledge-check)

---

## Background Concepts

### VPC Network Types

| Type | Description | Use Case |
|---|---|---|
| **Auto mode** | One subnet per region, auto-created (`10.128.0.0/9`) | Quick dev/test environments |
| **Custom mode** | You define all subnets, ranges, and regions | Production; precise IP control |
| **Legacy** | Pre-VPC flat network; deprecated | Avoid — do not create new ones |

**Key rules:**
- A VPC is a **global** resource; subnets are **regional**.
- VMs in the same VPC can communicate across regions using internal IPs (no VPN needed).
- You cannot convert a custom-mode VPC back to auto mode.
- CIDR ranges of subnets within the same VPC must **not** overlap.

### Subnetting in GCP

Each subnet has a **primary CIDR range** and optional **secondary ranges** (used by GKE Pods and Services).

```
Example: 10.10.0.0/24
  → Network address : 10.10.0.0
  → Broadcast       : 10.10.0.255
  → Usable hosts    : 10.10.0.1 – 10.10.0.254  (254 addresses)
  → GCP reserves 4 per subnet: .0, .1, .2 (default gateway), .255
```

**Subnet expansion** is supported — you can widen a prefix (e.g., `/24` → `/20`) without recreating VMs.

### Regional vs. Global Resources

| Resource | Scope |
|---|---|
| VPC Network | Global |
| Subnet | Regional |
| VM instance | Zonal |
| Static external IP (Regional) | Regional |
| Static external IP (Global) | Global (load balancers only) |
| Cloud Router | Regional |
| VPN Gateway | Regional |
| Global Load Balancer | Global |

### VPC Peering

- Connects two VPC networks (same or different projects/orgs) so VMs exchange traffic over **internal IPs**.
- **Non-transitive** — if VPC-A peers with VPC-B and VPC-B peers with VPC-C, VPC-A cannot reach VPC-C.
- Peering is established from **both sides** independently.
- CIDR ranges of peered VPCs **must not** overlap.
- Peering does **not** export/import custom routes by default — enable route exchange explicitly.

### Shared VPC

- A **host project** owns the VPC; **service projects** attach to it and deploy resources into shared subnets.
- Centralises network management while letting separate teams own their workloads.
- Requires `roles/compute.xpnAdmin` on the host project.
- Service projects see only the subnets the host project grants them access to.

---

## Module 1 — VPC Network Fundamentals {#module-1}

### Lab 1.1 — Create a Custom VPC with Multiple Subnets

#### Objective
Build a custom VPC with three subnets in different regions representing web, app, and data tiers.

#### Step 1 — Create the Custom VPC

1. In the GCP Console, navigate to **VPC Network → VPC Networks**.
2. Click **Create VPC Network**.
3. Configure the following:

   | Field | Value |
   |---|---|
   | Name | `lab-vpc` |
   | Description | `Multi-tier custom VPC for lab` |
   | Subnet creation mode | **Custom** |

4. Do **not** add subnets yet — click **Add Subnet** separately below.

#### Step 2 — Add the Web Subnet

Click **Add Subnet** and fill in:

| Field | Value |
|---|---|
| Name | `web-subnet` |
| Region | `us-central1` |
| IP address range | `10.10.1.0/24` |
| Private Google Access | **On** |
| Flow logs | **On** (set sample rate to 0.5) |

#### Step 3 — Add the App Subnet

Click **Add Subnet** again:

| Field | Value |
|---|---|
| Name | `app-subnet` |
| Region | `us-central1` |
| IP address range | `10.10.2.0/24` |
| Private Google Access | **On** |
| Flow logs | **On** |

#### Step 4 — Add the Data Subnet

| Field | Value |
|---|---|
| Name | `data-subnet` |
| Region | `us-central1` |
| IP address range | `10.10.3.0/24` |
| Private Google Access | **On** |
| Flow logs | **On** |

#### Step 5 — Set Dynamic Routing Mode

Under **Dynamic routing mode**, select **Global** (allows Cloud Router to advertise all subnets to on-premises routers).

Click **Create**.

---

#### Equivalent gCloud Commands

```bash
# Create the VPC
gcloud compute networks create lab-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=global

# Create web subnet
gcloud compute networks subnets create web-subnet \
  --network=lab-vpc \
  --region=us-central1 \
  --range=10.10.1.0/24 \
  --enable-private-ip-google-access \
  --enable-flow-logs \
  --logging-sample-rate=0.5

# Create app subnet
gcloud compute networks subnets create app-subnet \
  --network=lab-vpc \
  --region=us-central1 \
  --range=10.10.2.0/24 \
  --enable-private-ip-google-access \
  --enable-flow-logs

# Create data subnet
gcloud compute networks subnets create data-subnet \
  --network=lab-vpc \
  --region=us-central1 \
  --range=10.10.3.0/24 \
  --enable-private-ip-google-access \
  --enable-flow-logs
```

---

### Lab 1.2 — VPC Peering

#### Objective
Create a second VPC and peer it with `lab-vpc`.

#### Step 1 — Create the Peer VPC

```bash
gcloud compute networks create peer-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=regional

gcloud compute networks subnets create peer-subnet \
  --network=peer-vpc \
  --region=us-central1 \
  --range=10.20.1.0/24
```

#### Step 2 — Create Peering from lab-vpc → peer-vpc

1. Navigate to **VPC Network → VPC Network Peering**.
2. Click **Create Connection → Continue**.

   | Field | Value |
   |---|---|
   | Name | `lab-to-peer` |
   | Your VPC network | `lab-vpc` |
   | Peered VPC network | In same project |
   | VPC network name | `peer-vpc` |
   | Export custom routes |  Enabled |
   | Import custom routes |  Enabled |

3. Click **Create**.

#### Step 3 — Create Peering from peer-vpc → lab-vpc

Repeat the process:

| Field | Value |
|---|---|
| Name | `peer-to-lab` |
| Your VPC network | `peer-vpc` |
| VPC network name | `lab-vpc` |
| Export custom routes |  Enabled |
| Import custom routes | Enabled |

> **Note:** Both peerings must be **Active** before traffic flows. Status shows `INACTIVE` until both sides are configured.

#### Verify Peering Status

```bash
gcloud compute networks peerings list --network=lab-vpc
```

Expected output column `STATE`: `ACTIVE`

---

## Module 2 — Firewall Rules & Security {#module-2}

### Concepts

#### Ingress vs. Egress

| Direction | Applies to | Default GCP behaviour |
|---|---|---|
| **Ingress** | Incoming traffic to VMs | **Deny all** (implied rule, priority 65535) |
| **Egress** | Outgoing traffic from VMs | **Allow all** (implied rule, priority 65535) |

Rules are evaluated in **priority order** (lower number = higher priority, range 0–65535).

#### Targeting with Network Tags vs. Service Accounts

| Mechanism | How it works | Best for |
|---|---|---|
| **Network tag** | String label on a VM; firewall rule targets the tag | Simple grouping; easy to apply |
| **Service account** | Identity of the VM; firewall rule targets the SA | Fine-grained, identity-based control |

Service account targeting is more secure because tags can be self-assigned by users with VM edit rights, whereas service accounts are controlled by IAM.

---

### Lab 2.1 — Layered Firewall Rules with Network Tags

#### Step 1 — Allow IAP SSH to Web Tier

Identity-Aware Proxy (IAP) tunnels SSH through Google's network. Allow its IP range (`35.235.240.0/20`) to reach web VMs.

1. Navigate to **VPC Network → Firewall**.
2. Click **Create Firewall Rule**.

   | Field | Value |
   |---|---|
   | Name | `allow-iap-ssh-web` |
   | Network | `lab-vpc` |
   | Priority | `1000` |
   | Direction | Ingress |
   | Action on match | Allow |
   | Targets | Specified target tags |
   | Target tags | `web-server` |
   | Source filter | IPv4 ranges |
   | Source IPv4 ranges | `35.235.240.0/20` |
   | Protocols/ports | TCP: `22` |

3. Click **Create**.

#### Step 2 — Allow HTTP/HTTPS from Internet to Web Tier

| Field | Value |
|---|---|
| Name | `allow-http-https-web` |
| Network | `lab-vpc` |
| Priority | `1000` |
| Direction | Ingress |
| Action | Allow |
| Target tags | `web-server` |
| Source IP ranges | `0.0.0.0/0` |
| Protocols/ports | TCP: `80, 443` |

#### Step 3 — Allow Web → App Communication

| Field | Value |
|---|---|
| Name | `allow-web-to-app` |
| Network | `lab-vpc` |
| Priority | `1000` |
| Direction | Ingress |
| Action | Allow |
| Target tags | `app-server` |
| Source tags | `web-server` |
| Protocols/ports | TCP: `8080` |

#### Step 4 — Allow App → Data Communication

| Field | Value |
|---|---|
| Name | `allow-app-to-data` |
| Network | `lab-vpc` |
| Priority | `1000` |
| Direction | Ingress |
| Action | Allow |
| Target tags | `data-server` |
| Source tags | `app-server` |
| Protocols/ports | TCP: `3306` (MySQL) |

#### Step 5 — Deny All Other Ingress (Explicit)

| Field | Value |
|---|---|
| Name | `deny-all-ingress` |
| Network | `lab-vpc` |
| Priority | `65000` |
| Direction | Ingress |
| Action | **Deny** |
| Targets | All instances |
| Source IP ranges | `0.0.0.0/0` |
| Protocols/ports | All |

> The default implied deny rule at priority 65535 handles this, but an explicit rule at 65000 lets you **log** denied traffic.

---

#### Equivalent gCloud Commands

```bash
# IAP SSH to web tier
gcloud compute firewall-rules create allow-iap-ssh-web \
  --network=lab-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=web-server

# HTTP/HTTPS to web tier
gcloud compute firewall-rules create allow-http-https-web \
  --network=lab-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server

# Web to app
gcloud compute firewall-rules create allow-web-to-app \
  --network=lab-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:8080 \
  --source-tags=web-server \
  --target-tags=app-server

# App to data
gcloud compute firewall-rules create allow-app-to-data \
  --network=lab-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:3306 \
  --source-tags=app-server \
  --target-tags=data-server

# Explicit deny all with logging
gcloud compute firewall-rules create deny-all-ingress \
  --network=lab-vpc \
  --direction=INGRESS \
  --priority=65000 \
  --action=DENY \
  --rules=all \
  --source-ranges=0.0.0.0/0 \
  --enable-logging
```

---


## Module 3 — Advanced Networking {#module-3}

### Private Google Access

Enables VM instances **without external IPs** to reach Google APIs and services (Cloud Storage, BigQuery, etc.) over Google's private network.

- Configured **per subnet**.
- Requires the VM to have no external IP.
- DNS resolves `*.googleapis.com` to private IP ranges (`199.36.153.8/30` for `private.googleapis.com`).

#### Enable in Console

1. Navigate to **VPC Network → VPC Networks → lab-vpc**.
2. Click on **web-subnet**.
3. Click **Edit**.
4. Set **Private Google Access** to **On**.
5. Click **Save**.

Already enabled via CLI in Lab 1.1 (`--enable-private-ip-google-access`).

#### Verify PGA is Working

```bash
# SSH into a VM with no external IP via IAP
gcloud compute ssh web-vm-1 --tunnel-through-iap --zone=us-central1-a

# From inside the VM, test access to GCS (should succeed)
curl -o /dev/null -s -w "%{http_code}" \
  "https://storage.googleapis.com/storage/v1/b?project=$(gcloud config get-value project)"
# Expected: 200
```

---

### Private Service Connect (PSC)

PSC lets you privately access **managed Google services** or **third-party services** using **internal IP addresses** from your VPC — without traffic leaving Google's network.

#### PSC vs. Private Google Access

| Feature | Private Google Access | Private Service Connect |
|---|---|---|
| Target | Google APIs (GCS, BQ, etc.) | Google APIs **and** managed services (Cloud SQL, etc.) |
| IP type | Connects to shared Google ranges | Assigns a **private IP in your VPC** |
| DNS | Requires `private.googleapis.com` config | Custom DNS or auto-generated |
| Flexibility | Less granular | More granular; per-service endpoints |

#### Lab 3.1 — Create a PSC Endpoint for Google APIs

1. Navigate to **VPC Network → Private Service Connect**.
2. Click **Connect Endpoint → Google APIs**.

   | Field | Value |
   |---|---|
   | Endpoint name | `psc-google-apis` |
   | Target | `all-apis` (or `vpc-sc` for VPC Service Controls) |
   | Network | `lab-vpc` |
   | Subnetwork | `web-subnet` |
   | IP address | Create new: `psc-ip` → `10.10.1.100` |

3. Click **Add Endpoint**.

---

## Validation & Cleanup {#validation--cleanup}

### Pre-Cleanup Checklist

Run these checks before destroying resources:

```bash
# Confirm firewall rules
gcloud compute firewall-rules list --filter="network:lab-vpc"

# Confirm peering status
gcloud compute networks peerings list --network=lab-vpc

# Confirm subnets
gcloud compute networks subnets list --filter="network:lab-vpc"
```

### Cleanup Commands

```bash
# Delete firewall rules
gcloud compute firewall-rules delete \
  allow-iap-ssh-web allow-http-https-web allow-web-to-app \
  allow-app-to-data deny-all-ingress --quiet

# Delete peerings
gcloud compute networks peerings delete lab-to-peer --network=lab-vpc --quiet
gcloud compute networks peerings delete peer-to-lab --network=peer-vpc --quiet

# Delete subnets
gcloud compute networks subnets delete web-subnet app-subnet data-subnet \
  --region=us-central1 --quiet
gcloud compute networks subnets delete peer-subnet \
  --region=us-central1 --quiet

# Delete VPCs
gcloud compute networks delete lab-vpc peer-vpc --quiet
```

---

## Knowledge Check {#knowledge-check}

Answer these questions to consolidate what you've learned. Answers are at the bottom.

**Q1.** You have VPC-A peered with VPC-B, and VPC-B peered with VPC-C. Can a VM in VPC-A ping a VM in VPC-C?

**Q2.** A VM in `data-subnet` has no external IP. Which feature must be enabled on the subnet to allow it to reach Cloud Storage?

**Q3.** A firewall rule at priority `500` allows TCP:22. Another rule at priority `1000` denies TCP:22 from the same source. Which rule wins?

**Q4.** What is the minimum number of peering connections required to set up bidirectional peering between two VPCs?

**Q5.** You want to apply a firewall rule to all projects in your organisation without configuring each VPC individually. What should you use?

**Q6.** A subnet currently uses `10.10.1.0/24`. You need more IPs. Can you change it to `10.10.1.0/20` without recreating VMs?

---

### Answers

**A1.** No. VPC peering is non-transitive. VPC-A and VPC-C must be peered directly.

**A2.** Private Google Access must be enabled on `data-subnet`.

**A3.** The rule at priority `500` wins (lower number = higher priority). The allow rule takes effect.

**A4.** Two — one peering created from each VPC toward the other.

**A5.** A Hierarchical Firewall Policy applied at the Organisation or Folder level.

**A6.** Yes. GCP supports subnet range expansion (widening prefix). You can expand to `/20` without disrupting existing VMs. You cannot shrink a range.

---

*Lab authored for GCP Associate Cloud Engineer / Professional Cloud Network Engineer preparation. Always verify that current GCP Console UI matches these steps, as the interface is updated regularly.*
