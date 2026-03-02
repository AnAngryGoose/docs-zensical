---
title: Cron & Scheduling
description: Scheduling tasks with cron, crontab, and systemd timers.
---

# Cron & Scheduling

> **Reference:** [Debian Wiki — cron](https://wiki.debian.org/cron) · [Ubuntu Docs — CronHowto](https://help.ubuntu.com/community/CronHowto) · [man 5 crontab](https://manpages.debian.org/crontab.5)

---

## Cron Syntax

```
┌───────── minute        (0–59)
│ ┌─────── hour          (0–23)
│ │ ┌───── day of month  (1–31)
│ │ │ ┌─── month         (1–12 or jan–dec)
│ │ │ │ ┌─ day of week   (0–7, 0 and 7 = Sunday, or sun–sat)
│ │ │ │ │
* * * * *  command
```

### Common Examples

```bash
# Every minute
* * * * * /usr/local/bin/script.sh

# Every day at 2am
0 2 * * * /usr/local/bin/backup.sh

# Every Monday at 6am
0 6 * * 1 /usr/local/bin/weekly.sh

# Every 15 minutes
*/15 * * * * /usr/local/bin/check.sh

# First day of every month at midnight
0 0 1 * * /usr/local/bin/monthly.sh

# Weekdays at 8am
0 8 * * 1-5 /usr/local/bin/workday.sh

# Every 6 hours
0 */6 * * * /usr/local/bin/sync.sh

# Multiple specific hours (9am, 1pm, 5pm)
0 9,13,17 * * * /usr/local/bin/reminder.sh
```

!!! tip
    Use [crontab.guru](https://crontab.guru) to test and validate cron expressions.

---

## crontab — User Cron Jobs

Each user has their own crontab. User crontabs run as that user.

```bash
# Edit your crontab
crontab -e

# View your crontab
crontab -l

# Remove your crontab entirely
crontab -r

# Edit another user's crontab (root only)
sudo crontab -u username -e
```

### Example User Crontab

```bash
# Set shell and PATH explicitly (cron has a minimal environment)
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Redirect output to log file (suppress email)
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# Discard all output entirely
*/5 * * * * /usr/local/bin/check.sh > /dev/null 2>&1

# Send output to syslog via logger
0 3 * * * /usr/local/bin/cleanup.sh 2>&1 | logger -t cleanup
```

!!! warning "Cron environment is minimal"
    Cron doesn't load your `.bashrc` or `.profile`. Always use full paths for commands and scripts, and set `PATH` at the top of your crontab if needed.

---

## System Cron (`/etc/cron*`)

System-level cron jobs that run as root live in several directories:

```bash
/etc/cron.d/          # Individual crontab-style files (specify user)
/etc/cron.hourly/     # Scripts dropped here run hourly
/etc/cron.daily/      # Scripts dropped here run daily
/etc/cron.weekly/     # Scripts dropped here run weekly
/etc/cron.monthly/    # Scripts dropped here run monthly
/etc/crontab          # System crontab (includes username field)
```

### `/etc/cron.d/` Format

Files in `cron.d` require a username field (unlike user crontabs):

```bash
# /etc/cron.d/myapp
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# minute hour dom month dow user command
0 2 * * * root /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

### Drop-in Scripts (`cron.daily`, `cron.weekly`, etc.)

```bash
# Copy script to the appropriate directory
sudo cp myscript.sh /etc/cron.daily/myscript

# Must be executable and owned by root
sudo chmod 755 /etc/cron.daily/myscript
sudo chown root:root /etc/cron.daily/myscript

# Scripts must NOT have a file extension (cron ignores files with dots)
# Good: /etc/cron.daily/backup
# Bad:  /etc/cron.daily/backup.sh   ← will NOT run
```

---

## anacron

`anacron` runs jobs that were missed because the machine was off — unlike cron, it doesn't require the system to be running at the exact scheduled time.

```bash
# Config file
cat /etc/anacrontab

# Example entry
# period  delay  job-id    command
1         5      daily.job /etc/cron.daily
7         10     weekly.job /etc/cron.weekly
```

`/etc/cron.daily`, `weekly`, and `monthly` are typically run through anacron rather than cron directly.

!!! note "Debian vs Ubuntu"
    Both include anacron by default on desktop installs. On minimal server installs, `anacron` may not be present — verify with `which anacron`. The daily/weekly/monthly directories still work but through the system crontab instead.

---

## Viewing Cron Logs

```bash
# Debian — cron logs to syslog
grep CRON /var/log/syslog
grep CRON /var/log/syslog | tail -20

# Ubuntu (systemd journal)
journalctl -u cron
journalctl -u cron -f         # follow live
journalctl -u cron --since "1 hour ago"

# Check if cron is running
systemctl status cron
```

---

## Systemd Timers as Cron Alternative

For new services, systemd timers are more robust than cron — they log to the journal, handle missed runs, and integrate with service dependencies. See the [Systemd page](systemd.md#timer-units-systemd-cron-alternative) for full details.

Quick comparison:

| Feature | cron | systemd timer |
|---------|------|---------------|
| Easy syntax | ✅ | ❌ (two files) |
| Logs to journal | ❌ | ✅ |
| Run if missed | Only with anacron | ✅ (`Persistent=true`) |
| Dependency handling | ❌ | ✅ |
| Per-user | ✅ | ✅ |
| No extra install | ✅ | ✅ |