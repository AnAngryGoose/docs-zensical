# Filesystem Management Guide

---

## Overview

General idea on managing filesystems. This shows specifically `ext4` and `xfs` as that's what I used, but there are plently of other options. 

---

## 1. Identification

Before modifying any disks, identifying the correct device identifiers is critical to prevent data loss.

```bash
lsblk -o NAME,MODEL,SIZE,TYPE,FSTYPE

```

**Reference Output:**

```text
NAME                      MODEL                            SIZE TYPE FSTYPE
sda                       SanDisk SD8TB8U2               238.5G disk 
└─sda1                                                   238.5G part ext4
sdb                       WDC WD120EFGX-68                10.9T disk 
└─sdb1                                                    10.9T part xfs
sdc                       WDC WD120EFGX-68                10.9T disk 
└─sdc1                                                    10.9T part xfs
sdd                       WDC WD120EFGX-68                10.9T disk 
└─sdd1                                                    10.9T part xfs
nvme0n1                   WDC PC SN730 SDBQNTY-256G-1001 238.5G disk 
├─nvme0n1p1                                                  1M part 
├─nvme0n1p2                                                  2G part ext4
└─nvme0n1p3                                              236.5G part LVM2_member
  └─ubuntu--vg-ubuntu--lv                                  100G lvm  ext4

```

---

## 2. Partitioning & Formatting

You can choose between a scriptable command-line approach or an interactive visual tool.

### Option A: Command Line (Scriptable)

#### SSD Setup (ext4)

Target: `/dev/sda` (Appdata/Cache)

```bash
# 1. Wipe old filesystem signatures
sudo wipefs -a /dev/sda

# Use parted for GPT specifically (Recommended for drives >2TB)
sudo parted -s /dev/sda mklabel gpt 

# 3. Format to EXT4
sudo mkfs.ext4 -L "appdata" /dev/sda1

```

#### HDD Setup (XFS)

Target: `/dev/sdb`, `/dev/sdc`, `/dev/sdd` (Mass Storage)

```bash
# Drive 1 (sdb)
sudo wipefs -a /dev/sdb
sudo parted -s /dev/sdb mkpart primary xfs 0% 100%
sudo mkfs.xfs -f -L "disk1" /dev/sdb1

# Drive 2 (sdc)
sudo wipefs -a /dev/sdc
sudo parted -s /dev/sdc mkpart primary xfs 0% 100%
sudo mkfs.xfs -f -L "disk2" /dev/sdc1

# Drive 3 (sdd)
sudo wipefs -a /dev/sdd
sudo parted -s /dev/sdd mkpart primary xfs 0% 100%
sudo mkfs.xfs -f -L "parity1" /dev/sdd1

```

### Option B: Interactive (Visual)

For a visual interface, use `cfdisk`.

!!! warning "Formatting Required"
`cfdisk` only creates the *partition*. You must still run the `mkfs` commands listed in Option A to format the filesystem after creating the partitions.

1. **Launch Tool:** `sudo cfdisk /dev/sda` (Repeat for `sdb`, `sdc`, `sdd`).
2. **Label Type:** Select **gpt**.
3. **Delete:** Remove existing partitions if necessary.
4. **New:** Select **[ New ]** -> **[ Enter ]** (Use max size).
5. **Write:** Select **[ Write ]** -> Type `yes`.
6. **Quit:** Select **[ Quit ]**.

---

## 3. Mounting & Persistence

### Get UUIDs

Use UUIDs for mounting. Unlike `/dev/sdX` names, UUIDs do not change if drive order changes. You'll need to map them using these or they may not show on reboot

```bash
ls -l /dev/disk/by-uuid/

```

**Output:**

```bash
lrwxrwxrwx 1 root root 10 Dec  7 05:34 02de315f-8ced-4745-bb4f-2d24c46efa48 -> ../../sdb1
lrwxrwxrwx 1 root root 15 Dec  7 04:33 3c26dce0-7f10-426c-bd67-e9f2d77889ee -> ../../nvme0n1p2
lrwxrwxrwx 1 root root 10 Dec  7 05:34 603999da-c2dd-484d-80bf-ab4977680e90 -> ../../sdc1
lrwxrwxrwx 1 root root 10 Dec  7 05:34 844cae61-9599-4a57-9431-18c0a33904e4 -> ../../sda1
lrwxrwxrwx 1 root root 10 Dec  7 05:34 9860056e-b314-4cb9-acc5-30e76281795b -> ../../sdd1
lrwxrwxrwx 1 root root 10 Dec  7 04:33 d5ac7faa-876c-43ff-b27a-94c53bf79bb2 -> ../../dm-0

```

