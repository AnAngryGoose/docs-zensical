---
title: Networking Basics
description: Network configuration, diagnostics, and DNS management on Linux.
---

# Networking Basics

> **Reference:** [iproute2 Docs](https://wiki.linuxfoundation.org/networking/iproute2) · [Debian Wiki — NetworkConfiguration](https://wiki.debian.org/NetworkConfiguration) · [Ubuntu Docs — Networking](https://help.ubuntu.com/community/NetworkConfigurationCommandLine)

---

## Viewing Network Info

```bash
# Show all interfaces and addresses
ip addr show
ip a                    # shorthand

# Show a specific interface
ip addr show eth0

# Show routing table
ip route show
ip r                    # shorthand

# Show default gateway
ip route show default

# Show interface statistics (errors, drops)
ip -s link show eth0

# Show ARP cache (local network neighbors)
ip neigh show
```

---

## Interface Management

```bash
# Bring interface up / down
sudo ip link set eth0 up
sudo ip link set eth0 down

# Assign a temporary IP address (lost on reboot)
sudo ip addr add 192.168.1.50/24 dev eth0

# Remove an IP address
sudo ip addr del 192.168.1.50/24 dev eth0

# Add a temporary route
sudo ip route add 10.0.0.0/24 via 192.168.1.1

# Delete a route
sudo ip route del 10.0.0.0/24
```

---

## Persistent Network Configuration

!!! note "Debian vs Ubuntu"
    Debian uses `/etc/network/interfaces`. Ubuntu (17.10+) uses **Netplan** with `.yaml` files in `/etc/netplan/`.

### Debian — `/etc/network/interfaces`

```bash
# /etc/network/interfaces

# Loopback
auto lo
iface lo inet loopback

# Static IP
auto eth0
iface eth0 inet static
    address 192.168.1.10
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 192.168.1.1

# DHCP
auto eth0
iface eth0 inet dhcp
```

```bash
# Apply changes
sudo systemctl restart networking
# or for a single interface
sudo ifdown eth0 && sudo ifup eth0
```

### Ubuntu — Netplan (`/etc/netplan/*.yaml`)

```yaml
# /etc/netplan/01-netcfg.yaml

network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.1.10/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 192.168.1.1
          - 1.1.1.1
```

```bash
# Test config before applying
sudo netplan try

# Apply permanently
sudo netplan apply
```

---

## DNS

```bash
# /etc/hosts — local static hostname resolution (checked before DNS)
# /etc/resolv.conf — DNS server config (often managed automatically)
# /etc/nsswitch.conf — controls resolution order (files, dns, mdns)

# View current DNS servers
cat /etc/resolv.conf
resolvectl status           # systemd-resolved (Ubuntu default)

# Query DNS manually
dig google.com
dig @192.168.1.1 google.com   # query specific DNS server
dig google.com +short          # just the IP

# Reverse lookup
dig -x 8.8.8.8

# nslookup alternative
nslookup google.com
nslookup google.com 192.168.1.1

# Flush DNS cache (systemd-resolved)
sudo resolvectl flush-caches

# Flush on older systems
sudo systemd-resolve --flush-caches
```

### `/etc/hosts`

```bash
# Add a local hostname override
sudo nano /etc/hosts

# Example entries
127.0.0.1       localhost
192.168.1.10    appserver appserver.home
192.168.1.20    nas nas.home
192.168.1.1     router opnsense.home
```

---

## Checking Open Ports & Connections

```bash
# List all listening ports
ss -tlnp           # TCP listening, numeric, with process
ss -ulnp           # UDP listening

# All established connections
ss -tnp

# Check if a specific port is in use
ss -tlnp | grep :8080

# Legacy equivalent (if ss not available)
netstat -tlnp
netstat -an | grep :8080
```

| Flag | Meaning |
|------|---------|
| `-t` | TCP |
| `-u` | UDP |
| `-l` | Listening only |
| `-n` | Numeric (no hostname resolution) |
| `-p` | Show process name/PID |

---

## Diagnostics

```bash
# Test connectivity
ping 8.8.8.8               # ICMP to IP
ping google.com            # tests DNS + connectivity

# Trace network path
traceroute google.com
mtr google.com             # live combined ping + traceroute (install mtr)

# Test TCP connection to a port
nc -zv 192.168.1.10 22     # port 22 open?
nc -zv 192.168.1.10 80

# Test HTTP without a browser
curl -I https://example.com              # headers only
curl -v https://example.com             # verbose
wget --spider https://example.com       # check URL reachability

# Check bandwidth between two hosts (install iperf3)
# On server:  iperf3 -s
# On client:  iperf3 -c server-ip
```

---

## Hostname

```bash
# View hostname
hostname
hostnamectl

# Set hostname permanently
sudo hostnamectl set-hostname newhostname

# Also update /etc/hosts to match
sudo nano /etc/hosts
# Change: 127.0.1.1  oldhostname
# To:     127.0.1.1  newhostname
```

---

## Network Interfaces — Common Names

Modern Linux uses predictable interface names. You'll see these on servers:

| Name | Meaning |
|------|---------|
| `eth0`, `eth1` | Old-style Ethernet |
| `enp2s0` | PCI Ethernet (bus 2, slot 0) |
| `eno1` | Onboard Ethernet |
| `lo` | Loopback (always 127.0.0.1) |
| `docker0` | Docker bridge network |
| `veth*` | Virtual Ethernet (Docker container pairs) |