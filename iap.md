# GCP Lab: Nginx VM with Identity-Aware Proxy and Zero Trust Networking

**Level:** Intermediate  
**Duration:** 90 minutes  
**Prerequisites:** A GCP project with billing enabled, Owner or Editor IAM role  

---

## Overview

In this lab you will deploy a Compute Engine VM running Nginx, lock down all direct internet access, and expose the web server securely through Identity-Aware Proxy (IAP). By the end you will have a working Zero Trust setup where access to the VM is gated by Google identity — no VPN, no public SSH port, no open firewall rules.

**What you will build:**

- A VPC network with no default internet ingress
- A Compute Engine VM running Nginx (no public IP)
- Firewall rules that allow only IAP traffic
- IAP TCP tunneling for SSH access
- IAP HTTPS protection for the web application
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
  Internal Load Balancer  <-- HTTPS traffic (port 443)
        |
        v
  VM: nginx-server        <-- No public IP
  (us-central1-a)
        |
  VPC: lab-vpc
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

## Part 2 — Create a Custom VPC Network

### 2.1 Create the VPC

1. In the left navigation menu go to **VPC network > VPC networks**.
2. Click **Create VPC network**.
3. Fill in the following fields:

   | Field | Value |
   |---|---|
   | Name | `lab-vpc` |
   | Description | `Zero Trust lab network` |
   | Subnet creation mode | Custom |

4. In the **New subnet** section fill in:

   | Field | Value |
   |---|---|
   | Name | `lab-subnet` |
   | Region | `us-central1` |
   | IP address range | `10.0.0.0/24` |
   | Private Google Access | On |
   | Flow logs | Off |

5. Under **Dynamic routing mode** select **Regional**.
6. Click **Create** and wait for the VPC to appear in the list.

### 2.2 Delete the Default Routes (optional hardening)

> Skip this section if you are using a shared or production project. It only applies if you created a fresh VPC and want strict egress control.

1. Go to **VPC network > Routes**.
2. Filter by network `lab-vpc`.
3. The default route to `0.0.0.0/0` via `default-internet-gateway` will not exist because you created a custom-mode VPC — this is expected and correct.

---

## Part 3 — Configure Firewall Rules

You will create two firewall rules: one that allows IAP to reach the VM on SSH, and one that allows the load balancer health checker and IAP traffic on port 80.

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

### 3.2 Allow IAP and Load Balancer for HTTP (port 80)

1. Click **Create firewall rule** again.
2. Fill in:

   | Field | Value |
   |---|---|
   | Name | `allow-iap-lb-http` |
   | Network | `lab-vpc` |
   | Priority | `1000` |
   | Direction of traffic | Ingress |
   | Action on match | Allow |
   | Targets | Specified target tags |
   | Target tags | `http-server` |
   | Source filter | IPv4 ranges |
   | Source IPv4 ranges | `35.235.240.0/20,130.211.0.0/22,35.191.0.0/16` |
   | Protocols and ports | TCP: `80` |

3. Click **Create**.

> The additional ranges (`130.211.0.0/22` and `35.191.0.0/16`) are Google Cloud load balancer health checker ranges.

### 3.3 Deny All Other Ingress (verify default)

Custom VPC networks automatically have an implied deny-all ingress rule at priority 65535. Verify it exists:

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

## Part 6 — Expose Nginx via IAP (HTTPS)

To protect the web application with IAP you need an HTTPS load balancer with an IAP-enabled backend. Compute Engine backends require a backend service backed by an instance group.

### 6.1 Create an Instance Group

1. Go to **Compute Engine > Instance groups**.
2. Click **Create instance group**.
3. Select **New unmanaged instance group**.
4. Fill in:

   | Field | Value |
   |---|---|
   | Name | `nginx-ig` |
   | Location | Single zone |
   | Region | `us-central1` |
   | Zone | `us-central1-a` |
   | Network | `lab-vpc` |
   | Subnetwork | `lab-subnet` |

5. Under **VM instances** click **Select instances** and add `nginx-server`.
6. Click **Create**.

### 6.2 Reserve a Static External IP

1. Go to **VPC network > IP addresses**.
2. Click **Reserve external static address**.
3. Fill in:

   | Field | Value |
   |---|---|
   | Name | `lab-lb-ip` |
   | Network service tier | Premium |
   | IP version | IPv4 |
   | Type | Global |

4. Click **Reserve** and note the assigned IP address.

### 6.3 Create an HTTPS Load Balancer

1. Go to **Network services > Load balancing**.
2. Click **Create load balancer**.
3. Under **HTTP(S) Load Balancing** click **Start configuration**.
4. Select:
   - **From Internet to my VMs or serverless services**
   - **Global HTTP(S) Load Balancer (classic)**
5. Click **Continue**.
6. Name: `lab-lb`

**Backend configuration:**

7. Click **Backend configuration > Create a backend service**.
8. Fill in:

   | Field | Value |
   |---|---|
   | Name | `nginx-backend` |
   | Backend type | Instance group |
   | Protocol | HTTP |
   | Named port | `http` |

9. Under **Backends** click **Add backend**:
   - Instance group: `nginx-ig`
   - Port numbers: `80`
   - Balancing mode: Utilization
   - Maximum backend utilization: `80`

10. Under **Health check** click **Create a health check**:
    - Name: `nginx-health-check`
    - Protocol: HTTP
    - Port: `80`
    - Request path: `/`
    - Click **Save**.

11. Leave other defaults and click **Create**.

**Frontend configuration:**

