# MergerFS Configuration Guide

---

## Overview

**MergerFS** is a [union filesystem](https://en.wikipedia.org/wiki/Union_mount) designed to simplify storage management. It pools multiple storage devices (branches) into a single unified directory (mount point).

Unlike RAID, MergerFS does not strip data across drives. Files exist intact on the individual backing drives.

* **Flexibility:** Drives of different sizes, filesystems, and speeds can be mixed.
* **Resilience:** If one drive fails, only the data on that specific drive is lost; the rest of the pool remains accessible.
* **Dynamic:** Drives can be added or removed from the pool without rebuilding the array.

---

## Installation

MergerFS is available in most standard Linux repositories, though building from source ensures the latest feature set.

```bash
# Standard installation
sudo apt install mergerfs fuse

```

*For latest versions/source:* [GitHub Repository](https://github.com/trapexit/mergerfs)

---

## Branch Setup & Preparation

!!! WARNING Important
    
    Never write data directly to the `/mnt/disk1` folders if you can help it. Always write to `/mnt/storage`. If you modify files on the backing disks directly while MergerFS is running, you might see cache inconsistencies until you remount.

Before creating the pool, the underlying mount points (branches) should be organized and secured.

### 1. Naming Convention

Using a consistent naming structure helps identify drive types within the pool.

* **HDDs:** `/mnt/hdd/Serial-Number`
* **SATA SSDs:** `/mnt/ssd/Serial-Number`
* **NVMe:** `/mnt/nvme/Serial-Number`
* **Remote Shares:** `/mnt/remote/Share-Name`

**Example Layout:**

```text
/mnt/
├── hdd/
│   ├── 10T-01234567
│   └── 20T-12345678
└── nvme/
    └── 1T-ABCDEFGH

```

### 2. Permissions & Locking (Critical)

#### **mount points**

To ensure the directory is only used as a point to mount another filesystem it is good to lock it down as much as possible. Be sure to do this before mounting a filesystem to it.

Run these commands for **each** underlying drive mount point:

```bash
# 1. Set ownership to root
sudo chown root:root /mnt/hdd/10T-XYZ

# 2. Remove read/write/execute permissions (makes it inaccessible directly)
sudo chmod 0000 /mnt/hdd/10T-XYZ

# 3. Mark as a specific branch mount (prevents mounting if drive is missing)
sudo setfattr -n user.mergerfs.branch_mounts_here

```

The extended attribute user.mergerfs.branch_mounts_here is used by the branches-mount-timeout option to recognize whether or not a mergerfs branch path points to the intended filesystem.

The chattr is likely to only work on EXT{2,3,4} filesystems but will restrict even root from modifying the directory or its content but is still able to be a mount point.


#### **mounted filesystems**

For those new to Linux, intending to be the primary individual logged into the system, or simply want to simplify permissions it is recommended to set the root of mounted filesystems like /tmp/ is set to. Owned by root, ugo+rwx and sticky bit set.

This must be done after mounting the filesystem to the target mount point.

```bash
$ sudo chown root:root /mnt/hdd/10T-SERIALNUM
$ sudo chmod 1777 /mnt/hdd/10T-SERIALNUM
$ sudo setfattr -n user.mergerfs.branch /mnt/hdd/10T-SERIALNUM
```

---

## Pool Configuration

### 1. Create Pool Directory

!!! info inline end

    For Linux v6.6 and above (defaults - quick start)
    >-cache.files=off

    >-category.create=pfrd

    >-func.getattr=newest

    >-dropcacheonclose=false

Create the unified mount point where applications will access the data.

```bash
sudo mkdir -p /mnt/storage

```

### 2. Command Line Test

Test the pool creation manually before making it persistent.

!!! note
    These settings are just what I used. Your situation may benefit from different layout. Read the docs to see what would work best. 

```bash
mergerfs -o cache.files=off,category.create=mspmfs,func.getattr=newest,dropcacheonclose=false,minfreespace=200G,moveonenospc=true /mnt/disk*/mnt /mnt/storage

```

### 3. Persistence (`/etc/fstab`)

Add the configuration to `/etc/fstab` to ensure the pool mounts at boot.

**Recommended Options (Linux Kernel v6.6+):**

* `cache.files=off`: Disables page caching (reduces RAM usage/complexity).
* `category.create=mspmfs`: **M**ost **S**pace, **P**ath **M**ost **F**ree **S**pace. Writes new files to the drive with the most free space, unless the path already exists on another drive.
* `func.getattr=newest`: Returns file attributes from the file with the newest `mtime`. Critical for apps like Plex/Kodi to detect changes.
* `minfreespace=200G`: Prevents filling a drive completely; moves to the next drive when 200GB remains.

**Add to `/etc/fstab`:**

```bash
# <branches> <mountpoint> <type> <options> <dump> <pass>
/mnt/hdd/WD-B00WHLZD:/mnt/hdd/WD-B00WUHXD /mnt/storage mergerfs cache.files=off,category.create=mspmfs,func.getattr=newest,dropcacheonclose=false,minfreespace=200G,moveonenospc=true,fsname=mergerfs 0 0

```

### 4. Verification

Reload the system daemon and mount the pool.

```bash
# Verify syntax
sudo findmnt --verify

# Reload fstab
sudo systemctl daemon-reload

# Mount all
sudo mount -a

# Check size (Should equal sum of all drives)
df -h

```

---

## Docker Integration

When configuring containers, **always** use the unified `/mnt/storage` path. Never bind volumes to the individual disks (e.g., `/mnt/disk1`), as this bypasses MergerFS logic.

**Example `compose.yaml`:**

```yaml
services:
  plex:
    image: linuxserver/plex
    volumes:
      # Correct: Uses the pool
      - /mnt/storage/media/movies:/movies
      - /mnt/storage/media/tv:/tv
      
      # INCORRECT: Do not map directly to backing disks
      # - /mnt/disk1/media/movies:/movies 

```

---

## Maintenance & Troubleshooting

### Handling Cached/Deleted Files

If a file is deleted from the pool but appears to persist (ghost file), or if changes aren't reflecting immediately, the filesystem cache may need clearing.

```bash
sync && echo 3 | sudo tee /proc/sys/vm/drop_caches

```

* *Note:* Deleting a file from the pool removes it from the underlying drive.

### Common Issues

* **Permissions:** Run MergerFS as `root`. Running as a standard user can cause unpredictable behavior.
* **Missing Files:** If directories appear missing or permissions seem erratic, ensure the permissions on the underlying physical drives are identical. Use `mergerfs.fsck` to audit synchronization.
* **Scanner Lag (Plex/Kodi):** If media scanners miss new files, ensure `func.getattr=newest` is enabled in the options. This forces the pool to report the most recent modification time found across all branches.

---
