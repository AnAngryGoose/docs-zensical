---
title: DNS Stack — AdGuard + Unbound + dnsmasq
description: Setting up a three-tier DNS stack on OPNsense using AdGuard Home for filtering, Unbound for recursive resolution, and dnsmasq for DHCP and local hostname registration.
tags:
  - networking
  - dns
  - opnsense
  - adguard
  - unbound
  - dnsmasq
---

# DNS Stack — AdGuard + Unbound + dnsmasq

This guide covers setting up a layered DNS stack on OPNsense. Each service has a distinct role and they chain together so that all client DNS traffic is filtered, recursively resolved, and locally registered.

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

**Why three services?**

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

> **Warning — DNS port must not be set to `0`.** Port `0` disables dnsmasq DNS entirely. Unbound's Query Forwarding entries point to dnsmasq at port `53053` — if that port is `0`, Unbound forwards `.internal` queries to nothing and all local hostname resolution silently breaks. The DNS port field must always be set to `53053` (or whatever non-zero port you chose). This is a common misconfiguration that produces confusing symptoms because DHCP continues to work while DNS fails.

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

> **Decision rule:** Use dnsmasq Hosts for DHCP-managed devices. Use Unbound Overrides only for devices with static IPs configured on the device itself (no DHCP), or for custom DNS aliases.

---

### DHCP Ranges

**Services → Dnsmasq DNS & DHCP → DHCP ranges**

Add one range per VLAN. Static reservations defined in Hosts sit outside the pool by MAC binding — they do not need to be excluded from the range manually.

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

> Use the gateway IP of the VLAN the client is on, not a fixed IP — once OPNsense has VLAN interfaces each will have its own IP and AdGuard listens on all of them.

---

### DHCP Options — Omada Controller Discovery

If you are running Omada managed switches or APs on a dedicated MANAGEMENT VLAN, add an additional DHCP option on the MANAGEMENT interface so devices can locate the Omada controller after a reboot.

| Field | Value |
|---|---|
| Interface | MANAGEMENT |
| Option | `option_capwap_ac_v4 [138]` |
| Value | IP address of your Omada controller |

> This is required because broadcast-based controller discovery does not work across VLANs. Without Option 138, Omada devices survive a single session but become orphaned after rebooting because they cannot find the controller on a different subnet.

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

This is the critical link. Unbound forwards `.internal` queries to dnsmasq instead of trying to resolve them recursively (which would fail — `.internal` is not a real TLD).

| Domain | Server IP | Port | Description |
|---|---|---|---|
| `internal` | `127.0.0.1` | `53053` | Forward local hostname lookups to dnsmasq |
| `x.x.x.in-addr.arpa` | `127.0.0.1` | `53053` | Forward reverse PTR lookups to dnsmasq |

> Replace `x.x.x` with the reverse zone for your subnet. For `10.10.30.0/24` this is `30.10.10.in-addr.arpa`. Add one PTR entry per subnet.

Without these entries, Unbound attempts to resolve `hostname.internal` from root DNS servers, fails, and returns NXDOMAIN.

> **These Query Forwarding entries must exist and must point to the correct port.** If dnsmasq DNS is running on port `53053` but the Query Forwarding entries are missing or point to a different port, `.internal` resolution will fail silently — DHCP will still work, but hostnames will not resolve.

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

AdGuard sits in front of everything. Clients query it on port 53. It filters, then forwards upstream.

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

### DNS broke after editing dnsmasq settings

Check the DNS listen port field immediately. It is easy to accidentally set this to `0` while editing other settings. Port `0` disables dnsmasq DNS entirely, breaking the Unbound → dnsmasq chain and causing all `.internal` resolution to fail. Set it back to `53053` and restart dnsmasq.

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

### Docker containers cannot resolve .internal hostnames

Docker containers use their own DNS configuration and do not automatically use the host's DNS settings. Add the VLAN gateway as an explicit DNS server in the container's compose file:

```yaml
services:
  myservice:
    dns:
      - 10.10.30.1   # VLAN gateway IP — replace with your LAB/SERVERS gateway
```

Without this, containers query Docker's internal DNS (`127.0.0.11`) which cannot resolve `.internal` hostnames.

### Rogue DNS bypass not being caught

Verify NAT port forward rules exist for each VLAN under **Firewall → NAT → Port Forward** and that the destination invert is checked.