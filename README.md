# Table of Contents
1. [Introduction](#introduction)
2. [Tech Overview](#tech-overview)
3. [Setup](#setup)
4. [To Do](#to-do)


# Introduction

The main motivation for building this DIY NAS in the first place came out of necessity due to a few issues faced with my family data:
- My family had lost precious documents and photos previously due to failing internal hard drives.
- Family photos were scattered across multiple devices, often duplicated between members, with no central place to manage them — which made something as simple as selecting photos to print unnecessarily painful.

This DIY NAS (nicknamed **PiNAS**) was built to solve both of these problems:

- **Centralised shared storage** — a single location for family data, especially photos and documents, accessible by every family member from any device on the home network or remotely.
- **Foundation for a proper backup strategy** — a NAS alone is not a backup, but this setup is the first building block toward a full backup plan (see [To Do](#to-do)).

### Why not just use cloud services or buy an off-the-shelf NAS?
1. The cloud is NOT a backup, it is just someone else's machine, so I would prefer to have full ownership over the hardware and data, plus these services could increase prices at any time.
2. Keep costs low by utilizing hardware I already have at my disposal (Raspberry Pi, ethernet cables, and a beefy 4TB 3.5" HDD). For anyone planning to follow this setup guide, instead of following my hardware setup exactly, try looking around your home for any unused devices like mini PCs, Single Board Computers (SBC) or even old laptops.
3. Learn how self-hosted infrastructure works in the process.

For anyone wanting to extend this setup or go hardcore on homelabbing, [this](https://wiki.futo.org/index.php/Introduction_to_a_Self_Managed_Life:_a_13_hour_%26_28_minute_presentation_by_FUTO_software) is a great resource.


# Tech Overview

### Hardware

| Component | Details |
|---|---|
| **SBC (main board)** | Raspberry Pi (acts as the NAS server) |
| **Storage drive** | External HDD connected via USB |
| **Network** | Connected via Ethernet (with Wi-Fi as fallback) |
| **Client devices tested** | Windows 10, Windows 11, Android |

### Software & Features

| Feature | Implementation |
|---|---|
| **OS** | Raspberry Pi OS (Lite), flashed with Raspberry Pi Imager |
| **NAS software** | OpenMediaVault 7 (OMV7) — web GUI for NAS management |
| **File system** | EXT4 |
| **Network file sharing** | Samba (SMB/CIFS) — compatible with Windows, Android, iOS |
| **Remote access (VPN)** | WireGuard (primary) or Tailscale (alternative) |
| **Container platform** | Docker (managed via OMV compose plugin) |
| **Firewall** | UFW (Uncomplicated Firewall) at OS level |
| **Reverse proxy** | NGINX Proxy Manager (Docker container) |
| **Intrusion protection** | Fail2ban (Docker container, monitors NGINX logs) |
| **Drive health monitoring** | OMV built-in S.M.A.R.T. tests and system stats |
| **Email notifications** | Configured via SMTP/Gmail for system alerts |
| **SSH hardening** | Public key authentication, root login disabled, non-standard port |

### Key Capabilities

- **Shared family folder** (`PiNAS_share`) — accessible by all authorised users on the home network or over VPN.
- **Per-user home directories** — each family member has a private directory visible only to them.
- **Recycle bin** — enabled on both the shared folder and SMB share to prevent accidental permanent deletion.
- **Access control** — guest access disabled; only named user accounts can connect.
- **Remote access over VPN** — WireGuard tunnel allows secure access to the NAS from outside the home network, including from smartphones.
- **Docker-ready** — containerised services can be added alongside NAS functionality on the same device.


# Setup

> **Prerequisites:** Raspberry Pi board, microSD card (for OS), external USB hard drive, a PC on the same network, access to your home router admin page.

### Step 1 — Flash the OS

1. Download [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
2. In Imager customisation settings (before flashing), set a **username and password** for the main admin user (this will not be root).
3. Enable **SSH** in the customisation settings.
4. Flash Raspberry Pi OS Lite to the microSD card.

### Step 2 — First Boot & Find the Device IP

1. Insert the microSD, connect the Pi to your router via Ethernet, and power it on.
2. On your PC (connected to the same network), open a browser and navigate to your **router's gateway IP** (usually printed on the back of your router).
3. Log in to the router admin page and navigate to the **DHCP server / client list** section.
4. Find your Pi's hostname in the list and note its assigned IP address.

> **Tip:** If you cannot find the DHCP IP, run `arp -a` in Windows CMD or check your router's connected devices list.

### Step 3 — SSH Into the Pi & Update

```bash
# From Windows: use PuTTY or Windows Terminal
ssh <your_username>@<pi_ip_address>

# Once logged in, update the system
sudo apt update && sudo apt upgrade -y
```

> **Troubleshooting SSH access denied:** Verify SSH is enabled in OMV GUI (Services → SSH) and that your user is in the `_ssh` group (Users → edit user → Groups → check `_ssh`).

### Step 4 — Install OMV7

Follow the [official OMV7 install guide for Raspberry Pi](https://wiki.omv-extras.org/doku.php?id=omv7:raspberry_pi_install). Once installed, access the OMV web GUI at `http://<pi_ip_address>`.

### Step 5 — OMV Initial Configuration

1. **Enable system monitoring:** System → Monitoring → Enable.
2. **Set up email notifications:** System → Notification → configure SMTP with a Gmail account (refer to [OMV docs](https://docs.openmediavault.org/en/5.x/administration/general/notifications.html#gmail)).

### Step 6 — Set a Static IP

Setting a static IP ensures your NAS is always reachable at the same address and supports stable firewall rules.

**Via OMV GUI:**
1. Run `ipconfig` (Windows) on your PC to note your gateway, subnet, and current IP range.
2. OMV GUI: Network → Interfaces → Create (+) → select Ethernet (or Wi-Fi).
3. Set Method to **Static**, enter your chosen IP, subnet, and gateway.
4. Set IPv6 Method to **Disabled**.
5. Advanced Settings → DNS Servers: `1.1.1.1` (Cloudflare).
6. Save → Apply → Yes.

**Optional — also reserve the IP on the router:**
In your router's LAN settings, create a static IP reservation mapping the Pi's MAC address (get it via `ifconfig` over SSH — the `ether` field) to your chosen IP. This prevents DHCP conflicts with other devices.

### Step 7 — Prepare the Storage Drive

1. **Wipe the drive:** Storage → Disks → select the drive → Wipe (eraser icon) → confirm → Quick.
2. **Create & mount file system:** Storage → File Systems → Create (+) → EXT4 → select the wiped drive → Save → wait for formatting → Close → select the new file system → set usage warning threshold to **75%** → Apply → Yes.

### Step 8 — Create the Shared Folder

1. Storage → Shared Folders → Create (+).
2. Set a name (e.g., `PiNAS_share`), select the file system from Step 7.
3. Ensure the path ends with `/`.
4. Permissions: Everyone — read/write (will be tightened via Samba settings).
5. Save → Apply → Yes.

> **Note:** Shared folders created here are not yet visible to network clients. That happens in the next step.

### Step 9 — Enable Samba (Network File Sharing)

**Enable Samba:**
Services → SMB/CIFS → Settings → Enable → Save → Apply → Yes.

**Create the Samba share:**
Services → SMB/CIFS → Shares → Create (+):
- Shared folder: select `PiNAS_share`
- Public: **No** (guests not allowed)
- Check: Inherit Permissions, Extended Attributes, Store DOS Attributes
- Save → Apply → Yes.

**Verify from a Windows client:** Open File Explorer → Network → your Pi's hostname or IP should appear, and the share should be browseable after entering credentials.

> **Windows 10 tip:** If the share is not visible, refer to the [OMV forum guide for Windows 10 SMB connectivity](https://forum.openmediavault.org/index.php?Thread/27179).

### Step 10 — Set Up User Accounts

1. Users → Users → Create (+) a user for each family member.
2. Assign users to appropriate groups.
3. For Samba access: each user will authenticate with their OMV credentials.

### Step 11 — Set Up Per-User Home Directories

Per-user home directories let each family member have a private space alongside the shared folder.

1. Users → Settings → Enable home directories, select `PiNAS_share` as the location.
2. Manually create a folder for each user inside `PiNAS_share` via SSH: `sudo mkdir /srv/<disk>/PiNAS_share/<username>`
3. Services → SMB/CIFS → Settings → enable **Home Directories** and enable **Recycle Bin**.
4. **Do not** create a separate Samba share for the home directories parent folder — this keeps users from seeing each other's directories.

**Result — what users see when connecting:**
- The shared `PiNAS_share` folder (for common family data)
- Their own home directory (private)

> **Cleaning up deleted user directories:** If a deleted user's folder persists, manually remove it: `sudo rm -r /srv/<disk>/PiNAS_share/<deleted_username>`. Also check `/etc/passwd` and `/etc/shadow` via `sudo nano` and remove the deleted user's entry if it still exists.

### Step 12 — Remote Access via WireGuard VPN

Remote access is handled via a WireGuard VPN tunnel, which avoids exposing NAS services directly to the internet.

**Server side (on the Pi):**

1. Install the omv-extras plugin if not already present: System → Plugins → install `openmediavault-omvextrasorg`.
2. Install `openmediavault-wireguard`.
3. Services → WireGuard → Tunnels → Create a tunnel, set a listening port.
4. On your router: set up **port forwarding** for the WireGuard UDP port to the Pi's static IP.
5. Configure clients in Services → WireGuard → Clients.

**Client side:**

- **PC (Windows/Linux/macOS):** Install the WireGuard app, import the client config generated in OMV, activate the tunnel.
  - Enable network discovery and file/printer sharing on public networks: Settings → Advanced Network Settings → Advanced Sharing Settings.
- **Android/iOS:** Install the WireGuard app and import the config (via QR code from OMV or manual import). Use [Cx File Explorer](https://play.google.com/store/apps/details?id=com.cxinventor.file.explorer) to browse the NAS over VPN.

> **Alternative:** Tailscale VPN is simpler to set up and works without port forwarding if WireGuard's firewall configuration is too complex.

> **Firewall tip:** When testing a new VPN-related service, temporarily set the "block all" firewall rule to Accept to rule out firewall interference. Only re-enable "block all" once the service is confirmed working.

### Step 13 — Security Hardening

| Measure | How |
|---|---|
| **Disable SSH root login** | OMV GUI: Services → SSH → uncheck "Permit root login" |
| **SSH public key auth** | Generate a key pair (e.g., with PuTTYgen), add public key to the Pi, disable password auth in SSH config |
| **Change SSH port** | OMV GUI: Services → SSH → Port → set a non-standard port (1024–65535, avoid common ones); update your SSH client config accordingly |
| **Change WireGuard port** | Update router port forwarding, OMV tunnel port, and all client configs |
| **Enable UFW firewall** | Follow the [OMV firewall hardening guide](https://forum.openmediavault.org/index.php?thread/51800-basics-for-firewall-hardening/); allow only required ports (e.g., 22/SSH, 445/Samba, 5353/mDNS, WireGuard port) |
| **Install Fail2ban** | Run as a Docker container; configure it to monitor NGINX Proxy Manager access logs for repeated failed logins |
| **NGINX Proxy Manager** | Run as a Docker container for reverse proxy and SSL certificate management |
| **Keep system updated** | `sudo apt update && sudo apt upgrade` regularly; update Docker images |
| **Use strong passwords** | Unique, complex passwords for OMV admin, each user account, and the router |
| **Least privilege** | Run Docker containers as non-root; keep file permissions tight |

### Step 14 — Docker Setup

1. Install the OMV Compose plugin (via OMV Extras).
2. OMV GUI: Services → Compose → Files → Create (+).
3. Paste the `docker-compose.yml` content for the desired service, name the compose file to match the service, Save → Up.

> To fully remove a container: stop it, remove it, and delete its associated volumes and config files. Refer to your Docker cleanup notes for the exact steps.

### Step 15 — Drive Health Monitoring

OMV GUI: Storage → S.M.A.R.T. → enable and schedule regular self-tests on the drive. Enable system stats monitoring under System → Monitoring to track CPU, RAM, and disk usage over time.


# To Do

### 1. Offsite Backup (Second NAS Device)
A single NAS is **not a backup** — it is a single point of failure. The plan is to set up a second similar device (Raspberry Pi + OMV7 + HDD) at a separate physical location (offsite). Data from the primary NAS will be replicated to the offsite device, following the **3-2-1 backup rule**:
- 3 copies of data
- 2 different storage media
- 1 offsite copy

This protects against theft, fire, or any physical disaster at home.

### 2. Automatic Device Backup to NAS
Currently, family members must manually copy files to the NAS. The plan is to configure **automatic backup of chosen folders** (e.g., phone camera rolls, desktop documents) from each family member's devices to their respective NAS directories. This reduces friction and ensures photos and documents are always protected without requiring manual effort.

### 3. Storage Quota Per User/Group
To prevent personal data from consuming space reserved for Docker services, implement a storage quota for the `family` user group via OMV's file system quota feature. This is partly configured but not yet verified as working — needs further testing.

### 4. Resolve WireGuard + Firewall Conflict
WireGuard VPN currently only works when the UFW "block all" rule is disabled. The correct firewall rules to allow WireGuard traffic while keeping the firewall active need to be identified and applied.

### 5. Docker Services to Add
The following self-hosted services are planned for deployment as Docker containers:

| Service | Purpose |
|---|---|
| Immich | Photo and video library (self-hosted Google Photos alternative) |
| Nextcloud | File sync and collaboration |
| Jellyfin / Plex | Media server for home video/music streaming |
| Bitwarden | Self-hosted password manager |
| Pi-hole | Network-wide ad blocker for all home devices |
| Home Assistant | Smart home controller and automation hub |
| Syncthing | Continuous file synchronisation between devices |
| n8n | Automation workflow builder |
| Frigate | Network video recorder for security cameras |

### 6. SSL Certificates for OMV Web GUI
Set up TLS/SSL so the OMV admin GUI is served over HTTPS, protecting admin credentials from being transmitted in plaintext on the local network.

### 7. Fail2ban Email Notifications
Fail2ban is installed but email alerting for banning events needs to be fully configured and tested.
