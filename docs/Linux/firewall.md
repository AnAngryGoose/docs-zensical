---
title: Firewall (ufw / nftables)
description: Host-level firewall management with ufw and nftables on Debian and Ubuntu.
---

# Firewall (ufw / nftables)

> **Reference:** [Ubuntu Docs — ufw](https://help.ubuntu.com/community/UFW) · [Debian Wiki — nftables](https://wiki.debian.org/nftables) · [nftables Wiki](https://wiki.nftables.org/)

!!! info "Host firewall vs network firewall"
    This covers **host-level** firewall rules (on the Linux machine itself). Your OPNsense router handles network-level filtering. Host firewall adds defense-in-depth — if a container exposes a port you forgot about, the host firewall is a second layer of protection.

---

## ufw — Uncomplicated Firewall

`ufw` is the simplest firewall tool and is the Ubuntu default. It's a frontend to `iptables`/`nftables`.

!!! note "Debian vs Ubuntu"
    `ufw` is pre-installed on Ubuntu. On Debian, install it: `sudo apt install ufw`. Both work identically.

### Basic Setup

```bash
# Check current status
sudo ufw status
sudo ufw status verbose      # more detail
sudo ufw status numbered     # with rule numbers

# Enable / disable
sudo ufw enable
sudo ufw disable

# Reset all rules to default
sudo ufw reset
```

!!! warning
    Before enabling ufw, always allow SSH first or you will lock yourself out.

    ```bash
    sudo ufw allow ssh       # allow SSH before enabling
    sudo ufw enable
    ```

---

### Default Policies

```bash
# Deny all incoming, allow all outgoing (recommended starting point for servers)
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

---

### Allow Rules

```bash
# By service name (uses /etc/services)
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https

# By port number
sudo ufw allow 22
sudo ufw allow 8080
sudo ufw allow 443

# By port and protocol
sudo ufw allow 22/tcp
sudo ufw allow 53/udp

# Port range
sudo ufw allow 8000:8100/tcp

# From a specific IP
sudo ufw allow from 192.168.1.0/24
sudo ufw allow from 192.168.1.50

# From specific IP to specific port
sudo ufw allow from 192.168.1.50 to any port 22
sudo ufw allow from 192.168.1.0/24 to any port 5984   # CouchDB — LAN only
```

---

### Deny Rules

```bash
# Deny a port
sudo ufw deny 23            # block telnet

# Deny from an IP
sudo ufw deny from 10.0.0.5

# Deny from IP to specific port
sudo ufw deny from 10.0.0.5 to any port 22
```

---

### Delete Rules

```bash
# By rule number (get numbers from 'ufw status numbered')
sudo ufw delete 3

# By matching the rule
sudo ufw delete allow 8080
sudo ufw delete allow from 192.168.1.50 to any port 22
```

---

### App Profiles

ufw includes named profiles for common applications:

```bash
# List available profiles
sudo ufw app list

# View a profile's ports
sudo ufw app info nginx

# Allow by profile
sudo ufw allow 'Nginx Full'    # HTTP + HTTPS
sudo ufw allow 'Nginx HTTP'    # HTTP only
sudo ufw allow 'OpenSSH'
```

---

### Useful ufw Commands

```bash
# Check if ufw starts on boot
sudo systemctl is-enabled ufw

# View ufw logs
sudo journalctl -u ufw
sudo tail -f /var/log/ufw.log

# Enable logging
sudo ufw logging on
sudo ufw logging medium    # options: off, low, medium, high, full
```

---

## nftables

`nftables` is the modern replacement for `iptables` and is the default on Debian 10+. `ufw` uses it as its backend on newer systems, but you can also use `nftables` directly for more advanced rulesets.

!!! note "Debian vs Ubuntu"
    Debian 10+ uses nftables natively. Ubuntu still primarily exposes `ufw`/`iptables-nft` as the user-facing interface. Direct `nft` usage is more common on Debian servers.

### Basic Concepts

```
table    → top-level namespace (like a rulebook)
chain    → ordered list of rules within a table
rule     → individual match + action
```

### Checking nftables

```bash
# View all current rules
sudo nft list ruleset

# View a specific table
sudo nft list table inet filter

# Flush all rules (careful — removes all filtering)
sudo nft flush ruleset
```

### Basic Ruleset Example

```bash
sudo nano /etc/nftables.conf
```

```nft
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # Accept established/related connections
        ct state established,related accept

        # Accept loopback
        iif lo accept

        # Accept ICMP (ping)
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept

        # Accept SSH from LAN only
        ip saddr 192.168.1.0/24 tcp dport 22 accept

        # Accept HTTP and HTTPS
        tcp dport { 80, 443 } accept

        # Accept specific service ports from LAN
        ip saddr 192.168.1.0/24 tcp dport { 8080, 8090, 9898 } accept

        # Log and drop everything else
        log prefix "nftables drop: " drop
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

```bash
# Apply rules
sudo nft -f /etc/nftables.conf

# Enable on boot
sudo systemctl enable nftables
sudo systemctl start nftables

# Test config syntax without applying
sudo nft -c -f /etc/nftables.conf
```

---

## Docker & Firewall Interaction

!!! warning "Docker bypasses ufw"
    Docker directly modifies `iptables`/`nftables` and will expose ports **even if ufw blocks them**. A container with `-p 0.0.0.0:8080:8080` is publicly accessible regardless of ufw rules.

**Options to fix this:**

**Option 1 — Bind to localhost only (per container)**

In your compose file, bind to `127.0.0.1` instead of all interfaces:

```yaml
ports:
  - "127.0.0.1:8080:8080"   # only accessible locally
```

**Option 2 — Use a dedicated Docker network + reverse proxy**

Don't publish ports at all. Use an internal Docker network and let NPM/nginx handle external access via its own published ports.

**Option 3 — Restrict via `DOCKER-USER` chain (iptables)**

```bash
# Allow LAN access to Docker ports, block everything else
sudo iptables -I DOCKER-USER -i eth0 -s 192.168.1.0/24 -j ACCEPT
sudo iptables -I DOCKER-USER -i eth0 -j DROP
```

This is the only iptables chain Docker won't overwrite.