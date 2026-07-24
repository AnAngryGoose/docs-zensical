---
icon: simple/adguard
title: AdGuard Home
---

# AdGuard Home on OPNsense

AdGuard Home is the **client-facing** DNS server for the network — every device
queries it on port 53, it filters ads/trackers/malware, then forwards clean
queries upstream to [Unbound](opnsense-dns-stack.md). This page covers installing
and configuring the AdGuard Home plugin; for how it slots into the wider resolver
chain see the [DNS Stack](opnsense-dns-stack.md) guide.

!!! info "Where AdGuard fits"

    ```
    Client → AdGuard Home :53 → Unbound :5335 → dnsmasq (.internal) / root DNS
             (filtering)         (recursion/DNSSEC)   (local names)
    ```

    AdGuard only **filters and forwards** — it never resolves recursively itself.
    That's Unbound's job.

There's also a thorough external write-up worth keeping handy:
[windgate.net — AdGuard Home on OPNsense](https://windgate.net/setup-adguard-home-opnsense-adblocker/).

## 1. Install the plugin

The AdGuard Home plugin lives in the community repository.

1. **System → Firmware → Settings** → set **Community (unsupported) plugins** and save.
2. **System → Firmware → Plugins** → search **`os-adguardhome-maxit`** → click **+** to install.
3. After it installs, a new **Services → AdGuardHome** menu appears.

```bash
# (Alternative) install from the console / SSH
pkg install os-adguardhome-maxit
```

## 2. Free up port 53

AdGuard needs to bind port 53 on the interfaces clients use. Two things commonly
squat on 53:

- **Unbound** — move it to `5335` (**Services → Unbound DNS → General → Listen Port**).
- **dnsmasq** — if used for DHCP, move its DNS to `53053` and let AdGuard own 53.

See the [DNS Stack](opnsense-dns-stack.md) page — this repo runs AdGuard on 53,
Unbound on 5335, dnsmasq on 53053 so nothing collides.

## 3. Enable and run the setup wizard

1. **Services → AdGuardHome → General** → **Enable**, then **Apply**.
2. Open the dashboard (a link is on the plugin page, or `http://<opnsense-ip>:3000`).
3. Run the first-time wizard:
   - **Admin web interface:** an unused port (e.g. `3000`) on All interfaces.
   - **DNS server:** port `53` on the interfaces clients query (your VLAN gateways,
     or All interfaces).
   - Create the admin account.

## 4. Point AdGuard at Unbound

**Settings → DNS settings → Upstream DNS servers:**

```
127.0.0.1:5335
```

That's it — a single upstream. Everything (public and `.internal`) goes to Unbound,
which decides whether to resolve recursively or hand `.internal` to dnsmasq.

**Bootstrap DNS servers** (used only to look up upstreams at startup):

```
127.0.0.1
9.9.9.9
```

!!! tip "Test upstreams"

    Use the **Test upstreams** button. If it fails, Unbound isn't listening on
    `5335` or a firewall rule is blocking loopback — fix that before going further.

## 5. Filtering (blocklists)

**Filters → DNS blocklists → Add blocklist.** AdGuard ships with **AdGuard DNS
filter** enabled. Sensible additions:

- **AdGuard DNS filter** (default) — general ads/trackers
- **OISD Big** — large, well-maintained, low false positives
- **HaGeZi Multi Pro** — aggressive ads + tracking + malware

!!! warning "Don't over-block"

    Stacking many aggressive lists breaks logins, shortened links, and shopping
    sites. Start with one or two, then add per-domain **allowlist** entries under
    **Filters → Custom filtering rules** (`@@||domain^`) as things break.

## 6. Local domains — DNS Rewrites

To make internal services (e.g. `*.vanth.me` pointing at Nginx Proxy Manager)
resolve on the LAN and over Tailscale:

**Filters → DNS Rewrites → Add DNS rewrite:**

| Field | Value |
|---|---|
| Domain | `*.vanth.me` (wildcard covers every subdomain) |
| Answer | NPM's LAN IP (e.g. `10.10.30.x`) |

See [Tailscale → Split DNS](opnsensetailscale.md#configure-split-dns) for using
this to resolve local domains remotely.

## 7. Per-client settings (optional)

**Settings → Client settings** lets you treat devices differently by IP/MAC —
e.g. stricter blocking for kids' devices, or disabling filtering for a smart TV
that misbehaves. Clients show up by IP; give them names here for readable logs.

## 8. Enforce it — Force DNS via NAT

Filtering only works if clients actually use AdGuard. A device with a hardcoded
`8.8.8.8` bypasses everything. Redirect all rogue DNS back to AdGuard with a NAT
port-forward per VLAN — see [DNS Stack → Force DNS via NAT](opnsense-dns-stack.md#step-4-force-dns-via-nat).

## Maintenance

- **Query Log** (dashboard) — see exactly what each client asks for and what got
  blocked; the fastest way to diagnose "site won't load."
- **Clear cache** — **Settings → DNS settings → Clear cache** after changing rewrites
  or when stale records linger.
- **Back it up** — AdGuard config lives at
  `/usr/local/AdGuardHome/AdGuardHome.yaml`; include it in your OPNsense
  [config backups](maintenance.md#configuration-backup) or a file backup.

## Sources

- [AdGuard Home Wiki](https://github.com/AdguardTeam/AdGuardHome/wiki)
- [windgate.net — AdGuard Home on OPNsense](https://windgate.net/setup-adguard-home-opnsense-adblocker/)
- [OPNsense Unbound Documentation](https://docs.opnsense.org/manual/unbound.html)