12. Click **Frontend configuration**.
13. Fill in:

    | Field | Value |
    |---|---|
    | Name | `lab-frontend` |
    | Protocol | HTTPS |
    | Network service tier | Premium |
    | IP address | `lab-lb-ip` (the one you reserved) |
    | Port | `443` |

14. Under **Certificate** click **Create a new certificate**:
    - Name: `lab-cert`
    - Mode: **Google-managed certificate**
    - Domains: enter a domain you own, or use `<YOUR_IP>.nip.io` as a test domain (replace `<YOUR_IP>` with dots replaced by dashes, e.g. `34-120-1-5.nip.io`)
    - Click **Create**.

15. Click **Done**, then click **Review and finalize**, then **Create**.

> The load balancer may take 5-10 minutes to provision. The SSL certificate can take up to 15 minutes to activate if using a Google-managed cert with a real domain.

### 6.4 Enable IAP on the Backend

1. Go to **Security > Identity-Aware Proxy**.
2. Click the **Web** tab.
3. Find `nginx-backend` in the list.
4. Toggle the IAP switch to **On**.
5. In the confirmation dialog click **Turn on**.

### 6.5 Grant IAP Access to Your Identity

1. Check the checkbox next to `nginx-backend`.
2. In the right panel click **Add principal**.
3. Enter your Google account email.
4. Role: **Cloud IAP > IAP-secured Web App User**
5. Click **Save**.

---

## Part 7 — Test End-to-End Zero Trust Access

### 7.1 Test IAP Web Access

1. Wait for the load balancer to finish provisioning (check **Network services > Load balancing** — the status should show a green checkmark).
2. Open a browser and navigate to `https://<YOUR_DOMAIN_OR_NIP_IO>`.
3. Google's IAP login screen will appear. Sign in with the Google account you granted access to.
4. After successful authentication you will see:

   ```
   Zero Trust Lab — nginx-server
   Served securely via IAP.
   ```

### 7.2 Test That Direct Access Is Blocked

1. Try visiting `http://<lb-ip>` directly without HTTPS — it should fail or redirect.
2. Confirm that the VM has no public IP by going to **Compute Engine > VM instances** and checking the External IP column.
3. Confirm that no SSH port is open to the internet by going to **VPC network > Firewall** and verifying that port 22 is only allowed from `35.235.240.0/20`.

### 7.3 Test IAP TCP Tunnel from Cloud Shell

1. Open **Cloud Shell** by clicking the terminal icon at the top right of the console.
2. Run:

   ```bash
   gcloud compute ssh nginx-server \
     --zone=us-central1-a \
     --tunnel-through-iap \
     --project=YOUR_PROJECT_ID
   ```

3. The command should connect without needing a public IP or open SSH port.
4. Inside the VM, run `curl http://localhost` to confirm Nginx responds.
5. Type `exit`.

---

## Part 8 — Review Zero Trust Controls

At this point your setup enforces the following Zero Trust principles:

| Control | Implementation |
|---|---|
| Verify identity before granting access | IAP requires Google authentication for every request |
| Least privilege | IAM roles `IAP-secured Web App User` and `IAP-secured Tunnel User` grant only what is needed |
| No implicit trust from network location | VM has no public IP; firewall only allows IAP forwarder range |
| Encrypted in transit | All traffic goes through HTTPS; IAP tunnel encrypts SSH |
| Device-aware access (optional extension) | IAP can be combined with Access Context Manager for device posture checks |

### 8.1 Optional — Add Access Context Manager Policy

To enforce additional context (e.g. allow only corporate devices):

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

1. **Load balancer** — Go to **Network services > Load balancing**, select `lab-lb`, click **Delete**.
2. **Backend service** — Deleted with the load balancer, but verify under **Backend services**.
3. **Instance group** — Go to **Compute Engine > Instance groups**, delete `nginx-ig`.
4. **VM** — Go to **Compute Engine > VM instances**, select `nginx-server`, click **Delete**.
5. **Static IP** — Go to **VPC network > IP addresses**, release `lab-lb-ip`.
6. **Firewall rules** — Go to **VPC network > Firewall**, delete `allow-iap-ssh` and `allow-iap-lb-http`.
7. **VPC** — Go to **VPC network > VPC networks**, delete `lab-vpc` (this also deletes `lab-subnet`).
8. **SSL certificate** — Go to **Network services > Load balancing > Certificates**, delete `lab-cert`.

---

## Summary

You have completed the lab. Here is what you built:

- A custom VPC with no internet ingress except through IAP forwarder IPs
- A Compute Engine VM running Nginx with no external IP address
- IAP TCP tunneling so SSH access requires a verified Google identity and an IAM role
- An HTTPS load balancer with IAP enabled so web access requires authentication
- OS Login to replace static SSH keys with Google-identity-based access control

This architecture removes the need for a VPN, eliminates standing network-level trust, and ensures every access attempt — whether SSH or HTTP — is verified against Google's identity infrastructure before it reaches your workload.

---

## Reference Links

- [Identity-Aware Proxy documentation](https://cloud.google.com/iap/docs)
- [IAP TCP forwarding](https://cloud.google.com/iap/docs/tcp-forwarding-overview)
- [OS Login overview](https://cloud.google.com/compute/docs/oslogin)
- [Access Context Manager](https://cloud.google.com/access-context-manager/docs)
- [BeyondCorp Enterprise overview](https://cloud.google.com/beyondcorp-enterprise/docs/overview)
- [VPC firewall rules](https://cloud.google.com/vpc/docs/firewalls)
- [Google Cloud load balancing](https://cloud.google.com/load-balancing/docs/https)
