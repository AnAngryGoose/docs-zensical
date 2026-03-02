---
title: Performance & Debugging
description: Tools for monitoring system resources and diagnosing problems on Linux.
---

# Performance & Debugging

> **Reference:** [Linux man-pages](https://www.kernel.org/doc/man-pages/) · [procps-ng Docs](https://gitlab.com/procps-ng/procps) · [sysstat Docs](https://github.com/sysstat/sysstat)

---

## CPU & Memory

### top / htop

```bash
# Built-in process monitor
top

# htop — improved interactive version (install if needed)
sudo apt install htop
htop
```

**htop key bindings:**

| Key | Action |
|-----|--------|
| `F2` | Setup/config |
| `F3` | Search processes |
| `F5` | Tree view |
| `F6` | Sort by column |
| `F9` | Kill process |
| `F10` | Quit |
| `u` | Filter by user |
| `Space` | Tag process |

```bash
# One-liner CPU/memory snapshot
vmstat 1 5      # 5 readings, 1 second apart
# procs  memory       swap   io    system    cpu
# r  b   swpd  free  buff  cache  si so  bi bo  in cs us sy id wa

# Memory usage summary
free -h

# Detailed memory breakdown
cat /proc/meminfo

# Per-process memory
ps aux --sort=-%mem | head -20    # top 20 by memory
ps aux --sort=-%cpu | head -20    # top 20 by CPU
```

---

## Disk I/O

```bash
# Install iotop
sudo apt install iotop

# Show processes doing disk I/O
sudo iotop

# Non-interactive snapshot
sudo iotop -b -n 3     # 3 iterations, batch mode

# iostat — disk I/O statistics (install sysstat)
sudo apt install sysstat
iostat -x 1 5          # extended stats, 5 readings every 1 second

# Check disk read/write speed (simple)
# Write test
dd if=/dev/zero of=/tmp/testfile bs=1G count=1 oflag=direct
# Read test
dd if=/tmp/testfile of=/dev/null bs=1G count=1 iflag=direct
rm /tmp/testfile

# Check what's using disk
lsof +D /path/         # open files in a directory
fuser /path/to/file    # what processes have this file open
```

---

## Network Performance

```bash
# Current network connections and their state
ss -tnp

# Bandwidth usage per interface (install nload or iftop)
sudo apt install nload
nload eth0

sudo apt install iftop
sudo iftop -i eth0      # per-connection bandwidth

# Network statistics
netstat -s             # protocol statistics (if netstat installed)
cat /proc/net/dev      # raw interface counters

# Test bandwidth between hosts (install iperf3)
sudo apt install iperf3
# On server:
iperf3 -s
# On client:
iperf3 -c server-ip
iperf3 -c server-ip -t 30    # 30 second test
```

---

## Process Management

```bash
# List all processes
ps aux
ps aux | grep processname

# Process tree
pstree
pstree -p              # with PIDs

# Find a process by name
pgrep nginx
pgrep -la nginx        # with full command line

# Find which process is on a port
sudo ss -tlnp | grep :8080
sudo lsof -i :8080

# Kill a process
kill PID               # graceful (SIGTERM)
kill -9 PID            # force kill (SIGKILL)
killall nginx          # kill all processes named nginx
pkill -f "python script.py"   # kill by full command match

# Change process priority
nice -n 10 command              # start with lower priority (19 = lowest)
sudo renice -n 10 -p PID        # adjust running process priority
```

---

## System Info

```bash
# CPU info
lscpu
cat /proc/cpuinfo | grep "model name" | head -1
nproc                  # number of CPU cores

# Memory
free -h
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable"

# Uptime and load average
uptime
# load average: 0.5, 0.3, 0.2
# These are 1min, 5min, 15min averages
# Rule of thumb: load > number of CPU cores = system under pressure

# OS and kernel info
uname -a               # kernel version and architecture
cat /etc/os-release    # distro and version
hostnamectl            # hostname, OS, kernel

# Hardware info
lshw -short            # hardware summary (install lshw)
sudo lshw -class network   # network hardware only
lspci                  # PCI devices
lsusb                  # USB devices
```

---

## Debugging Tools

### strace — System Call Tracing

```bash
# Trace system calls of a running process
sudo strace -p PID

# Trace a command from start
strace command

# Show only file-related calls
strace -e trace=file command

# Count system calls
strace -c command
```

### lsof — Open Files

```bash
# List all open files
sudo lsof

# Files opened by a process
sudo lsof -p PID

# What process has a file open
sudo lsof /path/to/file

# What process is listening on a port
sudo lsof -i :8080
sudo lsof -i TCP:80

# Files opened by a user
sudo lsof -u username
```

### dmesg — Kernel Messages

```bash
# View kernel ring buffer
dmesg
dmesg -T                      # with human-readable timestamps
dmesg | tail -20
dmesg | grep -i "error"
dmesg | grep -i "oom"         # Out of Memory killer events
dmesg | grep -i "usb"         # USB events (useful for zigbee dongle issues)

# Follow live kernel messages
sudo dmesg -w
```

---

## OOM Killer

When the system runs out of memory, the kernel OOM (Out of Memory) killer terminates processes. Check if it's happened:

```bash
# Check for OOM kills
dmesg | grep -i "oom"
journalctl -k | grep -i "oom"
grep -i "out of memory" /var/log/syslog

# What got killed
dmesg | grep -i "killed process"
```

---

## Quick Diagnostic Workflow

When something is wrong and you don't know where to start:

```bash
# 1. Check system load
uptime

# 2. Check memory
free -h

# 3. Check disk space (full disk is a common silent failure cause)
df -h

# 4. Check for failed services
systemctl --failed

# 5. Check recent logs
journalctl -p err -n 50 --no-pager

# 6. Check what's using resources
ps aux --sort=-%cpu | head -10
ps aux --sort=-%mem | head -10

# 7. Check disk I/O
sudo iotop -b -n 1

# 8. Check open ports (something unexpected listening?)
sudo ss -tlnp
```

---

## Benchmarking

```bash
# CPU benchmark (simple — time to calculate pi)
time echo "scale=5000; 4*a(1)" | bc -l

# Disk write speed
dd if=/dev/zero of=/tmp/bench bs=1G count=1 oflag=direct && rm /tmp/bench

# sysbench (comprehensive — install first)
sudo apt install sysbench
sysbench cpu run
sysbench memory run
sysbench fileio --file-test-mode=rndrw prepare
sysbench fileio --file-test-mode=rndrw run
sysbench fileio --file-test-mode=rndrw cleanup
```