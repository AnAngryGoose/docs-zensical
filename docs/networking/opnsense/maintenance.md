---
icon: lucide/save
title: Backups & Maintenance
description: Backing up and restoring OPNsense config, firmware updates, plugins, and health.
---

# Backups, Updates & Maintenance

The router is the one box where a bad change takes the whole network down. A recent
config backup turns "rebuild from scratch" into "restore and reboot." This page
covers backing up, updating, and keeping OPNsense healthy.

## Configuration backup

The entire OPNsense config is a single XML file — no databases, no scattered state.

**System → Configuration → Backups.**

- **Download** — saves `config-<hostname>-<date>.xml`. Do this **before every
  significant change** (firewall rules, interface changes, updates).
- **Encrypt** — tick the box and set a password; the download is encrypted. Do this
  if you store backups anywhere off-box — the config contains secrets (VPN keys,
  RADIUS passwords, certificates).
- **Restore** — upload a backup on the same page. You can restore **specific areas**
  (just firewall, just interfaces) from the "Restore areas" dropdown, or the whole
  config. A full restore reboots.

!!! tip "Automate off-box backups"

    OPNsense can push encrypted backups automatically. Install one of the backup
    plugins (**System → Firmware → Plugins**) and configure it under
    **System → Configuration → Backups**:

    - `os-git-backup` — commit the config to a git repo (e.g. your
      [Forgejo](../../utilities/git/git.md)) on every change. Great audit trail.
    - Nextcloud / Google Drive backup options also exist.

!!! danger "Config XML contains secrets"

    Treat a plain (unencrypted) config export like a password file — it holds
    private keys and credentials. Encrypt it, and never commit an unencrypted one to
    a public repo.

## Snapshots (ZFS)

If you installed OPNsense on **ZFS** (the default in recent versions), it takes a
**boot environment** snapshot before firmware updates. If an update goes wrong, pick
the previous boot environment at the boot menu to roll back the entire OS — separate
from, and complementary to, config backups.

**System → Snapshots** manages them (ZFS installs only).

## Firmware updates

**System → Firmware → Status.**

```text
Check for updates → review changelog → Update
```

- **Minor updates** (e.g. 24.7.1 → 24.7.2) — routine; apply after reading the notes.
  Some require a reboot (it tells you).
- **Major upgrades** (e.g. 24.7 → 25.1) — released twice a year. **Back up first**,
  read the release notes for breaking changes, and do it when you can tolerate a
  reboot. Use the **Upgrade** button that appears when a major release is available.
- **Plugins** update from the same page (**Plugins** tab) — keep `os-*` plugins
  current with the base system.

!!! tip "Update during a maintenance window"

    An update reboots the firewall — every network drops for a minute or two. Don't
    do it remotely without an out-of-band way back in (Tailscale on a separate host,
    or physical access).

## Health & monitoring

- **Dashboard** — at-a-glance CPU, memory, interfaces, gateway status, services.
- **Reporting → Insight** — long-term traffic graphs per interface.
- **Reporting → Health** — system metrics (CPU, memory, disk, temperature) over time.
- **Firewall → Log Files → Live View** — real-time allow/deny
  ([see Firewall](firewall-aliases.md#logging-states)).
- **Interfaces → Overview** — link state, throughput, and the Tailscale interface IP.

For homelab-wide monitoring, OPNsense exposes metrics that Beszel/Uptime-Kuma or a
Prometheus exporter plugin can scrape — but its own dashboard covers day-to-day.

## Console menu (out-of-band)

Physical console or SSH gives the emergency menu — useful when the web UI is
unreachable:

| Option | Does |
|---|---|
| `1` Assign interfaces | Fix a wrong WAN/LAN assignment |
| `2` Set interface IP | Reset a LAN IP so you can reach the web UI |
| `4` Reset to factory defaults | Last resort |
| `7` Ping host | Basic connectivity test |
| `8` Shell | Full root shell |
| `11` Restart web UI | When the GUI hangs but the box is up |
| `12` Update from console | Firmware update without the web UI |

## A sane maintenance rhythm

- **Before any change:** download a config backup (or rely on `os-git-backup`).
- **Monthly:** apply minor firmware + plugin updates during a quiet window.
- **Twice a year:** plan the major-version upgrade after reading release notes.
- **Off-box:** keep at least one recent **encrypted** backup somewhere that survives
  the router dying.

## Reference

- [OPNsense — Backups](https://docs.opnsense.org/manual/backups.html)
- [OPNsense — Updates](https://docs.opnsense.org/manual/updates.html)
- [OPNsense — Boot Environments](https://docs.opnsense.org/manual/bootenvironments.html)
