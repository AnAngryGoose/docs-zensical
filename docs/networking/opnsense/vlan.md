---
title: VLANs with OPNsense & Omada
description: Complete guide to planning, configuring, and implementing VLANs using OPNsense as a router and TP-Link Omada managed switches.
tags:
  - networking
  - vlans
  - opnsense
  - omada
---

# VLANs with OPNsense & Omada

This guide covers planning and implementing a VLAN network using **OPNsense** as the router/firewall and **TP-Link Omada** managed switches. It covers the full stack: VLAN design, OPNsense interface and DHCP configuration, firewall rules, NAT, and Omada switch profiles and port assignments.

---

## Concepts

### What is a VLAN?

A VLAN (Virtual Local Area Network) logically separates devices on the same physical network into isolated segments. Devices on different VLANs cannot communicate with each other unless explicitly permitted by a router or firewall rule.

Common use cases:

- Isolating IoT devices from trusted workstations
- Separating guest WiFi from internal networks
- Segmenting servers from personal devices
- Creating a dedicated management network for infrastructure

### Tagged vs Untagged

Understanding the difference between tagged and untagged traffic is fundamental to VLAN configuration.

**Untagged (access ports)**
Used for end devices that are not VLAN-aware — computers, servers, NAS devices, printers. The device sends a normal ethernet frame with no VLAN information. The switch receives it, stamps the appropriate VLAN tag internally based on the port's profile, and routes it accordingly. On egress, the tag is stripped before delivery to the device. The device never knows VLANs exist.

**Tagged (trunk ports)**
Used for VLAN-aware devices — routers, upstream switches, hypervisors. The device itself includes a VLAN tag in each ethernet frame. The switch receives the frame with the tag intact and uses it to route traffic to the correct VLAN. Trunk ports carry multiple VLANs simultaneously, each identified by its tag.

| Port Type | Used For | Device Awareness |
|---|---|---|
| Untagged / Access | End devices (PCs, servers) | Device is VLAN-unaware |
| Tagged / Trunk | Routers, switches, hypervisors | Device tags its own frames |

### Router-on-a-Stick

With a single physical port connecting OPNsense to the switch, all VLAN traffic travels over one link — each VLAN tagged. OPNsense creates virtual sub-interfaces (one per VLAN) on top of that physical port. This is the most common homelab setup and works well at any scale that doesn't saturate the uplink.

---

## Planning

### VLAN Scheme

Design your VLANs before touching any configuration. A clean numbering scheme that maps to subnets makes the network far easier to manage and troubleshoot.

**Recommended pattern:** Use the VLAN ID as the third octet of the subnet.

| VLAN ID | Name | Subnet | Gateway | Purpose |
|---|---|---|---|---|
| 10 | MANAGEMENT | 10.10.10.0/24 | 10.10.10.1 | Infrastructure — router, switches, DNS |
| 20 | TRUSTED | 10.10.20.0/24 | 10.10.20.1 | Workstations, personal devices |
| 30 | SERVERS | 10.10.30.0/24 | 10.10.30.1 | Servers, self-hosted services |
| 50 | IOT | 10.10.50.0/24 | 10.10.50.1 | Smart home, cameras, IoT devices |
| 99 | GUEST | 10.10.99.0/24 | 10.10.99.1 | Guest WiFi — internet only |

> **Tip:** Leave gaps in your VLAN ID numbering (10, 20, 30 rather than 1, 2, 3) so you can insert new VLANs later without renumbering.

### Firewall Policy

Define your inter-VLAN routing policy before writing any rules. This makes the rules themselves straightforward to implement.

| Source | Destination | Policy | Reason |
|---|---|---|---|
| MANAGEMENT | All | Allow | Admins need access to everything |
| TRUSTED | All | Allow | Workstations need full access |
| SERVERS | TRUSTED | Block | Servers shouldn't reach workstations |
| SERVERS | MANAGEMENT | Block | Servers shouldn't manage infrastructure |
| SERVERS | WAN | Allow | Servers need internet |
| IOT | Internal (all) | Block | IoT isolated from all internal networks |
| IOT | WAN | Allow | IoT devices need internet |
| GUEST | Internal (all) | Block | Guests isolated from everything internal |
| GUEST | WAN | Allow | Guests need internet |

