# Google Cloud Compute Engine — Training Labs
### Minimal-Resource Edition (UI Only)

---

> **Prerequisites**
> - A Google Cloud account with billing enabled
> - Access to [Google Cloud Console](https://console.cloud.google.com)
> - A project selected and Compute Engine API enabled
>
> **Cost-saving tip:** Most steps ask you to explore and cancel rather than create resources. Follow the DO NOT CREATE markers to avoid charges. Delete any resources you do create immediately after each lab.

---

## Table of Contents

1. [Lab 1 — Instance Templates and Managed Instance Groups](#lab-1--instance-templates-and-managed-instance-groups)
2. [Lab 2 — Sole-Tenant Nodes](#lab-2--sole-tenant-nodes)
3. [Lab 3 — Live Migration and Maintenance Events](#lab-3--live-migration-and-maintenance-events)
4. [Lab 4 — Preemptible Instances and Spot VMs](#lab-4--preemptible-instances-and-spot-vms)
5. [Clean-Up](#clean-up)

---

## Lab 1 — Instance Templates and Managed Instance Groups

**Duration:** ~45 minutes
**Objective:** Create a minimal instance template, deploy a small Managed Instance Group (MIG), observe autoscaling, and perform a rolling update.

---

### Part A — Create an Instance Template

#### Step 1: Navigate to Instance Templates

1. Open [Google Cloud Console](https://console.cloud.google.com).
2. Click Navigation Menu → Compute Engine → Instance templates.
3. Click **+ CREATE INSTANCE TEMPLATE**.

#### Step 2: Configure the Template

1. **Name:** `lab-template-v1`
2. **Location:** `Global`
3. **Machine configuration:**
   - Series: E2
   - Machine type: `e2-micro`
4. **Boot disk:** Click **CHANGE**
   - OS: Debian GNU/Linux 11
   - Type: Standard persistent disk
   - Size: `10 GB`
   - Click **SELECT**
5. **Firewall:** Check **Allow HTTP traffic**
6. Expand **Advanced options → Management**.
7. In the **Startup script** box, paste:

```bash
#!/bin/bash
sudo apt-get update -y
sudo apt-get install -y apache2
sudo echo "<h1>v1 — $(hostname)</h1>" > /var/www/html/index.html
sudo systemctl start apache2
```

8. Click **CREATE**.

> **Key concept:** Instance templates are immutable — you cannot edit them. To change a configuration, create a new version and apply a rolling update to the MIG.

---

### Part B — Deploy a Managed Instance Group

#### Step 1: Navigate to Instance Groups

1. Go to **Compute Engine → Instance groups**.
2. Click **+ CREATE INSTANCE GROUP**.
3. Select **New managed instance group (stateless)**.

#### Step 2: Configure the MIG

1. **Name:** `lab-web-mig`
2. **Instance template:** `lab-template-v1`
3. **Location:**
   - Mode: Single zone
   - Region: `us-central1`
   - Zone: `us-central1-a`
4. **Autoscaling:**
   - Mode: On: add and remove instances to the group
   - Minimum instances: `1`
   - Maximum instances: `2`
   - Signal: CPU utilization → Target: `60%`
   - Cool-down period: `60 seconds`
5. **Health check:** Click **CREATE A HEALTH CHECK**
   - Name: `lab-http-health-check`
   - Protocol: HTTP | Port: 80 | Path: `/`
   - Check interval: 10s | Unhealthy threshold: 3
   - Click **SAVE AND CONTINUE**
6. Click **CREATE**.

#### Step 3: Observe the MIG

1. Click `lab-web-mig` to open its details.
2. Watch the Status column: Provisioning → Staging → Running.
3. Click the **Instances** tab — you should see 1 VM created from the template.

---

### Part C — Perform a Rolling Update

#### Step 1: Create Template v2

1. Go to **Compute Engine → Instance templates → + CREATE INSTANCE TEMPLATE**.
2. **Name:** `lab-template-v2`
3. Same settings as v1, but change the startup script to:

```bash
#!/bin/bash
sudo apt-get update -y
sudo apt-get install -y apache2
sudo echo "<h1>v2 — $(hostname)</h1>" > /var/www/html/index.html
sudo systemctl start apache2
```

4. Click **CREATE**.

#### Step 2: Apply the Rolling Update

1. Go to **Compute Engine → Instance groups → lab-web-mig**.
2. Click **UPDATE VMs** at the top.
3. **New template:** `lab-template-v2`
4. **Update type:** Rolling update
5. **Maximum surge:** `0` | **Maximum unavailable:** `1`
6. Click **UPDATE INSTANCES**.
7. Watch the Instances tab update one by one.

> **Canary tip:** Use the Canary update option in the same dialog to roll out to a subset of instances first before committing fully.

---

### Lab 1 Checklist

- [ ] Created a global instance template (e2-micro) with a startup script
- [ ] Deployed a MIG with autoscaling (min 1, max 2)
- [ ] Configured an HTTP health check
- [ ] Created template v2 and applied a rolling update
- [ ] Understood why templates are immutable

---

## Lab 2 — Sole-Tenant Nodes

**Duration:** ~15 minutes
**Objective:** Understand sole-tenant node concepts and explore the configuration UI without creating any resources.

> **Cost note:** This lab is explore-only. Do not click Create at any point. Sole-tenant nodes provision a full dedicated physical host and are billed immediately.

---

### Background

| Feature | Shared VMs | Sole-Tenant Nodes |
|---|---|---|
| Physical host | Shared with other customers | Dedicated to you |
| Use case | General workloads | Compliance, BYOL licensing, isolation |
| Cost | Standard VM pricing | Full host price regardless of utilization |

---

### Part A — Explore the Node Template Form

1. Go to **Compute Engine → Sole-tenant nodes**.
2. Click **+ CREATE NODE GROUP**.
3. On the Configure node group page, click **Create a node template**.
4. Review the following fields without filling them in:
   - **Node type:** Note available types such as `n1-node-96-624` (96 vCPUs, 624 GB RAM) and their vCPU/RAM ratios.
   - **Node affinity labels:** Used to pin specific VMs to this node group (e.g., `workload=prod`).
   - **Local SSD and GPU options:** Available for workloads needing high-speed storage or accelerators.
5. **Click CANCEL** — do not create the node template.

---

### Part B — Explore VM Placement Settings

1. **Click CANCEL** on the node group creation page.
2. Go to **Compute Engine → VM instances → + CREATE INSTANCE**.
3. Expand **Advanced options → Sole tenancy**.
4. Observe the **Node affinity labels** field — this is how a VM is assigned to a sole-tenant node group.
5. **Click CANCEL** — do not create the VM.

> **Key use cases for sole-tenant nodes:**
> - BYOL licensing: software licenses tied to physical cores (e.g., Windows Server, SQL Server)
> - Regulatory compliance: workloads that must not share hardware with other tenants
> - Workload isolation: security-sensitive applications requiring physical separation

---

### Lab 2 Checklist

- [ ] Navigated the node template creation form without creating anything
- [ ] Identified at least two node types and their vCPU/RAM specs
- [ ] Observed node affinity labels on both the node template and VM creation forms
- [ ] Identified at least two real-world use cases for sole-tenant nodes

---

## Lab 3 — Live Migration and Maintenance Events

**Duration:** ~30 minutes
**Objective:** Configure VM maintenance policies, understand live migration vs termination, and review maintenance event notifications.

---

### Background: Host Maintenance Policies

| Policy | Behavior | Downtime |
|---|---|---|
| Migrate (default) | VM is transparently moved to another host | None (typically under 1 second) |
| Terminate | VM is shut down during maintenance | Yes — requires restart |

---

### Part A — Create a VM with Migrate Policy

1. Go to **Compute Engine → VM instances → + CREATE INSTANCE**.
2. **Name:** `lab-maintenance-vm`
3. **Machine type:** `e2-micro`
4. Scroll to **Advanced options → Management → Availability policies**.
5. Confirm **On host maintenance** is set to **Migrate VM instance** (default).
6. Confirm **Automatic restart** is On (default).
7. Click **CREATE**.

---

### Part B — Change Maintenance Policy

> Maintenance policy changes require the VM to be stopped first.

#### Step 1: Stop the VM

1. Check the box next to `lab-maintenance-vm` → click **STOP** → confirm.
2. Wait for the VM to reach the Stopped state.

#### Step 2: Switch to Terminate Policy

1. Click `lab-maintenance-vm` → click **EDIT**.
2. Scroll to **Availability policies**.
3. Change **On host maintenance** to **Terminate VM instance**.
4. Ensure **Automatic restart** is On — this restarts the VM after maintenance completes.
5. Click **SAVE**.

> **When to use Terminate:** GPUs and certain high-memory VMs cannot be live-migrated. These machine types force the Terminate policy and depend on Automatic restart to recover.

#### Step 3: Revert and Restart

1. Click **EDIT** → change back to **Migrate VM instance** → click **SAVE**.
2. Click **START / RESUME** → confirm.

---

### Part C — Maintenance Event Notifications

#### Step 1: Review VM Logs

1. Click `lab-maintenance-vm` → click the **LOGS** tab.
2. This opens Cloud Logging filtered for this VM.
3. Set the time range to **Last 7 days**.
4. Search for: `compute.instances.migrateOnHostMaintenance` or `hostError`.
5. On a new VM these events will not yet exist — this is expected.

#### Step 2: Metadata Server Notification Endpoint

Applications can poll the instance metadata server for advance warning of an upcoming migration:

```
GET http://metadata.google.internal/computeMetadata/v1/instance/maintenance-event
Header: Metadata-Flavor: Google
```

| Response | Meaning |
|---|---|
| `NONE` | No maintenance scheduled |
| `MIGRATE_ON_HOST_MAINTENANCE` | Live migration imminent (60-second notice) |
| `TERMINATE_ON_HOST_MAINTENANCE` | Termination imminent |

> **Best practice:** Long-running jobs should poll this endpoint every 5–10 seconds and checkpoint their state when a migration or termination is detected.

---

### Lab 3 Checklist

- [ ] Created a VM and confirmed the default Migrate maintenance policy
- [ ] Stopped the VM and changed the policy to Terminate with Auto-restart enabled
- [ ] Reverted to Migrate and restarted the VM
- [ ] Reviewed Cloud Logging for maintenance event entries
- [ ] Understood the metadata server maintenance notification endpoint

---

## Lab 4 — Preemptible Instances and Spot VMs

**Duration:** ~25 minutes
**Objective:** Understand the difference between Preemptible and Spot VMs, create a Spot VM, observe forced availability settings, and learn preemption best practices.

---

### Background: Preemptible vs Spot

| Feature | Preemptible (legacy) | Spot VM (current) |
|---|---|---|
| Max lifetime | 24 hours (hard limit) | No fixed limit |
| Preemption trigger | After 24 hours or when GCP needs capacity | Only when GCP needs capacity |
| Discount | Up to ~80% | Up to ~91% |
| Preemption notice | 30-second warning | 30-second warning |
| Maintenance policy | Forced: Terminate | Forced: Terminate |
| Auto-restart | Forced: Off | Forced: Off |
| Ideal for | Batch, fault-tolerant workloads | Same, plus longer-running tolerant jobs |

> **Note:** Preemptible VMs are the legacy model. Google now recommends Spot VMs for all new workloads. The console no longer surfaces Preemptible as a separate option.

---

### Part A — Create a Spot VM

#### Step 1: Configure the VM

1. Go to **Compute Engine → VM instances → + CREATE INSTANCE**.
2. **Name:** `lab-spot-vm`
3. **Machine type:** `e2-micro`
4. Scroll to **Advanced options → Management → Availability policies**.
5. Under **VM provisioning model**, select **Spot**.

#### Step 2: Observe Forced Settings

1. Note that **On host maintenance** is automatically set to Terminate and grayed out (cannot be changed).
2. Note that **Automatic restart** is forced Off (cannot be enabled for Spot VMs).
3. Observe the discounted price in the cost estimate panel on the right.
4. Click **CREATE**.

#### Step 3: Verify Spot Status

1. In the VM list, find `lab-spot-vm`.
2. Check the **Preemptibility** column — it should show On.
3. Click the VM name → under **Availability policies**, confirm:
   - Preemptibility: On
   - Automatic restart: Off

---

### Part B — Preemption Handling

#### Step 1: Metadata Endpoint for Preemption Detection

Applications on a Spot VM can detect an imminent preemption via the metadata server:

```
GET http://metadata.google.internal/computeMetadata/v1/instance/preempted
Header: Metadata-Flavor: Google
```

| Response | Meaning |
|---|---|
| `FALSE` | VM is running normally |
| `TRUE` | VM has been preempted |

A 30-second preemption notice is sent via ACPI G2 Soft-Off. Applications can trap the SIGTERM signal to gracefully checkpoint work before shutdown.

#### Step 2: Spot VMs in Managed Instance Groups (Explore Only)

1. Go to **Compute Engine → Instance groups → + CREATE INSTANCE GROUP**.
2. Select **New managed instance group (stateless)**.
3. After selecting an instance template, scroll to the **Autoscaling** section.
4. Note that you can mix Standard and Spot provisioning at the MIG level for cost-optimized fleets.
5. **Click CANCEL** — do not create the MIG.

> **Best practices for Spot VMs:**
> - Checkpoint work frequently to persistent disk or Cloud Storage.
> - Use MIGs with health checks so preempted instances are replaced automatically.
> - Avoid Spot for latency-sensitive or stateful services.
> - Ideal workloads: batch processing, ML training, rendering, data pipelines.

---

### Part C — Compare Standard vs Spot Pricing

1. Go to **Compute Engine → VM instances → + CREATE INSTANCE**.
2. **Machine type:** `e2-standard-4`
3. Note the Monthly estimate on the right panel (Standard pricing).
4. Go to **Advanced options → Management → Availability policies**.
5. Switch **VM provisioning model** to **Spot**.
6. Observe the updated Monthly estimate — the discount is clearly visible.
7. **Click CANCEL** — do not create the VM.

---

### Lab 4 Checklist

- [ ] Explained the difference between legacy Preemptible and Spot VMs
- [ ] Created a Spot VM and observed forced Terminate policy and disabled Auto-restart
- [ ] Confirmed Preemptibility = On in the VM details
- [ ] Understood the 30-second preemption notice and metadata endpoint
- [ ] Compared Standard vs Spot pricing using the console estimator
- [ ] Identified at least two ideal Spot VM workload types

---

## Clean-Up

Delete resources in this order after completing all labs to avoid ongoing charges.

| Resource | Location | Action |
|---|---|---|
| `lab-web-mig` | Compute Engine → Instance groups | Delete |
| `lab-maintenance-vm` | Compute Engine → VM instances | Delete |
| `lab-spot-vm` | Compute Engine → VM instances | Delete |
| `lab-http-health-check` | Compute Engine → Health checks | Delete |
| `lab-template-v1` | Compute Engine → Instance templates | Delete |
| `lab-template-v2` | Compute Engine → Instance templates | Delete |
| Any orphaned disks | Compute Engine → Disks | Delete |

---

## Summary: Key Concepts

| Topic | Key Takeaway |
|---|---|
| Instance templates | Immutable VM blueprints; required for MIGs; create a new version for any change |
| Managed Instance Groups | Auto-heal, autoscale, and rolling-update VM fleets with zero downtime |
| Sole-tenant nodes | Dedicated physical hosts for compliance, BYOL licensing, or workload isolation |
| Live migration | Default policy; GCP moves your VM with near-zero downtime during host maintenance |
| Terminate + Auto-restart | Required for GPUs; VM stops then restarts automatically after maintenance |
| Spot VMs | Up to 91% discount; can be preempted anytime; maintenance forced to Terminate |
| Preemption handling | Checkpoint work to persistent storage; use MIGs for automatic replacement |

---

*Labs designed for Google Cloud Console UI — No gcloud CLI required.*
*All labs use e2-micro or explore-only flows to minimize cost.*
*Refer to [official GCP documentation](https://cloud.google.com/compute/docs) for the latest UI changes.*
