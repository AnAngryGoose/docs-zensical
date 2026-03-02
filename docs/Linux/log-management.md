---
title: Log Management
description: Reading, searching, and managing system logs on Linux.
---

# Log Management

> **Reference:** [systemd journald Docs](https://www.freedesktop.org/software/systemd/man/latest/journald.conf.html) · [Debian Wiki — syslog](https://wiki.debian.org/syslog) · [Ubuntu Docs — SystemLogs](https://help.ubuntu.com/community/LinuxLogFiles)

---

## Log Locations

Most system logs live in `/var/log/`:

| File | Contents |
|------|----------|
| `/var/log/syslog` | General system messages (Debian/Ubuntu default) |
| `/var/log/auth.log` | Authentication, sudo, SSH logins |
| `/var/log/kern.log` | Kernel messages |
| `/var/log/dmesg` | Boot-time kernel ring buffer |
| `/var/log/dpkg.log` | Package install/remove history |
| `/var/log/apt/history.log` | apt command history |
| `/var/log/nginx/` | Nginx access and error logs |
| `/var/log/journal/` | systemd journal (binary format) |

!!! note "Debian vs Ubuntu"
    Both use systemd journal by default. Debian also writes to `/var/log/syslog` via `rsyslog`. Ubuntu does the same but may vary by release. If `/var/log/syslog` is missing, check if `rsyslog` is installed: `dpkg -l rsyslog`.

---

## Reading Logs

```bash
# View a log file
cat /var/log/syslog
less /var/log/syslog       # paginated, searchable with /

# Follow live (like tail -f)
tail -f /var/log/syslog
tail -f /var/log/auth.log

# Last 50 lines
tail -n 50 /var/log/syslog

# Search within a log
grep "error" /var/log/syslog
grep -i "fail" /var/log/auth.log     # case-insensitive
grep "ssh" /var/log/auth.log | tail -20

# View kernel ring buffer (hardware/boot messages)
dmesg
dmesg | tail -20
dmesg | grep -i error
dmesg -T             # with human-readable timestamps
```

---

## journalctl — systemd Journal

The journal is the primary log store on modern Debian/Ubuntu. See also the [Systemd page](systemd.md#journalctl-log-viewing) for more detail.

```bash
# All logs
journalctl

# Follow live
journalctl -f

# Last 100 lines
journalctl -n 100

# Logs for a specific service
journalctl -u nginx
journalctl -u docker -f

# Logs since last boot
journalctl -b

# Logs from a specific time window
journalctl --since "2024-01-15 09:00:00" --until "2024-01-15 10:00:00"
journalctl --since "1 hour ago"
journalctl --since today

# Filter by priority
journalctl -p err          # errors and above
journalctl -p warning      # warnings and above
journalctl -p debug        # everything

# Priority levels: emerg, alert, crit, err, warning, notice, info, debug

# Kernel messages only
journalctl -k

# Show full output (no truncation)
journalctl --no-pager

# Output as JSON (useful for log forwarding)
journalctl -u myservice -o json-pretty
```

---

## Log Rotation — logrotate

`logrotate` automatically rotates, compresses, and removes old log files.

```bash
# Config files
/etc/logrotate.conf          # global defaults
/etc/logrotate.d/            # per-application configs

# Test a config (dry run)
sudo logrotate --debug /etc/logrotate.d/nginx

# Force run logrotate now
sudo logrotate -f /etc/logrotate.conf
```

### Example Config

```bash
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily               # rotate daily
    missingok           # don't error if log file missing
    rotate 14           # keep 14 rotated logs
    compress            # gzip old logs
    delaycompress       # compress on second rotation (app may still write)
    notifempty          # don't rotate empty files
    create 0640 myapp myapp   # create new log with these perms/owner
    postrotate
        systemctl reload myapp > /dev/null 2>&1 || true
    endscript
}
```

---

## journald Configuration

```bash
# Config file
sudo nano /etc/systemd/journald.conf

# Key settings:
[Journal]
# Limit journal disk usage
SystemMaxUse=500M

# Keep logs for max 30 days
MaxRetentionSec=30day

# Max size per log file
SystemMaxFileSize=50M

# Persist journal across reboots (default is sometimes volatile)
Storage=persistent
```

```bash
# Apply changes
sudo systemctl restart systemd-journald

# Check current disk usage
journalctl --disk-usage

# Manual cleanup
sudo journalctl --vacuum-size=200M    # trim to under 200MB
sudo journalctl --vacuum-time=14d     # remove logs older than 14 days
```

---

## Searching & Filtering Logs

```bash
# grep with context lines
grep -A 3 -B 3 "Out of memory" /var/log/syslog
# -A 3 = 3 lines after match
# -B 3 = 3 lines before match
# -C 3 = 3 lines either side

# Search across multiple log files
grep -r "connection refused" /var/log/

# Search with timestamps visible
grep "Jan 15" /var/log/syslog | grep "error"

# Count occurrences
grep -c "Failed password" /var/log/auth.log

# Show failed SSH attempts with source IPs
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn

# Show sudo usage
grep "sudo:" /var/log/auth.log
```

---

## Forwarding Logs (Loki / Remote Syslog)

### Docker Logging Driver → Loki

Add to `/etc/docker/daemon.json` to send all container logs to Loki without changing compose files:

```json
{
  "log-driver": "loki",
  "log-opts": {
    "loki-url": "http://localhost:3100/loki/api/v1/push",
    "loki-batch-size": "400",
    "loki-retries": "3",
    "loki-max-backoff": "800ms",
    "loki-timeout": "2s"
  }
}
```

```bash
# Install the Loki Docker driver plugin first
docker plugin install grafana/loki-docker-driver:latest \
    --alias loki \
    --grant-all-permissions

# Restart Docker to apply
sudo systemctl restart docker
```

### Remote Syslog Forwarding (rsyslog)

```bash
# Forward to remote syslog server
sudo nano /etc/rsyslog.conf

# Add at bottom:
*.* @@192.168.1.50:514    # TCP (@@)
*.* @192.168.1.50:514     # UDP (@)
```

---

## Audit Log (auditd)

For tracking file access, user commands, or privilege escalation:

```bash
sudo apt install auditd

# Watch a file for reads/writes
sudo auditctl -w /etc/passwd -p rwa -k passwd-watch

# View audit log
sudo ausearch -k passwd-watch
sudo aureport --summary
```