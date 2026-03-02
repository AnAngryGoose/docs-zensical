---
title: Systemd
description: Managing services, units, and the journal with systemd.
---

# Systemd

> **Reference:** [systemd Docs](https://systemd.io/) · [Freedesktop systemd Man Pages](https://www.freedesktop.org/software/systemd/man/latest/) · [Debian Wiki — systemd](https://wiki.debian.org/systemd)

---

## systemctl — Service Management

### Common Commands

```bash
# Start / stop / restart a service
sudo systemctl start servicename
sudo systemctl stop servicename
sudo systemctl restart servicename

# Reload config without full restart (if supported)
sudo systemctl reload servicename

# Enable at boot / disable
sudo systemctl enable servicename
sudo systemctl disable servicename

# Enable and start in one command
sudo systemctl enable --now servicename

# Check status
systemctl status servicename

# Is it running?
systemctl is-active servicename

# Is it enabled at boot?
systemctl is-enabled servicename
```

---

### System State

```bash
# List all running services
systemctl list-units --type=service --state=running

# List all failed units
systemctl --failed

# List all units (running, stopped, failed)
systemctl list-units --type=service

# Reboot / shutdown / halt
sudo systemctl reboot
sudo systemctl poweroff
sudo systemctl halt
```

---

## journalctl — Log Viewing

`journalctl` queries the systemd journal — the central log store for all services managed by systemd.

```bash
# View all logs (oldest first)
journalctl

# Follow live (like tail -f)
journalctl -f

# Logs for a specific service
journalctl -u servicename

# Follow logs for a service live
journalctl -fu servicename

# Last 50 lines for a service
journalctl -u servicename -n 50

# Since last boot
journalctl -b

# Logs from a previous boot (-1 = one boot ago)
journalctl -b -1

# Logs since a specific time
journalctl --since "2024-01-01 12:00:00"
journalctl --since "1 hour ago"

# Filter by priority (0=emerg, 3=err, 4=warning, 6=info, 7=debug)
journalctl -p err

# Show kernel messages
journalctl -k

# Disk space used by journal
journalctl --disk-usage

# Vacuum journal to limit size
sudo journalctl --vacuum-size=500M
sudo journalctl --vacuum-time=7d
```

---

## Writing a Custom Service Unit

Service unit files live in `/etc/systemd/system/`. Units here override anything in `/lib/systemd/system/`.

### Simple Service Example

```ini
# /etc/systemd/system/myapp.service

[Unit]
Description=My Application
Documentation=https://example.com/docs
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=myappuser
Group=myappuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/myapp --config /etc/myapp/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

# Environment variables
Environment=NODE_ENV=production
# Or load from file
EnvironmentFile=/etc/myapp/.env

[Install]
WantedBy=multi-user.target
```

```bash
# After creating the file, reload systemd and enable
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
```

---

### Service Types

| Type | Use Case |
|------|----------|
| `simple` | Process stays in foreground (most common) |
| `forking` | Process forks and parent exits (traditional daemons) |
| `oneshot` | Runs once and exits (scripts, setup tasks) |
| `notify` | Like simple but process signals when ready |
| `idle` | Like simple but waits until other jobs finish |

---

### Restart Policies

```ini
Restart=no           # Never restart (default)
Restart=on-failure   # Restart only on non-zero exit
Restart=always       # Always restart
Restart=on-abnormal  # Restart on signal, timeout, watchdog
```

---

## Timer Units (Systemd Cron Alternative)

Timers are the systemd alternative to cron. They require two files: a `.timer` and a matching `.service`.

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Run Backup Script

[Service]
Type=oneshot
User=youruser
ExecStart=/usr/local/bin/backup.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily Backup Timer
Requires=backup.service

[Timer]
OnCalendar=daily
OnCalendar=*-*-* 02:00:00   # Every day at 2am
Persistent=true              # Run if missed (e.g. system was off)

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer

# Check timer status and next run time
systemctl list-timers
```

---

## Targets (Runlevels)

Targets group units together — roughly equivalent to SysV runlevels.

```bash
# View current target
systemctl get-default

# Set default target
sudo systemctl set-default multi-user.target   # no GUI
sudo systemctl set-default graphical.target    # with GUI

# Common targets
# multi-user.target  → like runlevel 3 (multi-user, no GUI)
# graphical.target   → like runlevel 5 (multi-user + GUI)
# rescue.target      → single user / rescue mode
# emergency.target   → minimal emergency shell
```

---

## Quick Troubleshooting

```bash
# Service won't start — see why
sudo systemctl status servicename
journalctl -u servicename -n 50 --no-pager

# Find unit file location
systemctl cat servicename

# Show all properties of a unit
systemctl show servicename

# Check for dependency issues
systemctl list-dependencies servicename
```