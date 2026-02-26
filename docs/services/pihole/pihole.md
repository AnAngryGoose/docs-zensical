# Pi-hole DNS Blocker

## Overview
The [Pi-hole®](https://github.com/pi-hole/pi-hole) is a [DNS sinkhole](https://en.wikipedia.org/wiki/DNS_Sinkhole) that protects your devices from unwanted content without installing any client-side software.

---

## Installation Methods

### Option 1: Docker Compose (Recommended)
This is the preferred method for this homelab setup. It integrates easily with existing Docker networks and storage.

```yaml
# More info at [https://github.com/pi-hole/docker-pi-hole/](https://github.com/pi-hole/docker-pi-hole/) and [https://docs.pi-hole.net/](https://docs.pi-hole.net/)
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      # DNS Ports
      - "53:53/tcp"
      - "53:53/udp"
      # Default HTTP Port
      - "80:80/tcp"
      # Default HTTPs Port. FTL will generate a self-signed certificate
      - "443:443/tcp"
      # Uncomment the below if using Pi-hole as your DHCP Server
      #- "67:67/udp"
      # Uncomment the line below if you are using Pi-hole as your NTP server
      #- "123:123/udp"
    environment:
      # Set the appropriate timezone for your location from
      # [https://en.wikipedia.org/wiki/List_of_tz_database_time_zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones), e.g:
      TZ: 'Europe/London'
      # Set a password to access the web interface. Not setting one will result in a random password being assigned
      FTLCONF_webserver_api_password: 'correct horse battery staple'
      # If using Docker's default `bridge` network setting the dns listening mode should be set to 'ALL'
      FTLCONF_dns_listeningMode: 'ALL'
    # Volumes store your data between container upgrades
    volumes:
      # For persisting Pi-hole's databases and common configuration file
      - './etc-pihole:/etc/pihole'
      # Uncomment the below if you have custom dnsmasq config files that you want to persist. Not needed for most starting fresh with Pi-hole v6. If you're upgrading from v5 you and have used this directory before, you should keep it enabled for the first v6 container start to allow for a complete migration. It can be removed afterwards. Needs environment variable FTLCONF_misc_etc_dnsmasq_d: 'true'
      #- './etc-dnsmasq.d:/etc/dnsmasq.d'
    cap_add:
      # See [https://github.com/pi-hole/docker-pi-hole#note-on-capabilities](https://github.com/pi-hole/docker-pi-hole#note-on-capabilities)
      # Required if you are using Pi-hole as your DHCP server, else not needed
      - NET_ADMIN
      # Required if you are using Pi-hole as your NTP client to be able to set the host's system time
      - SYS_TIME
      # Optional, if Pi-hole should get some more processing time
      - SYS_NICE
    restart: unless-stopped

```

!!! info "Note On Capabilities"
[FTLDNS](https://docs.pi-hole.net/ftldns/) expects capabilities like `CAP_NET_BIND_SERVICE` (binding port 53) and `CAP_NET_ADMIN` (DHCP).

```
The docker image automatically grants these if available. We explicitly add `NET_ADMIN` in the compose file above to ensure smooth operation.

```

### Option 2: Automated Install (Bare Metal)

If you prefer to install directly on the OS:

```bash
curl -sSL [https://install.pi-hole.net](https://install.pi-hole.net) | bash

```

!!! warning "Security Warning"
[Piping to `bash` is a controversial topic](https://pi-hole.net/2016/07/25/curling-and-piping-to-bash/). It prevents you from reading code before it runs. If you prefer to review the code first, use the methods below.

**Alternatives:**

* **Clone & Run:**
```bash
git clone --depth 1 [https://github.com/pi-hole/pi-hole.git](https://github.com/pi-hole/pi-hole.git) Pi-hole
cd "Pi-hole/automated install/"
sudo bash basic-install.sh

```


* **Download Manually:**
```bash
wget -O basic-install.sh [https://install.pi-hole.net](https://install.pi-hole.net)
sudo bash basic-install.sh

```



---

## Configuration & Network Setup

Once installed, you must configure your network to actually use the Pi-hole.

### Router Configuration

Configure your router's DHCP settings to use the Pi-hole's IP address as the **Primary DNS Server**. This ensures all connected devices are automatically protected.

### Host Configuration (The Server itself)

The host machine running the Pi-hole container does not automatically use it.

**Method 1: DHCPCD (Standard Linux)**
Add to `/etc/dhcpcd.conf`:

```properties
static domain_name_servers=127.0.0.1

```

!!! warning
If your Pi-hole host uses itself as upstream DNS and Pi-hole fails, the host loses DNS. This can prevent repair attempts (e.g., `pihole -r`) because the internet connection will appear "down."

**Method 2: Permissions**
Pi-hole v6 uses a new API. To allow your user to run CLI commands without a password:

```bash
# Debian/Ubuntu/RasPi OS
sudo usermod -aG pihole $USER

# Alpine
sudo addgroup pihole $USER

```

---

## Advanced: Local DNS & Wildcards

To make your local services reachable via a custom domain (e.g., `service.vanth.me`) while on your LAN, we use `dnsmasq` to intercept requests and point them to your Reverse Proxy (Nginx Proxy Manager).

1. **Create Config:**
Create `/etc/dnsmasq.d/99-wildcard.conf` inside the container (or mapped volume):
```text
address=/.vanth.me/192.168.0.116

```


*(Replace `192.168.0.116` with your Nginx Proxy Manager Host IP)*
2. **Activate:**
In Pi-hole Settings > Misc > **Enable misc.etc_dnsmasq_d**.
3. **Flush:**
Run `pihole restartdns` in the container or `ipconfig /flushdns` on your client machine.

---

## Tailscale Subnet Router Integration

This setup allows you to access your home network (and use your Pi-hole for DNS) from anywhere in the world, without opening ports on your firewall.

### 1. Enable IP Forwarding

You must allow the host to route traffic.

```bash
# Enable IPv4 forwarding immediately
sudo sysctl -w net.ipv4.ip_forward=1

# Make the change permanent across reboots
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf

```

### 2. Advertise Subnet Routes

Tell Tailscale that this machine can route traffic to your LAN (192.168.0.x).

```bash
# Advertise the 192.168.0.0/24 subnet
sudo tailscale up --advertise-routes=192.168.0.0/24

```

### 3. Approve Route

1. Go to the [Tailscale Admin Console](https://login.tailscale.com/admin/machines).
2. Find your router node (the machine you just configured).
3. Click the **...** menu -> **Edit route settings**.
4. Approve the `192.168.0.0/24` route.

### 4. How it works

1. You set Pi-hole as your Nameserver in Tailscale DNS settings.
2. Your remote phone queries `service.vanth.me`.
3. Pi-hole resolves this to the LAN IP (`192.168.0.116`).
4. Because the Subnet Route is active, your phone sends the traffic through the Tailscale tunnel to your home server.

```
