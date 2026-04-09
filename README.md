# Self-Hosting Nextcloud on Ubuntu with Snap, External Storage, and Tailscale

A complete guide to deploying your own private cloud storage using [Nextcloud](https://nextcloud.com/) on an Ubuntu home server — with dedicated external storage and secure remote access through [Tailscale](https://tailscale.com/).

## Why Self-Host?

- **Full ownership** of your data — no third-party cloud provider involved.
- **No subscription fees** — use hardware you already own.
- **Private access anywhere** — Tailscale creates a secure WireGuard tunnel without exposing ports to the internet.

## What This Guide Covers

| Component | Purpose |
|-----------|---------|
| **Snap** | Simple, containerized Nextcloud installation with automatic updates |
| **External Storage** | Dedicated disk (USB/HDD/SSD) mounted as Nextcloud storage |
| **Tailscale** | Secure HTTPS access to Nextcloud from any device on your tailnet |

---

## Prerequisites

- An Ubuntu/Debian server (physical or VM) with `sudo` access
- A spare disk or USB drive for Nextcloud data storage
- A [Tailscale](https://tailscale.com/) account (free tier works)
- Tailscale installed on the server (`curl -fsSL https://tailscale.com/install.sh | sh`)

---

## Step 1 — Install Nextcloud via Snap

Snap packages Nextcloud with all its dependencies (Apache, PHP, MySQL) into a single, self-contained install.

```bash
# Install the snap daemon (if not already present)
sudo apt update
sudo apt install snapd

# Install Nextcloud
sudo snap install nextcloud
```

### Configure Ports

By default, the Nextcloud snap listens on ports 80 and 443. If those are already in use, change them:

```bash
sudo snap set nextcloud ports.http=8080
sudo snap set nextcloud ports.https=8443
```

### Verify the Installation

```bash
sudo snap services
```

You should see services like `nextcloud.apache`, `nextcloud.mysql`, `nextcloud.php-fpm`, etc., all marked as **active**.

---

## Step 2 — Initial Web Setup

Open a browser and navigate to:

```
http://<YOUR-SERVER-IP>:8080
```

You will be prompted to create an **admin account**. Choose a strong username and password — this is the administrator for your Nextcloud instance.

---

## Step 3 — Attach External Storage

Using a dedicated disk keeps your Nextcloud data separate from the OS, making backups and migrations straightforward.

### 3.1 — Identify the Disk

```bash
lsblk
```

Locate your target disk (e.g., `/dev/sdb`). Make sure it is the correct device before proceeding.

### 3.2 — Format the Disk

> **Warning:** This will erase all data on the target partition. Back up any existing data first.

```bash
# Unmount if already mounted
sudo umount /mnt/nextcloud

# Format with ext4 and label it
sudo mkfs.ext4 -L "NextcloudStorage" /dev/<YOUR_TARGET_DISK>
```

### 3.3 — Mount the Disk

```bash
# Create the mount point (if it doesn't exist)
sudo mkdir -p /mnt/nextcloud

# Mount the partition
sudo mount /dev/<YOUR_TARGET_DISK> /mnt/nextcloud
```

### 3.4 — Set Permissions

```bash
sudo chown -R <YOUR_USER>:<YOUR_USER> /mnt/nextcloud
sudo chmod -R 777 /mnt/nextcloud
```

### 3.5 — Persist the Mount Across Reboots

Find the UUID of your partition:

```bash
lsblk -f
```

Edit `/etc/fstab` and add the following line (replace the UUID with yours):

```
UUID=<YOUR-DISK-UUID>  /mnt/nextcloud  ext4  defaults,noatime  0  1
```

This ensures the disk is automatically mounted every time the server boots.

### 3.6 — Connect the Snap Removable Media Interface

The Nextcloud snap is sandboxed by default. You must explicitly grant it access to removable media:

```bash
sudo snap connect nextcloud:removable-media
sudo snap restart nextcloud
```

---

## Step 4 — Configure External Storage in Nextcloud

Now that the disk is mounted and accessible to the snap, configure Nextcloud to use it.

1. Log in to the Nextcloud web UI as your admin user.
2. Go to **Profile icon** (top right) → **Apps**.
3. Search for and **enable** the **External Storage Support** app.
4. Go to **Profile icon** → **Administration Settings** → **External storage** (under the Administration section).
5. Add a new storage with these settings:

   | Field | Value |
   |-------|-------|
   | Folder name | `External Drive` (or any name you prefer) |
   | External storage | `Local` |
   | Configuration | `/mnt/nextcloud` |
   | Available for | All users (or select specific users/groups) |

6. Click the **checkmark** to save. A green icon confirms Nextcloud can access the path.

---

## Step 5 — Secure Remote Access with Tailscale

Instead of exposing Nextcloud directly to the internet, use Tailscale to create a private, encrypted tunnel accessible only to your devices.

### 5.1 — Configure Trusted Proxy

Tell Nextcloud to trust the local Tailscale reverse proxy:

```bash
sudo snap run nextcloud.occ config:system:set trusted_proxies 0 --value=127.0.0.1
```

### 5.2 — Force HTTPS Protocol

Since Tailscale will handle TLS termination, tell Nextcloud to generate HTTPS URLs:

```bash
sudo snap run nextcloud.occ config:system:set overwriteprotocol --value=https
```

### 5.3 — Add Tailscale Domain as Trusted

Replace `<YOUR-TAILSCALE-DNS>` with your server's Tailscale MagicDNS hostname (e.g., `my-server.tail12345.ts.net`):

```bash
sudo snap run nextcloud.occ config:system:set trusted_domains 1 --value="<YOUR-TAILSCALE-DNS>"
```

### 5.4 — Start the Tailscale HTTPS Tunnel

```bash
sudo tailscale serve --bg --https=443 http://localhost:8080
```

This tells Tailscale to serve your local Nextcloud instance over HTTPS on port 443, with a valid TLS certificate provisioned automatically by Tailscale.

Notes: in this case your local Nextcloud instance runs on port 8080

---

## Step 6 — Verify Everything

### Check All Nextcloud Snap Services

```bash
systemctl status snap.nextcloud.*
```

All services should be **active (running)**.

### Access Nextcloud

Open a browser on any device connected to your tailnet and navigate to:

```
https://<YOUR-TAILSCALE-DNS>
```

You should see the Nextcloud login page served over a valid HTTPS connection.

---

## Step 7 — Connect Client Devices

On each device you want to access Nextcloud from, make sure Tailscale is installed and run:

```bash
sudo tailscale up --accept-dns --exit-node-allow-lan-access --exit-node=<YOUR-EXIT-NODE>
```

| Flag | Purpose |
|------|---------|
| `--accept-dns` | Use Tailscale's MagicDNS so you can reach your server by hostname |
| `--exit-node-allow-lan-access` | Allow access to local network resources while using the exit node (optional) |
| `--exit-node` | Route traffic through your home server (optional — only if you want to use it as a VPN exit node) |

You can also install the [Nextcloud desktop or mobile client](https://nextcloud.com/install/#install-clients) and point it at `https://<YOUR-TAILSCALE-DNS>` for automatic file sync.

---

## Troubleshooting

### View Nextcloud Application Logs (Live)

```bash
sudo nextcloud.occ log:watch
```

### View Snap Logs

```bash
# General Nextcloud snap logs
sudo snap logs nextcloud

# Last 50 log entries
sudo snap logs nextcloud -n 50

# Apache-specific logs (last 50 entries)
sudo snap logs nextcloud.apache -n 50

# Follow Apache logs in real time
sudo snap logs nextcloud.apache -f
```

---

## Architecture Overview

```
┌──────────────┐       Tailscale        ┌──────────────────────────────────┐
│ Your Devices │◄──── WireGuard ────►   │         Home Server              │
│ (Phone, PC)  │       Tunnel           │                                  │
└──────────────┘                        │  ┌─────────────┐                 │
                                        │  │  Tailscale  │ :443 (HTTPS)    │
                                        │  │  serve      │────┐            │
                                        │  └─────────────┘    │            │
                                        │                     ▼            │
                                        │  ┌─────────────────────────┐     │
                                        │  │   Nextcloud (Snap)      │     │
                                        │  │   Apache :8080          │     │
                                        │  │   PHP-FPM               │     │
                                        │  │   MySQL                 │     │
                                        │  └────────┬────────────────┘     │
                                        │           │                      │
                                        │           ▼                      │
                                        │  ┌─────────────────┐             │
                                        │  │  /mnt/nextcloud │             │
                                        │  │  (External Disk)│             │
                                        │  └─────────────────┘             │
                                        └──────────────────────────────────┘
```

---

## References

- [Nextcloud Snap GitHub](https://github.com/nextcloud-snap/nextcloud-snap)
- [Nextcloud Documentation](https://docs.nextcloud.com/)
- [Tailscale Serve & Funnel](https://tailscale.com/kb/1242/tailscale-serve)
- [Tailscale MagicDNS](https://tailscale.com/kb/1081/magicdns)
