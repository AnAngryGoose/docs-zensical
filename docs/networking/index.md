---
title: Network Overview
description: High-level reference for network layers, models, and standard protocols.
icon: material/lan
---

# Network Architecture Overview

!!! abstract "Quick Reference"
    This document serves as an index for standard network communication models. It covers the 7-layer OSI model, the 4-layer TCP/IP suite, and commonly referenced protocols.

## Conceptual Models

## OSI Model (7 Layers)

| Layer | Name | Function | Common Protocols |
| :---: | :--- | :--- | :--- |
| **7** | **Application** | Network process to application | HTTP, DNS, DHCP |
| **6** | **Presentation**| Data representation and encryption | TLS, SSL |
| **5** | **Session** | Interhost communication | NetBIOS |
| **4** | **Transport** | End-to-end connections and reliability | TCP, UDP |
| **3** | **Network** | Path determination and IP addressing | IPv4, IPv6, ICMP, NAT |
| **2** | **Data Link** | Physical addressing (MAC) | Ethernet, ARP |
| **1** | **Physical** | Media, signal, and binary transmission | Fiber, Wi-Fi |

## TCP/IP Model (4 Layers)

| Layer | Name | OSI Equivalent | Core Protocols |
| :---: | :--- | :--- | :--- |
| **4** | **Application** | Application, Presentation, Session (7, 6, 5) | HTTP, DNS, DHCP |
| **3** | **Transport** | Transport (4) | TCP, UDP |
| **2** | **Internet** | Network (3) | IP, ICMP, NAT, ARP |
| **1** | **Network Access**| Data Link, Physical (2, 1) | Ethernet, Wi-Fi |
---

## Core Protocols & Services

### Address & Name Resolution
* **DNS (Domain Name System):** Translates human-readable domain names (e.g., `google.com`) into routable IP addresses. Uses Port 53.
* **DHCP (Dynamic Host Configuration Protocol):** Automatically assigns IP addresses, subnet masks, and default gateways to devices joining a network. Uses Ports 67/68.
* **ARP (Address Resolution Protocol):** Bridges Layer 2 and Layer 3 by resolving a known IP address to an unknown physical MAC address on a local network.

### Routing & Translation
* **NAT (Network Address Translation):** Operates on routers/firewalls to translate private, internal IP addresses (e.g., `192.168.x.x`) into a single public IP address for internet access. Crucial for conserving IPv4 space.

### Transport Layer
* **TCP (Transmission Control Protocol):** Connection-oriented. Guarantees delivery, ordering, and error-checking. Used for web traffic, SSH, and file transfers.
* **UDP (User Datagram Protocol):** Connectionless. Fast, but provides no delivery guarantees. Used for live video, VoIP, and DNS queries.

## Common Ports

| Port | Protocol | Usage |
| :--- | :--- | :--- |
| `22` | SSH | Secure remote command-line access |
| `53` | DNS | Domain Name System |
| `67/68`| DHCP | Dynamic Host Configuration |
| `80` | HTTP | Unencrypted web traffic |
| `443`| HTTPS | Encrypted web traffic (TLS/SSL) |
| `3389`| RDP | Remote Desktop Protocol |

## Common Architectures

### Layered DNS Stack

A standard approach to network-wide filtering, recursive resolution, and local hostname registration (commonly deployed on OPNsense/pfSense).

**The Resolution Chain:**
`Client ➔ AdGuard Home (:53) ➔ Unbound (:5335) ➔ dnsmasq (:53053) / Root Servers`

* **AdGuard Home (Frontend):** Ad/tracker filtering. Sits in front so every query is inspected.
* **Unbound (Middle):** Recursive resolver and DNSSEC validation. Forwards local `.internal` queries to dnsmasq, resolves public domains directly.
* **dnsmasq (Backend):** DHCP server and local DNS registry. Resolves `hostname.internal` automatically based on DHCP leases.

---

### VLAN Planning & Concepts

!!! tip "Numbering Strategy"
    A clean numbering scheme maps the VLAN ID directly to the third octet of the subnet (e.g., VLAN **30** = `10.10.`**`30`**`.0/24`). Leave gaps (10, 20, 30) to allow for future expansion.

## Standard Numbering Scheme

| ID | Name | Purpose | Default Routing Policy |
| :---: | :--- | :--- | :--- |
| **10** | MANAGEMENT | Infrastructure, routers, switches | Allow to All |
| **20** | TRUSTED | Workstations, personal devices | Allow to All |
| **30** | SERVERS | Self-hosted services, NAS | Block to Trusted/Mgmt |
| **50** | IOT | Smart home, cameras | Block to all Internal |
| **99** | GUEST | Guest WiFi | Internet only |

## Tagged vs Untagged Ports

| Port Type | Traffic | Used For | Device Awareness |
| :--- | :--- | :--- | :--- |
| **Access** | Untagged | End devices (PCs, servers, IoT) | **Unaware.** Switch adds/removes tags internally. |
| **Trunk** | Tagged | Routers, switches, hypervisors | **Aware.** Device tags its own ethernet frames. |