### IP Address Allocation

Plan static IP allocation to avoid conflicts with DHCP ranges.

**Recommended layout per subnet:**

| Range | Purpose |
|---|---|
| `.1` | Gateway (OPNsense VLAN interface) |
| `.2 – .9` | Reserved infrastructure (DNS, NTP, etc.) |
| `.10 – .99` | Static device assignments |
| `.100 – .200` | DHCP pool |
| `.201 – .254` | Reserved |

---

## OPNsense Configuration

### Step 1 — Create VLAN Interfaces

Navigate to **Interfaces → Other Types → VLAN**.

For each VLAN, click **+** and set:

| Field | Value | Notes |
|---|---|---|
| Device | Leave auto-generated | OPNsense names it `vlan0.X` |
| Parent Interface | Your LAN physical port (e.g. `igb0`) | The port connected to your switch |
| VLAN tag | VLAN ID (e.g. `10`) | Must match switch config exactly |
| VLAN priority | `0` | Leave default unless using QoS |
| Description | e.g. `MANAGEMENT` | For your reference only |

Click **Save** after each entry, then **Apply changes**.

---

### Step 2 — Assign VLAN Interfaces

Navigate to **Interfaces → Assignments**.

Each VLAN device you created needs to be assigned as an interface so OPNsense can route for it.

1. At the bottom, select a VLAN device from the **Device** dropdown
2. Click **+ Add**
3. Repeat for each VLAN
4. Click **Save**

You will now see each VLAN listed as an interface (OPT1, OPT2, etc.). Click each one to configure it.

---

### Step 3 — Configure Each VLAN Interface

Navigate to **Interfaces → [VLAN NAME]** for each interface.

| Field | Value | Notes |
|---|---|---|
| Enable | ✅ Checked | Must be enabled |
| Description | e.g. `MANAGEMENT` | Replaces the OPT label |
| IPv4 Configuration Type | Static IPv4 | |
| IPv4 Address | Gateway IP (e.g. `10.10.10.1`) | OPNsense acts as gateway |
| Subnet mask | `24` | For /24 subnets |
| IPv6 Configuration Type | None | Unless using IPv6 |

Click **Save** → **Apply changes** after each interface.

---

### Step 4 — Configure DHCP per VLAN

Navigate to **Services → Dnsmasq DNS & DHCP → DHCP ranges**.

Add one DHCP range per VLAN interface:

| Field | Value |
|---|---|
| Interface | Select the VLAN interface |
| Start address | e.g. `10.10.10.100` |
| End address | e.g. `10.10.10.200` |

> Do not create overlapping ranges. Static reservations (dnsmasq Hosts) do not need their own DHCP range — they sit outside the pool by convention.

---

### Step 5 — Push DNS Server via DHCP

Clients need to receive your DNS server (e.g. AdGuard Home) via DHCP so all DNS queries are filtered.

Navigate to **Services → Dnsmasq DNS & DHCP → DHCP options**.

For each VLAN interface, add:

| Field | Value |
|---|---|
| Interface | VLAN interface |
| Option | `6` (DNS server) |
| Value | IP address of your DNS server |

---

### Step 6 — Static DHCP Reservations

For servers and infrastructure that need consistent IPs, use static reservations rather than configuring IPs on the devices themselves where possible.

Navigate to **Services → Dnsmasq DNS & DHCP → Hosts**.

| Field | Value |
|---|---|
| Host | Hostname (e.g. `server01`) |
| IP address | Static IP outside DHCP pool (e.g. `10.10.30.10`) |
| Hardware address | Device MAC address |

> **Note:** Do not duplicate entries across both Dnsmasq Hosts and Unbound Overrides. Pick one. Unbound Overrides are preferred for DNS records. Dnsmasq Hosts are preferred for DHCP-bound static reservations.

---

### Step 7 — Firewall Rules

