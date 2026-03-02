---
title: Disk Management
description: Partitioning, mounting, fstab, and disk health on Linux.
---

# Disk Management

> **Reference:** [Debian Wiki — DiskPartitioningGuide](https://wiki.debian.org/DiskPartitioningGuide) · [Ubuntu Docs — Partitioning](https://help.ubuntu.com/community/DiskSpace) · [Linux man-pages: mount, fstab, fdisk](https://www.kernel.org/doc/man-pages/)

---

## Viewing Disks & Partitions

```bash
# List all block devices (disks, partitions, loop devices)
lsblk

# Detailed view with filesystem type and mount points
lsblk -f

# List disks with partition table info
sudo fdisk -l

# Show disk UUIDs (use these in fstab, not /dev/sdX)
blkid

# Specific device
blkid /dev/sda1

# Disk usage — how full are your filesystems?
df -h               # human-readable sizes
df -hT              # includes filesystem type

# Directory disk usage
du -sh /path/       # total size of directory
du -sh /*           # size of each top-level directory
du -h --max-depth=1 /var    # one level deep
```

---

## Partition Management

### fdisk (MBR / GPT)

```bash
# Open a disk for editing (destructive — be careful)
sudo fdisk /dev/sda

# fdisk interactive commands:
# p — print partition table
# n — new partition
# d — delete partition
# t — change partition type
# w — write changes and exit
# q — quit without saving
```

### parted (preferred for large disks / GPT)

```bash
# Open disk
sudo parted /dev/sda

# parted commands:
# print           — show partition table
# mklabel gpt     — create GPT partition table
# mkpart primary ext4 0% 100%   — create partition using full disk
# quit
```

!!! warning
    Partition changes are destructive. Always verify the correct device (`lsblk`) before running `fdisk` or `parted`. `/dev/sda` vs `/dev/sdb` matters.

---

## Filesystems

```bash
# Create ext4 filesystem on a partition
sudo mkfs.ext4 /dev/sdb1

# Create with a label
sudo mkfs.ext4 -L mydrive /dev/sdb1

# XFS (common on RHEL/CentOS, used by some NAS setups)
sudo mkfs.xfs /dev/sdb1

# Check and repair a filesystem (must be unmounted)
sudo fsck /dev/sdb1
sudo fsck -y /dev/sdb1    # auto-fix errors

# Resize ext4 filesystem (after expanding partition)
sudo resize2fs /dev/sdb1
```

---

## Mounting

```bash
# Mount a partition
sudo mount /dev/sdb1 /mnt/data

# Mount with specific filesystem type
sudo mount -t ext4 /dev/sdb1 /mnt/data

# Mount an ISO
sudo mount -o loop image.iso /mnt/iso

# Unmount
sudo umount /mnt/data
sudo umount /dev/sdb1    # either works

# View currently mounted filesystems
mount | column -t
findmnt                  # tree view
```

---

## `/etc/fstab` — Persistent Mounts

`fstab` defines filesystems that mount automatically at boot.

```bash
# Format:
# <device>   <mountpoint>   <fstype>   <options>   <dump>  <pass>

# View current fstab
cat /etc/fstab
```

### Example fstab Entries

```bash
# Mount by UUID (always prefer UUID over /dev/sdX — device names can change)
UUID=abc123-def456  /mnt/data  ext4  defaults  0  2

# NFS share
192.168.1.20:/volume1/media  /mnt/media  nfs  defaults,_netdev  0  0

# tmpfs (RAM-based temp filesystem)
tmpfs  /tmp  tmpfs  defaults,size=1G  0  0

# Bind mount (make a directory appear at another path)
/source/path  /target/path  none  bind  0  0
```

### fstab Options

| Option | Meaning |
|--------|---------|
| `defaults` | rw, suid, dev, exec, auto, nouser, async |
| `noatime` | Don't update access time (performance improvement) |
| `ro` | Read-only |
| `_netdev` | Wait for network before mounting (NFS, SMB) |
| `nofail` | Don't fail boot if device missing |
| `user` | Allow non-root users to mount |

```bash
# Test fstab without rebooting
sudo mount -a      # mounts everything in fstab not yet mounted

# Verify fstab syntax
sudo findmnt --verify
```

---

## Disk Health

```bash
# Install smartmontools
sudo apt install smartmontools

# Quick health check
sudo smartctl -H /dev/sda

# Full SMART info
sudo smartctl -a /dev/sda

# Run short self-test (~2 min)
sudo smartctl -t short /dev/sda

# Run long self-test (~hours, do overnight)
sudo smartctl -t long /dev/sda

# Check test results
sudo smartctl -l selftest /dev/sda
```

### SMART Attributes to Watch

| Attribute | Warning Sign |
|-----------|-------------|
| `Reallocated_Sector_Ct` | Any value > 0 |
| `Current_Pending_Sector` | Any value > 0 |
| `Offline_Uncorrectable` | Any value > 0 |
| `Reallocated_Event_Count` | Increasing count |

---

## Swap

```bash
# View swap usage
swapon --show
free -h

# Create a swapfile
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent — add to fstab
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Adjust swappiness (how aggressively kernel uses swap)
# 0 = avoid swap, 100 = use swap aggressively, 10-20 typical for servers
cat /proc/sys/vm/swappiness
sudo sysctl vm.swappiness=10

# Persist swappiness across reboots
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
```

---

## LVM Basics (Logical Volume Manager)

LVM adds a flexible abstraction over physical disks — resize volumes without partitioning.

```bash
# Show physical volumes
sudo pvs
sudo pvdisplay

# Show volume groups
sudo vgs
sudo vgdisplay

# Show logical volumes
sudo lvs
sudo lvdisplay

# Extend a logical volume by 10GB
sudo lvextend -L +10G /dev/vgname/lvname

# Resize the filesystem after extending LV
sudo resize2fs /dev/vgname/lvname    # ext4
sudo xfs_growfs /mountpoint          # xfs
```