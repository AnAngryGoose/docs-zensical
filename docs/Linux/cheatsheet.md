---
icon: lucide/terminal
title: Cheatsheet
description: A one-page quick reference for the most-used Linux commands.
tags:
  - linux
  - cheat-sheet
  - reference
---

# Linux Cheat Sheet

A fast lookup for the commands used most often on these servers. Each section has
a fuller page linked for depth — this page is just the "I need the command *now*"
reference.

## Files & Navigation

| Command | Description |
|---|---|
| `ls -lah` | List all files, long format, human-readable sizes |
| `cd -` | Jump back to the previous directory |
| `pwd` | Print current directory |
| `cp -r src dst` | Copy recursively |
| `mv src dst` | Move / rename |
| `rm -rf dir` | Delete recursively (**careful**) |
| `mkdir -p a/b/c` | Create nested directories |
| `ln -s target link` | Create a symlink |
| `tree -L 2` | Directory tree, 2 levels deep |
| `stat file` | Size, permissions, timestamps |
| `readlink -f file` | Resolve to absolute/real path |

## Viewing & Searching Text

| Command | Description |
|---|---|
| `cat` / `bat` | Print a file (`bat` = syntax-highlighted) |
| `less file` | Page through a file (`q` to quit) |
| `head -n 20` / `tail -n 20` | First / last 20 lines |
| `tail -f file` | Follow a file live |
| `grep -rin "text" .` | Recursive, case-insensitive search with line numbers |
| `grep -v "text"` | Invert — lines *without* the match |
| `find . -name "*.conf"` | Find files by name |
| `find . -mtime -1` | Files modified in the last day |
| `sed -n '10,20p' file` | Print lines 10–20 |
| `awk '{print $1}'` | Print the first column |
| `wc -l file` | Count lines |
| `diff a b` | Compare two files |

## Permissions & Ownership

See **[Users & Permissions](users-permissions.md)**.

| Command | Description |
|---|---|
| `chmod 644 file` | `rw-r--r--` (files) |
| `chmod 755 dir` | `rwxr-xr-x` (dirs / scripts) |
| `chmod 600 .env` | Owner-only (secrets) |
| `chmod -R 755 dir/` | Recursive |
| `chown user:group file` | Change owner and group |
| `chown -R u:g /opt/app` | Recursive ownership |
| `umask 027` | Default new-file mask |

## Users & Groups

| Command | Description |
|---|---|
| `sudo useradd -m -s /bin/bash user` | Create user with home + shell |
| `sudo passwd user` | Set a password |
| `sudo usermod -aG group user` | Add user to a group (**always `-aG`**) |
| `id user` | UID, GID, and groups |
| `groups user` | List a user's groups |
| `sudo userdel -r user` | Delete user and home |

## Processes & Resources

| Command | Description |
|---|---|
| `htop` / `btop` | Interactive process/resource viewer |
| `ps aux \| grep name` | Find a process |
| `kill -9 PID` | Force-kill a process |
| `pkill -f pattern` | Kill by command pattern |
| `free -h` | Memory usage |
| `uptime` | Load averages |
| `nproc` | CPU core count |
| `df -h` | Disk free per filesystem |
| `du -sh *` | Size of each item in the current dir |
| `lsof -i :8080` | What's using port 8080 |
| `watch -n1 cmd` | Re-run `cmd` every second |

## Services (systemd)

See **[Systemd](systemd.md)**.

| Command | Description |
|---|---|
| `systemctl status svc` | Service status |
| `sudo systemctl restart svc` | Restart |
| `sudo systemctl enable --now svc` | Enable at boot **and** start now |
| `systemctl is-enabled svc` | Is it set to start at boot? |
| `systemctl --failed` | List failed units |
| `systemctl list-timers` | Scheduled timers + next run |

## Logs (journalctl)

| Command | Description |
|---|---|
| `journalctl -u svc -n 50` | Last 50 lines for a service |
| `journalctl -fu svc` | Follow a service live |
| `journalctl -b` | Logs since last boot |
| `journalctl -p err` | Errors only |
| `journalctl --since "1 hour ago"` | Time-filtered |
| `journalctl --vacuum-time=7d` | Trim journal to 7 days |

## Disk & Storage

See **[Disk Management](disk-management.md)**.

| Command | Description |
|---|---|
| `lsblk -f` | Block devices with FS + mount points |
| `blkid` | Show UUIDs (use these in `fstab`) |
| `sudo mount -a` | Mount everything in `fstab` (test after editing) |
| `findmnt --verify` | Validate `fstab` syntax |
| `sudo smartctl -H /dev/sda` | Quick disk health check |
| `duf` / `df -hT` | Disk usage with FS type |

## Networking

See **[Networking](networking.md)**.

| Command | Description |
|---|---|
| `ip a` | Interfaces and addresses |
| `ip r` | Routing table |
| `ss -tlnp` | Listening TCP ports + process |
| `dig name +short` | Quick DNS lookup |
| `ping -c4 host` | 4 pings and stop |
| `nc -zv host 22` | Is a port open? |
| `curl -I url` | HTTP headers only |
| `mtr host` | Live traceroute + ping |

## Packages (apt)

See **[Package Management](package-management.md)**.

| Command | Description |
|---|---|
| `sudo apt update && sudo apt upgrade` | Refresh index + upgrade |
| `sudo apt install pkg` | Install |
| `sudo apt remove --purge pkg` | Remove + configs |
| `sudo apt autoremove` | Drop unused dependencies |
| `apt search term` / `apt show pkg` | Find / inspect a package |
| `sudo apt-mark hold pkg` | Pin a package's version |
| `dpkg -S /path/to/file` | Which package owns a file |

## Archives & Transfer

| Command | Description |
|---|---|
| `tar czf out.tar.gz dir/` | Create a gzip tarball |
| `tar xzf in.tar.gz` | Extract a gzip tarball |
| `tar tzf in.tar.gz` | List contents without extracting |
| `scp file user@host:/path` | Copy a file over SSH |
| `rsync -avh --progress src/ dst/` | Sync directories (resumable, fast) |
| `rsync -avh src/ user@host:/dst/` | Sync to a remote host |

## Handy One-Liners

```bash
# Biggest 10 items in the current directory
du -sh * | sort -rh | head -n 10

# Free up disk: what's using space under /var
sudo du -xh /var | sort -rh | head -n 20

# Follow a service and grep at the same time
journalctl -fu docker | grep -i error

# Find and delete files older than 30 days
find /path -type f -mtime +30 -delete

# Replace text across files (in place)
grep -rl "old" . | xargs sed -i 's/old/new/g'

# Quick HTTP server for the current directory
python3 -m http.server 8000

# Show the 10 largest files under a path
find /path -type f -printf '%s %p\n' | sort -rn | head | numfmt --field=1 --to=iec
```