### Create Mount Points

Create the directories where the drives will be accessed.

```bash
sudo mkdir -p /mnt/disk1
sudo mkdir -p /mnt/disk2
sudo mkdir -p /mnt/parity1
sudo mkdir -p /mnt/appdata

```

### Edit fstab

Add the following to the bottom of `/etc/fstab` to ensure drives mount automatically at boot.

```bash
# ---------------------------------------------------------
# DATA DRIVES (XFS) - 12TB WD Reds
# ---------------------------------------------------------
# Disk 1 (sdb1)
UUID=02de315f-8ced-4745-bb4f-2d24c46efa48 /mnt/disk1 xfs defaults,noatime 0 0

# Disk 2 (sdc1)
UUID=603999da-c2dd-484d-80bf-ab4977680e90 /mnt/disk2 xfs defaults,noatime 0 0

# Parity Drive (sdd1)
UUID=9860056e-b314-4cb9-acc5-30e76281795b /mnt/parity1 xfs defaults,noatime 0 0

# ---------------------------------------------------------
# FAST STORAGE (EXT4) - 240GB SSD
# ---------------------------------------------------------
# Appdata/Cache (sda1)
UUID=844cae61-9599-4a57-9431-18c0a33904e4 /mnt/appdata ext4 defaults,noatime 0 0

```

### Reload Configuration

After editing `fstab`, verify the syntax and reload system daemons.

```bash
sudo findmnt --verify
sudo systemctl daemon-reload

```

---

## 4. Verification

Perform these checks **before** rebooting to ensure the configuration is valid.

1. **Test Mount:**
```bash
# No output is good news
sudo mount -a

```


2. **Check Sizing:**
```bash
df -h

```


* `disk1`, `disk2`, `parity1`: Should be ~11T
* `appdata`: Should be ~220G


3. **Check Permissions:**
```bash
ls -ld /mnt/disk1 /mnt/disk2 /mnt/parity1 /mnt/appdata

```


If listed as `root`, change ownership to your specific user (e.g., `goose`):
```bash
sudo chown -R goose:goose /mnt/disk1 /mnt/disk2 /mnt/parity1 /mnt/appdata

```



---

## 5. Docker Integration Reference

Once mounted, pass these paths to containers via `compose.yaml`.

### FileBrowser (File Management)

```yaml
services:
  filebrowser:
    volumes:
      # Map Host Path : Container Path
      - /mnt/nvme_storage:/mnt/nvme_storage

```

### Beszel (System Monitoring)

```yaml
services:
  beszel-agent:
    volumes:
      - /mnt/nvme_storage:/mnt/nvme_storage:ro
    environment:
      # Comma-separated list of mount points to monitor
      - EXTRA_FILESYSTEMS=/mnt/nvme_storage

```

---

## 6. General CLI Cheat Sheet

### Identification & Hardware Info

| Command | Description |
| --- | --- |
| `lsblk` | **List Block Devices.** The best "at a glance" view of disks, partitions, and mount points. |
| `lsblk -f` | Lists devices plus **filesystem types** (ext4, ntfs) and **UUIDs**. |
| `sudo fdisk -l` | Detailed low-level partition table dump. Good for verifying sector sizes. |
| `sudo lshw -class disk` | Hardware details (model numbers, serial numbers, firmware). |
| `sudo blkid` | Prints the UUIDs and labels for all block devices. |

### Disk Usage & Space

| Command | Description |
| --- | --- |
| `df -h` | **Disk Free.** Shows total/used/avail space in "Human Readable" format (GB/TB). |
| `du -sh /path/to/dir` | **Disk Usage.** Calculates the size of a specific directory. |
| `ncdu` | **Interactive Usage.** A visual, navigable tool to find large files. (Requires install). |

### Health & Maintenance

| Command | Description |
| --- | --- |
| `sudo smartctl -a /dev/sda` | **SMART Data.** Checks physical health, temp, and error logs (Requires `smartmontools`). |
| `sudo fsck /dev/sda1` | **File System Check.** Repairs corruption. **Only run on unmounted drives.** |

### Mounting Operations

| Command | Description |
| --- | --- |
| `sudo mount /dev/sdb1 /mnt/usb` | Manually mounts a partition to a folder. |
| `sudo umount /mnt/usb` | Unmounts the drive safely. |
| `sudo mount -a` | Mounts everything listed in `/etc/fstab` that isn't already mounted. |