Navigate to **Firewall → Rules → [Interface tab]**.

OPNsense processes rules **top to bottom, first match wins**. Always place block rules above allow rules.

#### Field Reference

| Field | Description | Typical Value |
|---|---|---|
| **Action** | What to do with matching traffic | `Pass`, `Block`, or `Reject` |
| **Disabled** | Temporarily disable the rule | Unchecked |
| **Interface** | Which interface this rule applies to | The SOURCE interface |
| **Direction** | Inbound or outbound | Always `In` |
| **TCP/IP Version** | IP version | `IPv4` |
| **Protocol** | Traffic type | `Any`, `TCP`, `UDP`, `TCP/UDP`, `ICMP` |
| **Source** | Origin of traffic | Interface network (e.g. `SERVERS net`) |
| **Source Port** | Source port | `any` — source ports are ephemeral |
| **Destination** | Target of traffic | Specific network, host alias, or `any` |
| **Destination Port** | Target port | `any` for broad rules, specific port for service rules |
| **Log** | Log matching packets | Enable on all block rules |
| **Description** | Label for the rule | Always fill in — critical for maintenance |
| **State Type** | Connection tracking | `Keep State` (default) |

> **Block vs Reject:** `Block` silently drops packets (client times out). `Reject` sends an immediate refusal. Use `Block` for security rules — it gives less information to potential attackers and is the safer default.

---

#### MANAGEMENT Rules

One rule — full unrestricted access:

```
Action:      Pass
Interface:   MANAGEMENT
Direction:   In
Protocol:    Any
Source:      MANAGEMENT net
Destination: any
Log:         No
Description: Management full access
```

---

#### TRUSTED Rules

One rule — full unrestricted access:

```
Action:      Pass
Interface:   TRUSTED
Direction:   In
Protocol:    Any
Source:      TRUSTED net
Destination: any
Log:         No
Description: Trusted full access
```

---

#### SERVERS Rules

Three rules in this order:

**Rule 1 — Block servers from reaching trusted workstations**
```
Action:      Block
Interface:   SERVERS
Direction:   In
Protocol:    Any
Source:      SERVERS net
Destination: TRUSTED net
Log:         Yes
Description: Block SERVERS to TRUSTED
```

**Rule 2 — Block servers from reaching management infrastructure**
```
Action:      Block
Interface:   SERVERS
Direction:   In
Protocol:    Any
Source:      SERVERS net
Destination: MANAGEMENT net
Log:         Yes
Description: Block SERVERS to MANAGEMENT
```

**Rule 3 — Allow servers internet access**
```
Action:      Pass
Interface:   SERVERS
Direction:   In
Protocol:    Any
Source:      SERVERS net
Destination: any
Log:         No
Description: SERVERS internet access
```

> Rule 3 uses `any` as destination. Rules 1 and 2 above it have already blocked the specific internal targets — `any` here effectively means everything not already matched.

---

#### IOT Rules

Two rules in this order:

**Rule 1 — Block IoT from all internal networks**
```
Action:      Block
Interface:   IOT
Direction:   In
Protocol:    Any
Source:      IOT net
Destination: 10.0.0.0/8  (or your internal range)
Log:         Yes
Description: Block IOT to all internal
```

> Use an alias if your internal networks span multiple RFC1918 ranges (10.x.x.x, 192.168.x.x, 172.16.x.x). Create the alias under **Firewall → Aliases** and reference it as the destination.

**Rule 2 — Allow IoT internet access**
```
Action:      Pass
Interface:   IOT
Direction:   In
Protocol:    Any
Source:      IOT net
Destination: any
Log:         No
Description: IOT internet access
```

---

#### GUEST Rules

Two rules in this order:

**Rule 1 — Block guests from all internal networks**
```
Action:      Block
Interface:   GUEST
Direction:   In
Protocol:    Any
Source:      GUEST net
Destination: 10.0.0.0/8
Log:         Yes
Description: Block GUEST to all internal
```

