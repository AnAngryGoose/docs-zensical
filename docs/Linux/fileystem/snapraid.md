# SnapRAID Configuration Guide

[SnapRAID Official Docs](https://www.snapraid.it/)

---

## Overview

**SnapRAID** is a backup program for disk arrays. It stores parity information of the data to recover from up to six disk failures.

Unlike traditional RAID 5 or 6, SnapRAID is not real-time; it is a snapshot-based system.


* Provide fault tolerance to protect against (inevitable) hard drive failure
* Checksum files to guard against bitrot
* Support hard drives of differing / mismatched sizes
* Enable incremental upgrading of hard drives in batches as small as one
* Each drive should have a separately readable filesystem with no striping of data


### The Parity Concept

SnapRAID requires dedicating specific disks to "Parity."

* **Parity Level:** One parity disk protects against one drive failure. Two parity disks protect against two failures, and so on.
* **Sizing:** The parity disk **must** be equal to or larger than the largest single data disk in the array.
* **Storage:** Parity disks contain only the large `.parity` file; they cannot be used for standard data storage.

---

## Installation

To ensure the latest features and compatibility, SnapRAID is often compiled from source.

### Compile from Source

Run the following commands to download, compile, and install version 13.0.

```bash
# Download source
wget https://github.com/amadvance/snapraid/releases/download/v13.0/snapraid-13.0.tar.gz

# Extract archive
tar xzvf snapraid-13.0.tar.gz

# Enter directory
cd snapraid

# Compile and Install
./autogen.sh
./configure
make
make check
sudo make install

# Verify installation
snapraid -V

```

---

## Configuration

SnapRAID is configured via a single text file located at `/etc/snapraid.conf`. This file defines the parity volumes, content files, and data disks.

### Key Components

1. **Parity Files:** The destination for redundancy data.
2. **Content Files:** These act as the "index" or "database" of the array. They contain checksums and file maps.
* *Critical:* Multiple copies should be stored on separate physical disks (e.g., one on the boot drive, others on data drives) to ensure the index survives a disk failure.


3. **Data Disks:** The actual storage drives containing the media/files to protect.

### Configuration Example (`/etc/snapraid.conf`)

Create or edit the file using a text editor:

```bash
# --- Parity Location --- 
# The parity file must be on the dedicated parity drive
parity /mnt/parity1/snapraid.parity 

# --- Content files ---
# Store copies on multiple physical drives for safety
content /var/snapraid/snapraid.content
content /mnt/disk1/snapraid.content
content /mnt/disk2/snapraid.content

# --- Data Disks ---
# Assign a name (d1, d2) and a mount point for each disk
data d1 /mnt/disk1
data d2 /mnt/disk2

# --- Excludes ---
# Prevent temporary files from breaking the sync
exclude *.unrecoverable
exclude /tmp/
exclude /lost+found/

```

---

## Usage & Maintenance

Because SnapRAID is snapshot-based, the array is not protected until a "sync" is performed.

### Syncing

Run the `sync` command to update the parity information. This reads data from the data disks and computes the parity.

```bash
snapraid sync

```

* **Note:** The first sync may take several hours. Subsequent syncs only process changed data.
* **Interruptible:** The process can be stopped (`Ctrl+C`) and resumed later.

### Scrubbing

Scrubbing checks the data and parity for silent corruption (bitrot).

```bash
snapraid scrub

```

* **Default Behavior:** Checks approximately 8% of the array that hasn't been scrubbed in the last 10 days.
* **Custom Plan:** To check a specific percentage (e.g., 5%) of blocks older than 20 days:
```bash
snapraid -p 5 -o 20 scrub

```



### Automation

Since SnapRAID requires manual commands, it is highly recommended to automate these tasks using scripts. The [SnapRAID AIO Script](https://github.com/auanasgheps/snapraid-aio-script?tab=readme-ov-file) is a popular community solution for automating syncs, scrubs, and monitoring.

---

## Recovery

### Undeleting Files

If a file is accidentally deleted, SnapRAID can restore it to its previous state (like a backup).

**Restore specific file:**

```bash
snapraid fix -f /path/to/file

```

**Restore specific directory:**

```bash
snapraid fix -f /path/to/directory/

```

**Restore only missing files:**
Use the `-m` flag to recover deleted files without overwriting existing changed files.

```bash
snapraid fix -m -f /path/to/directory/

```

### Recovering a Failed Drive

In the event of a total disk failure, follow this procedure strictly.

**1. Halt Operations**
Disable any automated tasks (cron jobs, AIO scripts) and stop writing new data to the array.

**2. Reconfigure**
Install the replacement disk and mount it. Edit `/etc/snapraid.conf` to point the failed disk identifier to the new mount point.

* *Example:* Change `data d1 /mnt/failed_disk` to `data d1 /mnt/new_disk`.

**3. Fix (Restore Data)**
Run the fix command targeting the specific disk identifier (`-d`). Log the output to a file on a different drive for review.

```bash
snapraid -d d1 -l fix.log fix

```

* *Note:* This process will read from all other disks to reconstruct the missing data. It runs at the speed of the slowest drive.

**4. Review**
Check `fix.log` for "unrecoverable" errors. If files were modified since the last sync, they may not be fully recoverable.

**5. Verification (Optional)**
Run a check on the restored disk to verify integrity.

```bash
snapraid -d d1 -a check

```

**6. Resync**
Once verified, run a standard sync to update the array status.

```bash
snapraid sync

```

---
