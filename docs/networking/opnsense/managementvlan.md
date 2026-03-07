---
title: Management VLAN Migration — OPNsense & Omada
description: Step-by-step guide to migrating Omada switches and APs to a dedicated management VLAN using OPNsense as a non-Omada gateway. Includes exact firewall rules, DHCP options, order of operations, and lessons learned from a real migration.
tags:
  - networking
  - vlans
  - opnsense
  - omada
  - managements
---

# Management VLAN Migration — OPNsense & Omada

This guide covers migrating Omada managed switches and APs from flat LAN management to a dedicated **MANAGEMENT VLAN** when using OPNsense as the gateway (non-Omada router scenario).

This is **Topology 2** in TP-Link's official documentation: [How to configure Management VLAN in Omada SDN Controller](https://www.omadanetworks.com/us/support/faq/2814/)

---

## Overview

By default, Omada devices are managed over whatever network they're on — typically a flat LAN (`192.168.0.x`). Moving management traffic to a dedicated VLAN isolates it from client traffic, reduces attack surface, and makes the network easier to reason about.

**The challenge:** the Omada controller itself is likely on a different VLAN (e.g. LAB/SERVERS), and after migration the switches will be on a separate MANAGEMENT VLAN. OPNsense must route between them, and firewall rules must explicitly permit it.

---

## Network Reference

This guide uses the following addressing. Adjust to match your own setup.

| VLAN | Name | Subnet | Gateway |
|---|---|---|---|
| 10 | MANAGEMENT | 10.10.10.0/24 | 10.10.10.1 |
| 20 | TRUSTED | 10.10.20.0/24 | 10.10.20.1 |
| 30 | LAB | 10.10.30.0/24 | 10.10.30.1 |
| 50 | IOT | 10.10.50.0/24 | 10.10.50.1 |
| 99 | GUEST | 10.10.99.0/24 | 10.10.99.1 |

| Device | IP | VLAN |
|---|---|---|
| OPNsense | 10.10.10.1 | MANAGEMENT (and all other VLANs as gateway) |
| Omada Controller (prod-deb-01) | 10.10.30.20 | LAB |
| SG2210P (core switch) | 10.10.10.2 | MANAGEMENT |
| SG2016P (downstream switch) | 10.10.10.3 | MANAGEMENT |
| EAP650 (WAP) | 10.10.10.4 | MANAGEMENT |

---

## Prerequisites

Before touching anything in Omada, the following must be in place in OPNsense. Skipping any of these will cause devices to disconnect and be unrecoverable without physical access.

### 1. MANAGEMENT VLAN Interface

The MANAGEMENT VLAN interface must exist on OPNsense with a static IP and DHCP range.

**Interfaces → [MANAGEMENT]:**
- IPv4: Static, `10.10.10.1/24`

**Services → Dnsmasq → DHCP ranges:**
- Interface: MANAGEMENT
- Range: `10.10.10.100` – `10.10.10.200`

---

### 2. DHCP Option 138 — Omada Controller Discovery

