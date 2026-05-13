# GCP Lab: Nginx VM with Identity-Aware Proxy and Zero Trust Networking

**Level:** Intermediate  
**Duration:** 75 minutes  
**Prerequisites:** A GCP project with billing enabled, Owner or Editor IAM role, existing VPC network

---

## Overview

In this lab you will deploy a Compute Engine VM running Nginx, lock down all direct internet access, and expose the web server securely through Identity-Aware Proxy (IAP). By the end you will have a working Zero Trust setup where access to the VM is gated by Google identity — no VPN, no public SSH port, no open firewall rules.

**What you will build:**

- A Compute Engine VM running Nginx (no public IP) attached to your existing VPC
- Firewall rules that allow only IAP traffic
- IAP TCP tunneling for SSH access
- IAP TCP tunneling for authenticated HTTP access (no load balancer or SSL certificate required)
- OS Login for identity-based SSH authentication

---

## Architecture

```
User (Google Identity)
        |
        v
  Google Front End
  (iap.googleapis.com)
        |
   [IAP checks identity
    and IAM policy]
        |
        v
  IAP TCP Tunnel
  (port 22 for SSH, port 80 for HTTP)
        |
        v
  VM: nginx-server        <-- No public IP
  (us-central1-a)
        |
  VPC: lab-vpc  (pre-existing)
  Subnet: lab-subnet (10.0.0.0/24)
```

---

## Part 1 — Enable Required APIs

