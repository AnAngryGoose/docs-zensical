# OPNsense DNS Setup — AdGuard + Unbound + Split DNS

## Goal
- Network-wide filtering via AdGuard Home
- `.internal` hostname resolution for local machines
- Local services at `*.home.mydomain.me` via NPM
- Public subdomains at `*.mydomain.me` resolve normally

---

## Architecture

```
Clients → AdGuard Home (:53) → Unbound (:5335) → Quad9 DoT (9.9.9.9:853)
```

DHCP hands out AdGuard's IP as the DNS server for all clients.

---

## 1. Unbound — Services → Unbound DNS → General

- **Port:** `5335`
- **Domain:** `internal`
- **Enable DHCP lease registration:** yes
- **Enable DHCP static mapping registration:** yes
- **DNS over TLS upstream:** `9.9.9.9` port `853`, server name `dns.quad9.net`
- **Query forwarding:** disabled

> Unbound handles `.internal` authoritatively via DHCP registration. No need for DNSMasq in the DNS chain.

---

## 2. AdGuard Home — Settings → DNS Settings

- **Upstream DNS:** `127.0.0.1:5335`
- **Bootstrap DNS:** `9.9.9.10`, `149.112.112.10`
- **Fallback DNS:** empty
- **Listen on:** port `53`

> Must use `127.0.0.1:5335` — not the hostname or LAN IP. AdGuard can't resolve its own upstream if it depends on itself to do so.

---

## 3. Split DNS — AdGuard → Filters → DNS Rewrites

Add a single wildcard entry:

| Domain | Answer |
|--------|--------|
| `*.home.mydomain.me` | `<NPM local IP>` |

All `*.home.mydomain.me` queries resolve to NPM. NPM routes by hostname to the correct service. Any subdomain not configured in NPM returns a 404 from NPM — it never hits the public internet.

Public subdomains (`docs.mydomain.me`, etc.) are not covered by the wildcard and resolve normally via Unbound → Quad9 → Cloudflare.

---

## 4. DHCP

Set DNS server in **Services → DHCPv4 → LAN → DNS Servers** to AdGuard's IP (same as OPNsense LAN IP if running on the firewall).

---

## 5. DNSMasq

DNS functionality disabled. Can remain active for **DHCP only** — this is the preferred option as ISC DHCP is deprecated in OPNsense.

- **Services → DHCPv4** — DNSMasq handles lease assignment here, this is fine
- **Services → Dnsmasq DNS** — must be disabled, otherwise it binds to port 53 and conflicts with AdGuard

Unbound reads DNSMasq's lease file automatically via the "Register DHCP leases" checkbox — no extra config needed.


---

## Verification

This script will confirm client DNS config, local hostname resolution via AdGuard and Unbound, DoT activity, filtering, local service resolution via NPM, and public subdomain resolution. Replace `<opnsense-ip>` with the actual IP address of your OPNsense firewall.

```bash
# Confirm client DNS config
resolvectl status

# Local hostname via AdGuard
dig mymachine.internal @<opnsense-ip>

# Local hostname direct to Unbound
dig mymachine.internal @<opnsense-ip> -p 5335

# Confirm DoT active (expect: "dot.")
dig txt proto.on.quad9.net @<opnsense-ip> +short

# Confirm filtering active (expect: 0.0.0.0)
dig doubleclick.net @<opnsense-ip> +short

# Local service via NPM
dig service.home.mydomain.me @<opnsense-ip> +short

# Public subdomain still resolves externally
dig docs.mydomain.me @<opnsense-ip> +short
```