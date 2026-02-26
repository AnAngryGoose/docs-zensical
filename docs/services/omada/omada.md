# TP-Link Omada Controller

## Overview

The **Omada Software Defined Networking (SDN)** platform is TP-Link's centralized management solution. It manages all your TP-Link hardwaer from a single interface.

I have not gotten too into the deeper settings of omada, but this is basics from what I did. 


---

## Installation

### 1. Volume Creation

The `compose.yaml` configuration defines the storage volumes as `external: true`. This means Docker will **not** create them automatically. You must create them manually before launching the container, or the stack will fail to start.

```bash
docker volume create omada_data
docker volume create omada_logs
docker volume create omada_work

```

### 2. Docker Compose

This setup uses the community-standard `mbentley/omada-controller` image. The container is configured with `network_mode: host` to ensure the controller can easily discover devices on the local network via Layer 2 broadcasts.

**`compose.yaml`**

```yaml
### Omada Controller - TP-Link / Omada network controller ###

services:
  omada-controller:
    image: mbentley/omada-controller:latest
    container_name: omada-controller
    restart: always
    network_mode: host
    environment:
      - MANAGE_HTTP_PORT=8088
      - MANAGE_HTTPS_PORT=8043
      - PORTAL_HTTP_PORT=8088
      - PORTAL_HTTPS_PORT=8843
      - SHOW_SERVER_LOGS=true
      - SHOW_MONGODB_LOGS=false
      - TZ=America/Chicago
    # networks: 
    #   - proxy_net  
    volumes:   
      - omada_data:/opt/tplink/EAPController/data
      - omada_logs:/opt/tplink/EAPController/logs
      - omada_work:/opt/tplink/EAPController/work

volumes:
  omada_data:
    external: true
  omada_logs:
    external: true
  omada_work:
    external: true

# # --- Network Definitions --- #
# networks: 
#   proxy_net: 
#     external: true

```

Start the controller:

```bash
docker compose up -d

```

---

## Initial Setup & Adoption

Once the controller is running, follow this workflow to bring the network online.

### 1. Access the Interface

* **URL:** `https://<server-ip>:8043`
* **Note:** You may receive a security warning because the SSL certificate is self-signed. This is normal; proceed past it.
* **First Login:** The default is often `admin` / `admin`, but the setup wizard will force a password change immediately.

### 2. Device Adoption

The controller does not automatically manage devices plugged into the network; they must be explicitly "Adopted".

1. Go to the **Devices** tab (Access Point icon).
2. Devices (Gateway, Switches, APs) should be listed as **"Pending"**.
3. Click the **Adopt** button on the right.
4. The status will cycle: **Provisioning** -> **Configuring** -> **Connected**.

### 3. WAN/LAN Configuration

If using an Omada Gateway (like the ER605):

* Navigate to **Settings (Gear icon) > Wired Networks > Internet**.
* Configure the WAN type provided by the ISP (DHCP, PPPoE, or Static).

---

## Configuration Features

### Wireless Networks (SSIDs)

* **Location:** Settings > Wireless Networks.
* **Usage:** Create multiple SSIDs here (e.g., Main, IoT, Guest).
* **Guest Network:** Checking the "Guest Network" box automatically applies isolation rules, preventing those clients from accessing the internal LAN.

### VLANs (Virtual LANs)

To segment traffic (e.g., separating IoT devices from the main PC):

1. **Create Interface:** Settings > Wired Networks > LAN. Create a new Interface with a VLAN ID (e.g., `20`).
2. **Tag Ports:** Ensure managed switches utilize the correct profiles to pass this VLAN tag to the APs.
3. **Assign to Wi-Fi:** In Wireless settings, assign an SSID to use that specific VLAN ID.

### ACLs (Access Control Lists)

* **Location:** Settings > Network Security > ACL.
* **Usage:** Use "Switch ACLs" or "Gateway ACLs" to block traffic between VLANs (e.g., Block the IoT VLAN from accessing the Main VLAN).

---

## Maintenance

* **Backups:** Go to **Settings > Maintenance > Backup & Restore**. Set up "Auto Backup" to save the config locally. Since the `omada_data` volume is mapped, these backups persist on the disk.
* **Roaming:** Enable **Fast Roaming (802.11r)** in Site Settings to help devices switch between APs smoothly. *Note: Test this first, as some older IoT devices struggle with it.*
* **AI Optimization:** The "AI WLAN Optimization" tool scans the RF environment and automatically adjusts channels and transmit power to reduce interference. Run this once after the initial deployment.

---
