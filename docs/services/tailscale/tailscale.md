# Tailscale Quick Guide

## Overview

**[Tailscale](https://tailscale.com/)** is a zero-configuration VPN that creates a secure private network (a "Tailnet") between devices. Unlike traditional VPNs that route traffic through a central gateway, Tailscale establishes a **mesh network** where devices connect directly to each other (peer-to-peer) using the **WireGuard** protocol.

### Key Benefits

* **No Port Forwarding:** It traverses firewalls and NATs automatically. No router ports need to be opened.
* **WireGuard Based:** utilizes modern, high-performance encryption.
* **Identity Based:** Authentication is handled via existing identity providers (Google, Microsoft, GitHub) rather than managing separate VPN keys.

---

## Installation & Setup

Setting up a basic mesh network involves three steps.

### 1. Account Creation

Navigate to [tailscale.com](https://tailscale.com) and sign up using an existing Single Sign-On (SSO) provider.

### 2. Client Installation

Install the application on every device intended for the mesh (Windows, macOS, Linux, iOS, Android).

**Linux Installation Command:**

```bash
curl -fsSL https://tailscale.com/install.sh | sh

```

### 3. Authentication

Open the application on the device and log in. The device will immediately join the **Tailnet**.

**MagicDNS:**
Tailscale automatically assigns a readable domain name to every device (e.g., `server-pc` or `iphone`). Devices can be accessed via `ping` or SSH using this hostname, eliminating the need to memorize IP addresses.

---

## Advanced Features

Once connected, several configuration options are available to extend the network's utility.

| Feature | Description | Best Use Case |
| --- | --- | --- |
| **Exit Nodes** | Routes *all* internet traffic through a specific device on the network. | Securing traffic on public Wi-Fi; appearing to be at a specific location while traveling. |
| **Subnet Routers** | Allows access to LAN devices that *cannot* run Tailscale (printers, IoT, legacy servers). | Remote access to an entire home LAN without installing the client on every device. |
| **Taildrop** | A peer-to-peer file transfer tool. | Transferring files between different operating systems instantly. |

### Configuring an Exit Node (Linux)

To configure a Linux server to act as an Exit Node:

1. **Enable IP Forwarding:** Ensure the host allows packet forwarding.
2. **Advertise the Node:**
```bash
sudo tailscale up --advertise-exit-node

```


3. **Approve:** In the Admin Console (web), locate the machine, open the **...** menu > **Edit route settings**, and check "Use as exit node."

---

## Subnet Router Configuration (Pi-hole Integration)

This setup allows remote access to local LAN services using a Pi-hole for DNS resolution.

### 1. Enable IP Forwarding

Packet forwarding must be enabled at the kernel level.

```bash
# 1. Enable IPv4 forwarding immediately
sudo sysctl -w net.ipv4.ip_forward=1

# 2. Make the change permanent across reboots
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf

```

### 2. Advertise Subnet Routes

Run the following command to advertise the local subnet (replace `192.168.0.0/24` with the actual LAN subnet):

```bash
sudo tailscale up --advertise-routes=192.168.0.0/24

```

### 3. Approve Route

1. Navigate to the **Tailscale Admin Console**.
2. Go to **Machines** -> Select the Router Node -> **Edit route settings**.
3. Approve the previously advertised subnet.

### 4. DNS Resolution Flow

Once configured, the resolution flow works as follows:

1. **DNS Config:** The Pi-hole is set as the Nameserver in Tailscale settings.
2. **Query:** A remote client queries `service.domain.me`.
3. **Resolution:** The Pi-hole responds with the service's local LAN IP.
4. **Routing:** Since the subnet route is approved, the remote client traffic is automatically routed through the Tailscale tunnel to the Subnet Router, which accesses the device on the LAN.

---

## Access Control Lists (ACLs)

By default, every device in a Tailnet can communicate with every other device. **ACLs** (defined in JSON) are used to restrict this access.

**Example: Server Isolation**
This rule allows Admins to access everything, but prevents servers from initiating connections to personal devices.

```json
{
  "acls": [
    // Admins can access everything
    { "action": "accept", "src": ["group:admin"], "dst": ["*:*"] },
    
    // Servers can only talk to other servers (tagged devices)
    { "action": "accept", "src": ["tag:server"], "dst": ["tag:server:*"] }
  ]
}

```

> **Note:** Tailscale uses "Tags" (e.g., `tag:server`) to manage permissions for headless devices, rather than relying on user identities. This ensures services do not lose access if a specific user account is removed.

---

## Maintenance & Best Practices

* **Key Expiry:** By default, authentication keys expire every 180 days.
* *Recommendation:* In the Admin Console, select **"Disable Key Expiry"** for servers and headless devices to prevent remote lockouts.


* **Tailscale SSH:** Enable this feature to allow SSH access between Tailscale devices using Tailscale identity verification, removing the need to manage local SSH keys.
* **Battery Life:** While efficient, toggling Tailscale off on mobile devices when not accessing home services can conserve battery life.