1. Open the [GCP Console](https://console.cloud.google.com) and select your project from the top project picker.

2. Navigate to **APIs & Services > Library** using the left navigation menu.

3. Search for and enable each of the following APIs by clicking the API name, then clicking **Enable**:

   - `Compute Engine API`
   - `Identity-Aware Proxy API`
   - `Cloud Resource Manager API`
   - `OS Login API`

4. Wait for each API to finish enabling before moving on. You will see a green checkmark and the button will change to **Manage** when done.

---

## Part 2 — Verify the Existing VPC Network

> Your VPC (`lab-vpc`) is already created. This part verifies it is correctly configured before you attach the VM to it.

### 2.1 Confirm the VPC and Subnet Exist

1. In the left navigation menu go to **VPC network > VPC networks**.
2. Locate **`lab-vpc`** in the list and click it.
3. Under the **Subnets** tab confirm that **`lab-subnet`** exists in `us-central1` with the range `10.0.0.0/24`.
4. Confirm that **Private Google Access** is set to **On** for `lab-subnet`. If it is Off:
   - Click `lab-subnet`.
   - Click **Edit**.
   - Toggle **Private Google Access** to **On**.
   - Click **Save**.

### 2.2 Confirm No Unwanted Ingress Rules

1. Go to **VPC network > Firewall**.
2. Filter by **Network: lab-vpc**.
3. Verify there is no existing rule that allows ingress from `0.0.0.0/0` on port 22 or port 80. If one exists, note its name — you may need to delete or adjust it after creating the tighter IAP-only rules in Part 3.

---

## Part 3 — Configure Firewall Rules

You will create two firewall rules: one that allows IAP to reach the VM on SSH, and one that allows IAP traffic on port 80.

### 3.1 Allow IAP for SSH (port 22)

1. Go to **VPC network > Firewall**.
2. Click **Create firewall rule**.
3. Fill in:

   | Field | Value |
   |---|---|
   | Name | `allow-iap-ssh` |
   | Network | `lab-vpc` |
   | Priority | `1000` |
   | Direction of traffic | Ingress |
   | Action on match | Allow |
   | Targets | Specified target tags |
   | Target tags | `iap-ssh-target` |
   | Source filter | IPv4 ranges |
   | Source IPv4 ranges | `35.235.240.0/20` |
   | Protocols and ports | TCP: `22` |

4. Click **Create**.

> The range `35.235.240.0/20` is Google's IAP forwarder range. Traffic arriving from this range has already passed IAP authentication.

### 3.2 Allow IAP for HTTP (port 80)

1. Click **Create firewall rule** again.
2. Fill in:

   | Field | Value |
   |---|---|
   | Name | `allow-iap-http` |
   | Network | `lab-vpc` |
   | Priority | `1000` |
   | Direction of traffic | Ingress |
   | Action on match | Allow |
   | Targets | Specified target tags |
   | Target tags | `http-server` |
   | Source filter | IPv4 ranges |
   | Source IPv4 ranges | `35.235.240.0/20` |
   | Protocols and ports | TCP: `80` |

3. Click **Create**.

> Unlike a load balancer setup, IAP TCP forwarding for port 80 only requires the IAP forwarder range — no additional health checker ranges are needed.

### 3.3 Verify Default Deny

Custom VPC networks have an implied deny-all ingress rule at priority 65535. To confirm:

1. On the Firewall page filter by **Network: lab-vpc**.
2. Scroll to the bottom and confirm you see `default-deny-all-ingress` with priority `65535`.  
   If it is not visible, it is implied and still enforced — GCP does not always surface implied rules in the UI.

---

## Part 4 — Create the VM

### 4.1 Launch the Instance

1. Go to **Compute Engine > VM instances**.
2. Click **Create instance**.
3. Configure the following:

   **Basics**

   | Field | Value |
   |---|---|
   | Name | `nginx-server` |
   | Region | `us-central1` |
   | Zone | `us-central1-a` |

   **Machine configuration**

   | Field | Value |
   |---|---|
   | Series | E2 |
   | Machine type | `e2-micro` |

   **Boot disk** — click **Change**:

   | Field | Value |
   |---|---|
   | Operating system | Debian |
   | Version | Debian GNU/Linux 12 (bookworm) |
   | Boot disk type | Balanced persistent disk |
   | Size | 10 GB |

   Click **Select**.

4. Under **Identity and API access**:
   - Service account: **Compute Engine default service account**
   - Access scopes: **Allow default access**

5. Expand **Advanced options > Security**:
   - Turn on **OS Login** (set to **Enable**)
   - Turn on **Shielded VM** options: Secure Boot, vTPM, Integrity Monitoring

6. Expand **Advanced options > Networking**:
   - Click **Add a network interface**
   - Network: `lab-vpc`
   - Subnetwork: `lab-subnet`
   - External IPv4 address: **None**
   - Network tags: `iap-ssh-target`, `http-server`

   Remove the `default` interface if it was pre-populated.

7. Under **Management > Metadata**, add:

   | Key | Value |
   |---|---|
   | `enable-oslogin` | `TRUE` |

8. Paste the following into the **Startup script** field (still under Management):

   ```bash
   #!/bin/bash
   apt-get update -y
   apt-get install -y nginx
   systemctl enable nginx
   systemctl start nginx
   echo "<h1>Zero Trust Lab — $(hostname)</h1><p>Served securely via IAP.</p>" \
     > /var/www/html/index.html
   ```

9. Click **Create** and wait for the instance status to show a green checkmark.

### 4.2 Verify the VM Has No Public IP

1. On the VM instances page, locate `nginx-server`.
2. Confirm that the **External IP** column shows `None`.

---

## Part 5 — Connect via IAP TCP Tunneling (SSH)

IAP TCP tunneling lets you SSH into the VM without opening port 22 to the internet.

### 5.1 Grant IAP Tunnel Access to Your Identity

1. Go to **Security > Identity-Aware Proxy**.
2. Click the **SSH and TCP Resources** tab.
3. Find `nginx-server` in the list and check its checkbox.
4. In the right info panel click **Add principal**.
5. Enter your Google account email address.
6. Role: **Cloud IAP > IAP-secured Tunnel User**
7. Click **Save**.

### 5.2 Open an SSH Session

1. Go back to **Compute Engine > VM instances**.
2. Click the **SSH** button next to `nginx-server`.
3. A browser-based SSH terminal will open. GCP automatically tunnels the connection through IAP.

> If SSH does not connect, wait 2 minutes for the firewall rules and IAP policy to propagate, then try again.

### 5.3 Verify Nginx is Running

Inside the SSH terminal run:

```bash
sudo systemctl status nginx
curl http://localhost
```

Expected output from `curl`:

```
<h1>Zero Trust Lab — nginx-server</h1><p>Served securely via IAP.</p>
```

Type `exit` to close the terminal.

---

## Part 6 — Access Nginx via IAP TCP Port Forwarding (HTTP)

Instead of provisioning a load balancer and SSL certificate, you will use IAP TCP forwarding to tunnel port 80 directly to your local machine through Cloud Shell. This is instant to set up and requires no certificate provisioning.

### 6.1 Grant IAP Tunnel Access (already done in Part 5)

The **IAP-secured Tunnel User** role granted in Part 5.1 covers both SSH (port 22) and HTTP (port 80) tunneling. No additional IAM changes are needed.

### 6.2 Open a Port 80 Tunnel via Cloud Shell

1. Open **Cloud Shell** by clicking the terminal icon at the top right of the console.
2. Run the following command to create a local tunnel on port 8080 that forwards to port 80 on the VM:

   ```bash
   gcloud compute start-iap-tunnel nginx-server 80 \
     --local-host-port=localhost:8080 \
     --zone=us-central1-a \
     --project=YOUR_PROJECT_ID
   ```

3. Cloud Shell will print:

   ```
   Listening on port [8080].
   ```

   Leave this terminal open — the tunnel is active as long as the command runs.

4. Click the **Web Preview** button (the eye icon at the top right of Cloud Shell) and select **Preview on port 8080**.

5. A browser tab opens and displays:

   ```
   Zero Trust Lab — nginx-server
   Served securely via IAP.
   ```

> Cloud Shell's Web Preview routes through IAP automatically. Any request that reaches the VM has already been authenticated by Google identity. No SSL certificate configuration is needed because IAP handles the TLS termination at Google's edge.

### 6.3 Test That Direct Access Is Blocked

1. Confirm the VM has no public IP: go to **Compute Engine > VM instances** and check the External IP column — it should show `None`.
2. Try navigating directly to `http://<internal-ip>` in a browser — it will time out because there is no public IP and no internet-facing port open.
3. Verify port 80 is restricted: go to **VPC network > Firewall**, confirm `allow-iap-http` only permits source `35.235.240.0/20` — no wide-open ingress.

---

## Part 7 — Test IAP SSH Tunnel from Cloud Shell

1. Open a second Cloud Shell tab (or press Ctrl+C to stop the port tunnel first).
2. Run:

   ```bash
   gcloud compute ssh nginx-server \
     --zone=us-central1-a \
     --tunnel-through-iap \
     --project=YOUR_PROJECT_ID
   ```

3. The command connects without needing a public IP or open SSH port.
4. Inside the VM, run `curl http://localhost` to confirm Nginx responds.
5. Type `exit`.

---

## Part 8 — Review Zero Trust Controls

At this point your setup enforces the following Zero Trust principles:

| Control | Implementation |
|---|---|
| Verify identity before granting access | IAP requires Google authentication for every SSH and HTTP request |
| Least privilege | IAM role `IAP-secured Tunnel User` grants only tunnel access — nothing else |
| No implicit trust from network location | VM has no public IP; firewall only allows IAP forwarder range |
| Encrypted in transit | IAP TCP tunnel is TLS-encrypted end-to-end at Google's edge |
| Device-aware access (optional extension) | IAP can be combined with Access Context Manager for device posture checks |

### 8.1 Optional — Add Access Context Manager Policy

To enforce additional context (e.g. allow only corporate devices or IPs):

1. Go to **Security > Access Context Manager**.
2. Click **New access level**.
3. Create a level based on **IP subnetworks** (your corporate egress IP) or **Device policy** (require managed devices via Endpoint Verification).
4. Attach the access level to the IAP resource via **IAP > Edit access level**.

---

## Part 9 — OS Login Verification

OS Login replaces traditional SSH key management with Google identity-based access.

1. Go to **Compute Engine > VM instances > nginx-server**.
2. Click **Edit**.
3. Under **Metadata** confirm `enable-oslogin = TRUE`.
4. Under **Security** confirm **OS Login** is enabled.

With OS Login:
- SSH keys stored in `~/.ssh/authorized_keys` on the VM are ignored.
- Access is controlled via the `roles/compute.osLogin` or `roles/compute.osAdminLogin` IAM role.
- All SSH sessions are logged to Cloud Audit Logs.

To grant OS Login access to a user:

1. Go to **IAM & Admin > IAM**.
2. Click **Grant access**.
3. Principal: the user's Google account.
4. Role: **Compute Engine > Compute OS Login** (for non-root) or **Compute OS Admin Login** (for sudo).
5. Click **Save**.

---

## Part 10 — Cleanup

To avoid ongoing charges, delete the resources created in this lab.

> The VPC (`lab-vpc`) was pre-existing and is not deleted here. Only resources created during this lab are removed.

1. **VM** — Go to **Compute Engine > VM instances**, select `nginx-server`, click **Delete**.
2. **Firewall rules** — Go to **VPC network > Firewall**, delete `allow-iap-ssh` and `allow-iap-http`.

No load balancer, backend service, instance group, static IP, or SSL certificate was created, so there is nothing further to clean up.

---

## Summary

You have completed the lab. Here is what you built:

- Verified and used a pre-existing custom VPC with Private Google Access enabled
- A Compute Engine VM running Nginx with no external IP address
- IAP TCP tunneling so SSH access requires a verified Google identity and an IAM role
- IAP TCP port forwarding on port 80 so web access requires authentication — no load balancer, no SSL certificate, no waiting on certificate provisioning
- OS Login to replace static SSH keys with Google-identity-based access control

This architecture removes the need for a VPN, eliminates standing network-level trust, and ensures every access attempt — whether SSH or HTTP — is verified against Google's identity infrastructure before it reaches your workload.

---

## Reference Links

- [Identity-Aware Proxy documentation](https://cloud.google.com/iap/docs)
- [IAP TCP forwarding overview](https://cloud.google.com/iap/docs/tcp-forwarding-overview)
- [gcloud compute start-iap-tunnel](https://cloud.google.com/sdk/gcloud/reference/compute/start-iap-tunnel)
- [OS Login overview](https://cloud.google.com/compute/docs/oslogin)
- [Access Context Manager](https://cloud.google.com/access-context-manager/docs)
- [BeyondCorp Enterprise overview](https://cloud.google.com/beyondcorp-enterprise/docs/overview)
- [VPC firewall rules](https://cloud.google.com/vpc/docs/firewalls)
