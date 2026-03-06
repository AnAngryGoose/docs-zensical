---
title: DNS Stack — AdGuard + Unbound + dnsmasq
  - networking
  - dns
  - opnsense
  - adguard
  - unbound
  - dnsmasq
---

# DNS Stack — AdGuard + Unbound + dnsmasq

This guide covers setting up a layered DNS stack on OPNsense. Each service has a distinct role and they chain together so that all client DNS traffic is filtered, recursively resolved, and locally registered.

I am only using IPv4 for this guide. Not familiar enough with IPv6, and I don't feel like dealing with for now. I'll come around to it (maybe).

This is made for DNSMasq and Unbound DNS specifically, but same ideas apply if using something else. Same for omada controller (for the most part) 

## References

- [OPNsense Dnsmasq Documentation](https://docs.opnsense.org/manual/dnsmasq.html)
- [OPNsense Unbound Documentation](https://docs.opnsense.org/manual/unbound.html)
- [How to Migrate from ISC DHCP to dnsmasq or Kea DHCP in OPNsense](https://homenetworkguy.com/how-to/migrate-from-isc-dhcp-to-dnsmasq-or-kea-dhcp-in-opnsense/)

---

## Architecture

```
Client
  └─► AdGuard Home :53        — ad/tracker filtering, client-facing DNS
        └─► Unbound :5335     — recursive resolver, DNSSEC, caching
              └─► dnsmasq :53053   — local hostname resolution (.internal)
              └─► Root DNS servers — all other queries
```

**Purpose of each service:**

- **AdGuard** — filtering layer. Sits in front so every query is inspected before resolution. Cannot do recursive resolution itself.
- **Unbound** — recursive resolver. Resolves public DNS from root servers directly (no third-party upstream needed). Forwards internal domain queries to dnsmasq.
- **dnsmasq** — DHCP server and local DNS registry. Registers DHCP hostnames automatically so `hostname.internal` resolves without manual DNS entries. Cannot resolve public DNS itself — it needs Unbound as its upstream.

None of these roles overlap. Each is doing something the others cannot.

---

## Step 1 — dnsmasq

**Services → Dnsmasq DNS & DHCP → General**

| Setting | Value | Why |
|---|---|---|
| Enable | **Enabled** | |
| Interface | All VLANs + LAN | Listen for DHCP on all networks |
| DNS Listen port | `53053` | Non-standard port — Unbound forwards here, clients never query dnsmasq directly |
| DHCP FQDN | **Enabled** | Registers fully qualified names (e.g. `host.internal`) not just bare hostnames |
| DHCP default domain | `internal` | Appended to all DHCP hostnames — `server01` becomes `server01.internal` |
| DHCP local domain | **Enabled** | Makes dnsmasq authoritative for `.internal` — it will not forward these queries upstream |
| Do not forward to system defined DNS | **Enabled** | Prevents query loops — dnsmasq must not forward back to Unbound for local lookups |

!!!tip
    **dnsmasq DNS is enabled here on port 53053.** It is not disabled. Port 0 would disable DNS entirely and break the chain — Unbound would forward `.internal` queries to nothing.

---

### Static Host Reservations

**Services → Dnsmasq DNS & DHCP → Hosts**

Add one entry per static host. A single entry handles both DHCP reservation and DNS registration.

| Field | Value |
|---|---|
| Host | Hostname only (e.g. `server01`) |
| Domain | `internal` |
| IP Address | Static IP (e.g. `10.10.30.20`) |
| Hardware Address | Device MAC address |

This gives you three things from one entry:

1. DHCP always assigns this IP to this MAC
2. `server01` resolves to the IP
3. `server01.internal` resolves to the IP

!!!tip
    Use dnsmasq Hosts for DHCP-managed devices. Use Unbound Overrides only for devices with static IPs configured on the device itself (no DHCP), or for custom DNS aliases.

!!!tip
    **DHCP Static Reservation** = Assigns a static IP to MAC address

    **DNS Override** = Assigns a DNS address to an IP address.

    The DNS record must match the DHCP Reservation if it is set, or it will not resolve. 
---

### DHCP Ranges

**Services → Dnsmasq DNS & DHCP → DHCP ranges**

Add one range per VLAN. Static reservations defined in Hosts sit outside the pool by MAC binding — they do not need to be excluded from the range manually.

!!!note
    Reservations will reserve the IP address inside a range, meaning the reserved IP will not be offered to dynamic clients.

    A dynamic range like 192.168.1.100-192.168.1.199 and a reservation like 192.168.1.101 are valid and there will be no collisions.

    The reservation can also be outside the dynamic range, but it is not recommended for simple setups as the dynamic dns registration with dhcp-fqdn will not work correctly

| Field | Value |
|---|---|
| Interface | VLAN interface |
| Start | e.g. `10.10.30.100` |
| End | e.g. `10.10.30.200` |

---

### DHCP Options — Push DNS to Clients

**Services → Dnsmasq DNS & DHCP → DHCP options**

Clients need to receive AdGuard's IP as their DNS server via DHCP. Add one entry per VLAN:

| Field | Value |
|---|---|
| Interface | VLAN interface |
| Option | `6` (DNS server) |
| Value | IP of OPNsense interface where AdGuard listens (e.g. `10.10.10.1`) |

!!!tip 
    Use the gateway IP of the VLAN the client is on, not a fixed IP — once OPNsense has VLAN interfaces each will have its own IP and AdGuard listens on all of them.

---

## Step 2 — Unbound

**Services → Unbound DNS → General**

| Setting | Value | Why |
|---|---|---|
| Enable | **Enabled** | |
| Listen Port | `5335` | Non-standard port — AdGuard forwards here, clients never query Unbound directly |
| DHCP Domain Override | `internal` | Tells Unbound which domain suffix to use when registering dnsmasq leases |
| Register DHCP Static Mappings | **Enabled** | Imports dnsmasq Host entries into Unbound's awareness |
| Register ISC DHCP4 Leases | **Enabled** | Registers active DHCP leases for dynamic hostname resolution |
| Flush DNS Cache during reload | **Enabled** | Clears stale records on every restart — important when IPs change |
| Local Zone Type | `transparent` | Passes `.internal` queries through to dnsmasq rather than blocking them |
| Enable DNSSEC | **Enabled** | Validates DNS responses from upstream — Unbound is the right place to terminate DNSSEC |

---

### Query Forwarding

**Services → Unbound DNS → Query Forwarding**

!!!warning

 Unbound forwards `.internal` queries to dnsmasq instead of trying to resolve them recursively (which would fail — `.internal` is not a real TLD).

| Domain | Server IP | Port | Description |
|---|---|---|---|
| `internal` | `127.0.0.1` | `53053` | Forward local hostname lookups to dnsmasq |
| `x.x.x.in-addr.arpa` | `127.0.0.1` | `53053` | Forward reverse PTR lookups to dnsmasq |

!!!warning
    Replace `x.x.x` with the reverse zone for your subnet. For `10.10.30.0/24` this is `30.10.10.in-addr.arpa`. Add one PTR entry per subnet.

    Without these entries, Unbound attempts to resolve `hostname.internal` from root DNS servers, fails, and returns NXDOMAIN.

---

### Host Overrides

**Services → Unbound DNS → Overrides**

Use sparingly. Unbound Overrides are for:

- Devices with static IPs set on the device itself (not via DHCP) — dnsmasq never sees them
- Custom DNS aliases or CNAMEs (e.g. `omada.internal` → `prod-deb-01.internal`)
- Any record that needs to exist regardless of DHCP lease state

Do **not** duplicate entries that already exist as dnsmasq Hosts — they will conflict.

---

## Step 3 — AdGuard Home

**Services → Adguardhome**

AdGuard sits in front of everything. Clients query it on port 53 (DNS port). It filters, then forwards upstream to Unbound on 5335.

**Settings → DNS settings → Upstream DNS servers:**

```
127.0.0.1:5335
```

This points AdGuard at Unbound. All queries — local and public — go through Unbound. Unbound then decides whether to forward to dnsmasq (`.internal`) or resolve recursively (everything else).

**Settings → DNS settings → Bootstrap DNS:**

```
1.1.1.1
9.9.9.9
```

Used only to resolve the upstream DNS server address itself at startup. Not used for client queries.

**Settings → DNS settings → Clear cache** — use this when stale records persist after configuration changes.

---

## Step 4 — Force DNS via NAT

Prevents clients from bypassing AdGuard by pointing at a public resolver directly (e.g. `8.8.8.8`).

**Firewall → NAT → Port Forward** — add one rule per VLAN:

| Field | Value |
|---|---|
| Interface | VLAN interface |
| Protocol | TCP/UDP |
| Source | VLAN net |
| Destination / Invert | **Enabled** — enter AdGuard IP |
| Destination Port | `53` |
| Redirect Target IP | AdGuard IP (OPNsense VLAN gateway) |
| Redirect Target Port | `53` |
| Description | Force DNS — [VLAN NAME] |

Inverting the destination means: redirect any DNS query NOT already going to AdGuard. Queries going to AdGuard pass through normally. Rogue queries get silently redirected.

---

## Verification

After configuring all three services, verify each layer independently.

**Test dnsmasq directly:**
```bash
dig hostname.internal @<opnsense-ip> -p 53053
```

**Test Unbound directly (bypasses AdGuard):**
```bash
dig hostname.internal @<opnsense-ip> -p 5335
```

**Test full chain through AdGuard:**
```bash
dig hostname.internal @<opnsense-ip>
```

**Test public resolution:**
```bash
dig example.com @<opnsense-ip>
```

All four should return correct answers. If dnsmasq works but Unbound doesn't, the Query Forwarding entries are missing or wrong. If Unbound works but AdGuard doesn't, the upstream DNS setting in AdGuard is incorrect.

---

## Troubleshooting AKA "It'll be better a different way and I break my fucking internet (again)"

### NXDOMAIN for local hostnames

Check in order:

1. Does the dnsmasq Host entry exist with correct MAC and IP?
2. Is dnsmasq DNS listen port set to `53053` (not `0`)?
3. Do the Unbound Query Forwarding entries point to `127.0.0.1:53053`?
4. Is AdGuard upstream set to `127.0.0.1:5335`?

### Stale DNS record after IP change

Clear cache at every layer in order:

1. AdGuard → Settings → DNS settings → Clear cache
2. Restart Unbound (flushes cache on reload if enabled)
3. Client: `sudo systemctl restart systemd-resolved` (Linux) or `ipconfig /flushdns` (Windows)

### Client not receiving correct DNS via DHCP

Check **Services → Dnsmasq DNS & DHCP → DHCP options** — confirm option `6` is set for the client's VLAN interface with the correct AdGuard IP.

Verify what DNS the client actually received:
```bash
resolvectl status          # Linux
ipconfig /all              # Windows
```

### Rogue DNS bypass not being caught

Verify NAT port forward rules exist for each VLAN under **Firewall → NAT → Port Forward** and that the destination invert is checked.