**Rule 2 — Allow guest internet access**
```
Action:      Pass
Interface:   GUEST
Direction:   In
Protocol:    Any
Source:      GUEST net
Destination: any
Log:         No
Description: GUEST internet access
```

---

### Step 8 — Force DNS via NAT (Optional but Recommended)

Prevents clients from bypassing your DNS server by pointing at a public resolver directly (e.g. `8.8.8.8`). Any DNS query not going to your resolver gets redirected.

Navigate to **Firewall → NAT → Port Forward**. Create one rule per VLAN:

| Field | Value |
|---|---|
| Interface | VLAN interface (e.g. IOT) |
| TCP/IP Version | IPv4 |
| Protocol | TCP/UDP |
| Source | VLAN net |
| Source Port | any |
| Destination / Invert | ✅ Invert — enter your DNS server IP |
| Destination Port | 53 |
| Redirect Target IP | Your DNS server IP |
| Redirect Target Port | 53 |
| Log | Yes |
| Description | Force DNS to resolver - [VLAN NAME] |

> The **invert** on destination means: match any destination that is NOT your DNS server. This redirects rogue DNS queries while leaving legitimate queries to your resolver untouched.

---

## Omada Switch Configuration

### Step 1 — Create Networks in Omada

Navigate to **Settings → Wired Networks → Networks**.

Create a network entry for each VLAN. The VLAN IDs must exactly match what you configured in OPNsense.

| Field | Value | Notes |
|---|---|---|
| Name | e.g. `10 - MANAGEMENT` | Descriptive name |
| Purpose | `VLAN` | Not Interface |
| VLAN | VLAN ID (e.g. `10`) | Must match OPNsense |
| Application | `Gateways and Switches` | Standard choice for routed VLANs |
| IGMP Snooping | Disabled | Enable only if running multicast (e.g. Plex on local network) |
| MLD Snooping | Disabled | IPv6 multicast — leave off unless needed |
| QoS Queue | Disabled | Enable only if implementing QoS policy |
| Legal DHCP Servers | Disabled | Enable to restrict DHCP to known servers only (optional hardening) |
| DHCP L2 Relay | Disabled | Only needed if DHCP server is on a different L3 segment |

---

### Step 2 — Create Switch Port Profiles

Navigate to **Settings → Wired Networks → Switch Profile**.

Port profiles define how a switch port handles VLAN traffic. Create one profile per VLAN for access ports, plus one trunk profile for uplinks.

#### Access Port Profiles (one per VLAN)

Used for end devices — servers, workstations, NAS, IoT devices.

| Field | Value | Notes |
|---|---|---|
| Name | e.g. `20 - TRUSTED` | |
| PoE | Keep Device's Settings | Adjust per port if needed |
| Native Network | The VLAN (e.g. `20 - TRUSTED`) | Untagged frames are assigned this VLAN |
| Tagged Networks | None | Access ports carry one VLAN only |
| Untagged Networks | The same VLAN | Frames exit untagged to the device |
| Voice Network | None | Only needed for VoIP deployments |

Create one profile for each VLAN:

| Profile Name | Native Network | Tagged | Untagged |
|---|---|---|---|
| 10 - MANAGEMENT | 10 - MANAGEMENT | — | 10 - MANAGEMENT |
| 20 - TRUSTED | 20 - TRUSTED | — | 20 - TRUSTED |
| 30 - SERVERS | 30 - SERVERS | — | 30 - SERVERS |
| 50 - IOT | 50 - IOT | — | 50 - IOT |
| 99 - GUEST | 99 - GUEST | — | 99 - GUEST |

#### Trunk Profile (for uplink ports)

Used for ports connecting to OPNsense or between switches. Carries all VLANs tagged.

| Field | Value | Notes |
|---|---|---|
| Name | `TRUNK - ALL` | |
| PoE | Keep Device's Settings | |
| Native Network | `10 - MANAGEMENT` | Untagged frames default to management VLAN |
| Tagged Networks | All VLANs (10, 20, 30, 50, 99) | All VLANs pass through tagged |
| Untagged Networks | None | Uplink devices are VLAN-aware |
| Voice Network | None | |