After migration, switches reboot into the MANAGEMENT VLAN. They can no longer find the Omada controller by broadcast (broadcast doesn't cross VLANs). DHCP Option 138 (CAPWAP AC) embeds the controller IP in the DHCP response so the switch knows where to connect.

**Services → Dnsmasq → DHCP options → Add:**

| Field | Value |
|---|---|
| Interface | MANAGEMENT |
| Option | `option_capwap_ac_v4 [138]` |
| Value | `10.10.30.20` (Omada controller IP) |

> Without this, switches survive a single session but become orphaned after a reboot.

---

### 3. Firewall Rule — Allow Omada Controller to Reach MANAGEMENT VLAN

The Omada controller runs on the LAB VLAN (`10.10.30.20`). After migration, the switches are on the MANAGEMENT VLAN (`10.10.10.x`). Without an explicit allow rule, OPNsense blocks this traffic.

**Firewall → Rules → LAB → Add (place ABOVE any block rules):**

| Field | Value |
|---|---|
| Action | Pass |
| Protocol | Any |
| Source | `10.10.30.20` (Omada controller) |
| Destination | MANAGEMENT net |
| Description | Allow Omada to manage MGMT VLAN |

> **Order matters.** This rule must be above any "Block LAB to MANAGEMENT" rule. OPNsense evaluates rules top to bottom, first match wins.

**Example LAB rules in correct order:**

```
1. PASS   — 10.10.30.20 → MANAGEMENT net    [Allow Omada to manage MGMT VLAN]
2. BLOCK  — LAB net → TRUSTED net            [Block LAB to TRUSTED]
3. BLOCK  — LAB net → MANAGEMENT net         [Block LAB to MANAGEMENT]
4. PASS   — LAB net → any                    [LAB Internet Access]
5. PASS   — LAB net → 10.10.30.1:53 DNS      [Force DNS to AdGuard - LAB]
```

---

### 4. DHCP Option 6 — Push Correct DNS to MANAGEMENT Devices

Switches and APs use DNS for NTP lookups and firmware update checks. Push the correct DNS server for the MANAGEMENT VLAN.

**Services → Dnsmasq → DHCP options → Add:**

| Field | Value |
|---|---|
| Interface | MANAGEMENT |
| Option | `dns-server [6]` |
| Value | `10.10.10.1` |

---

### 5. NAT DNS Redirect for MANAGEMENT

Prevents devices on the MANAGEMENT VLAN from bypassing your DNS server.

**Firewall → NAT → Port Forward → Add:**

| Field | Value |
|---|---|
| Interface | MANAGEMENT |
| Protocol | TCP/UDP |
| Destination / Invert | ✅ Enabled |
| Destination | `10.10.10.1` |
| Destination Port | 53 |
| Redirect Target IP | `10.10.10.1` |
| Redirect Target Port | 53 |
| Description | Force DNS to AdGuard - MANAGEMENT |

---

### 6. Verify OPNsense is Reachable on MANAGEMENT VLAN

From any device on the network, confirm OPNsense responds on the MANAGEMENT VLAN interface before touching any switches:

```bash
ping 10.10.10.1
```

If this doesn't respond, do not proceed — the MANAGEMENT VLAN interface is not configured correctly.

---

## Trunk Profile Verification

Every trunk port in the chain from OPNsense to each switch must carry VLAN 10 tagged. Verify in Omada:

**Settings → Wired Networks → Switch Profile → [your trunk profile]**

Confirm VLAN 10 - MANAGEMENT is in the **Tagged Networks** list alongside all other VLANs.

If VLAN 10 is missing from the trunk profile, switches will never receive MANAGEMENT VLAN traffic even if correctly configured.

---

## Migration Order

> **Critical:** Always work upstream first. Migrating a downstream switch before its upstream switch is configured means tagged traffic cannot flow, and the device will be unreachable.

```
OPNsense (already on 10.10.10.1)
    ↓
SG2210P (core/upstream switch) → migrate first
    ↓
SG2016P (downstream switch) → migrate second
    ↓
EAP650 (WAP) → migrate last
```

---

## Switch Migration — SG2210P and SG2016P

In Omada: **Devices → [Switch] → Config → VLAN Interface → Edit [10 - MANAGEMENT]**

Enable the **Management VLAN** checkbox first — this reveals the gateway and DNS fields.

| Field | Value |
|---|---|
| Management VLAN | ✅ Enable |
| IP Address Mode | Static |
| IP Address | `10.10.10.2` (SG2210P) or `10.10.10.3` (SG2016P) |
| Subnet Mask | `255.255.255.0` |
| Gateway | `10.10.10.1` |
| DNS | `10.10.10.1` |
| DHCP Mode | None |

Click **Apply**.

The switch will briefly drop from Omada as it transitions. Immediately ping the new IP:

```bash
ping 10.10.10.2   # SG2210P
ping 10.10.10.3   # SG2016P
```

Do not proceed to the next device until the current one has reconnected in Omada and ping responds.

---

## AP Migration — EAP650

APs use a different configuration path than switches.

In Omada: **Devices → [EAP] → Config → Services → VLAN → Management VLAN**

| Field | Value |
|---|---|
| Management VLAN | Custom |
| LAN Network | 10 - MANAGEMENT |

Click **Apply**.

The EAP will get its IP via DHCP from OPNsense's MANAGEMENT range (`10.10.10.100–200`). It does not have a static IP option in this interface — DHCP Option 138 handles controller discovery after each reboot.

> **Note:** The EAP also has an IP Settings section (Config → IP Settings) with a fixed IP reservation field. This is separate and refers to the IP assigned via DHCP reservation, not the management VLAN assignment. Ignore it for this migration — the Services → Management VLAN setting is what matters.

---

## Post-Migration Verification

Once all devices are migrated:

```bash
# Confirm all devices reachable on MANAGEMENT VLAN
ping 10.10.10.1   # OPNsense
ping 10.10.10.2   # SG2210P
ping 10.10.10.3   # SG2016P
ping 10.10.10.4   # EAP650
```

Confirm in Omada device list that all three devices show `10.10.10.x` IPs and status **Connected**.

---

## What Can Go Wrong

### Switch disconnects and never reconnects

**Most likely cause:** The firewall rule allowing Omada controller (`10.10.30.20`) to reach MANAGEMENT VLAN is missing or below a block rule.

Check **Firewall → Rules → LAB** and confirm the allow rule is above the block rule. Then power cycle the switch — it will retry connecting to Omada.

### Switch reconnects but Omada can't manage it

**Most likely cause:** DHCP Option 138 is missing or set to the wrong IP. The switch rebooted, got a `10.10.10.x` address, but doesn't know where the Omada controller is.

Check **Services → Dnsmasq → DHCP options → MANAGEMENT** and confirm `option_capwap_ac_v4 [138]` = `10.10.30.20`.

### Switch completely unreachable after applying config

**Most likely cause:** VLAN 10 is not in the trunk profile on one or more links in the chain between the switch and OPNsense.

Check the trunk profile in Omada and verify VLAN 10 is tagged. Also verify OPNsense responds at `10.10.10.1` — if not, the MANAGEMENT VLAN interface itself is the issue.

### Factory reset required

If a switch becomes completely unrecoverable (no ping, not appearing in Omada):

1. Physically reset the switch (hold reset button 5–10 seconds)
2. The switch returns to flat LAN defaults (`192.168.0.x` DHCP)
3. All port profiles are wiped — every device connected to it loses its VLAN assignment
4. Connect to Omada via a device on the upstream switch or flat LAN
5. Re-adopt the reset switch
6. Re-apply all port profiles before moving cables back

> **Lesson learned:** When a downstream switch factory resets, any devices with static IPs on VLANs (e.g. servers at `10.10.30.x`) will lose connectivity because their ports revert to Default VLAN 1. Recovery requires either physical access to set a temporary flat LAN IP on the server, or connecting the server temporarily to a working switch upstream.

---

## dnsmasq Host Entries for Infrastructure

Once switches are on the MANAGEMENT VLAN with static IPs, add hostname entries so they resolve via `.internal` DNS.

**Services → Dnsmasq → Hosts → Add** (one per device):

| Host | Domain | IP |
|---|---|---|
| sg2210p | internal | 10.10.10.2 |
| sg2016p | internal | 10.10.10.3 |
| eap650 | internal | 10.10.10.4 |

Restart dnsmasq after adding entries.

---

## Security Hardening (Post-Migration)

Once management is isolated, add a rule to prevent TRUSTED VLAN devices from accessing MANAGEMENT infrastructure:

**Firewall → Rules → TRUSTED → Add (above the TRUSTED full access rule):**

| Field | Value |
|---|---|
| Action | Block |
| Protocol | Any |
| Source | TRUSTED net |
| Destination | MANAGEMENT net |
| Log | Yes |
| Description | Block TRUSTED to MANAGEMENT |