> **Why Management as native on trunk?** Untagged frames that arrive on a trunk port (e.g. switch management traffic) need to land somewhere. Assigning management as native ensures switch-to-switch communication stays on the correct VLAN rather than defaulting to VLAN 1.

---

### Step 3 — Assign Profiles to Switch Ports

Navigate to **Devices → [Switch] → Ports**.

Click a port and assign the appropriate profile.

#### Example: Upstream Switch (connected to router)

| Port | Connected To | Profile |
|---|---|---|
| Port 1 | OPNsense LAN port | `TRUNK - ALL` |
| Port 2 | Downstream switch | `TRUNK - ALL` |
| Port 3–8 | End devices | Appropriate access profile |

#### Example: Downstream Switch

| Port | Connected To | Profile |
|---|---|---|
| Port 1 | Upstream switch | `TRUNK - ALL` |
| Port 2 | Server A | `30 - SERVERS` |
| Port 3 | Server B | `30 - SERVERS` |
| Port 4 | Workstation | `20 - TRUSTED` |
| Port 5 | IoT device | `50 - IOT` |
| Port 6–8 | Spare | Default or appropriate profile |

> **Trunk ports must be trunk on both ends.** If port 1 on the downstream switch is set to `TRUNK - ALL` but the corresponding port on the upstream switch is set to an access profile, VLANs will not pass correctly.

---

## Verification

Once configuration is complete, verify each layer is working correctly.

### DNS Resolution

From a client on each VLAN, confirm DNS resolves correctly:

```bash
# Should return an IP via your internal DNS resolver
dig example.com @<dns-server-ip>

# Internal hostname should resolve
dig server01.internal @<dns-server-ip>
```

### Inter-VLAN Connectivity

Test that your firewall policy is enforced:

```bash
# From TRUSTED — should reach servers
ping 10.10.30.10

# From SERVERS — should NOT reach trusted workstations
ping 10.10.20.10   # expect: timeout

# From IOT — should NOT reach any internal host
ping 10.10.20.10   # expect: timeout
ping 10.10.30.10   # expect: timeout

# From any VLAN — internet should work
ping 8.8.8.8
```

### DHCP Leases

Confirm devices are receiving IPs in the correct subnet:

```bash
ip addr show   # Linux
ipconfig       # Windows
```

Check active leases in OPNsense under **Services → Dnsmasq → Leases** — confirm each device's IP falls within its expected VLAN subnet.

### Firewall Logs

Navigate to **Firewall → Log Files → Live View** in OPNsense. Generate some cross-VLAN traffic and confirm block rules are logging hits as expected.

---

## Common Issues

### Device received wrong subnet IP

The port profile on the switch does not match the intended VLAN. Check:

1. The port's assigned profile in Omada
2. That the profile's Native/Untagged network matches the intended VLAN
3. That the VLAN ID in the Omada network matches OPNsense exactly

### No internet from a VLAN

The allow rule for that VLAN is missing or below a block rule that is matching first. Check rule order in **Firewall → Rules → [Interface]**.

### Can't reach OPNsense gateway from a VLAN

The VLAN interface IP is not configured. Check **Interfaces → [VLAN]** and confirm a static IPv4 address is set.

### Cross-VLAN traffic that should be blocked is getting through

A block rule is missing or in the wrong position. Remember: first match wins. Block rules must be above any pass rules that would otherwise match the same traffic.

### Trunk port not passing all VLANs

Either the trunk profile is missing a VLAN in Tagged Networks, or the OPNsense VLAN interface is not assigned/enabled. Check both ends of every trunk link.

---

## Reference

- [OPNsense VLAN Documentation](https://docs.opnsense.org/manual/other-interfaces.html)
- [OPNsense Firewall Rules](https://docs.opnsense.org/manual/firewall.html)
- [OPNsense NAT](https://docs.opnsense.org/manual/nat.html)
- [OPNsense Dnsmasq](https://docs.opnsense.org/manual/dnsmasq.html)
- [TP-Link Omada VLAN Configuration](https://www.tp-link.com/us/support/faq/3089/)
