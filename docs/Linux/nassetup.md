# Server Setup: NAS 

* [SnapRAID Manual](https://www.snapraid.it/manual)
* [MergerFS Documentation](https://github.com/trapexit/mergerfs)
---
## You can find more in depth info in the related guides here. 
* This is just how I did it. There are plenty of other ways to do it using different configurations (ZFS, Proxmox, Unraid, TrueNAS, OMV, etc). All options have pros and cons. 

* For my configuration I created an Ubuntu server with MergerFS + SnapRAID. 

    **MergerFS:** Pool drives and read them as one drive. FUSE system. Various configurations, read the official docs. THe docs are **very literal**.

    **SnapRAID:** Create routine snapshots of drives to restore them in case of failure. No striping, easy to expand, straightforward to manage. 

## Mount points
[Branches Docs](https://trapexit.github.io/mergerfs/latest/config/branches/)
### Naming Convention 
**HDDs:** `/mnt/hdd`
**SATA SSDs:** `/mnt/ssd`
**NVME/M2:** `/mnt/nvme`
**REMOTE:** `/mnt/remote`

##### Example
```bash 
$ ls -lh /mnt/
total 16K
drwxr-xr-x 8 root root 4.0K Aug 18  2024 hdd
drwxr-xr-x 6 root root 4.0K Oct  8  2024 nvme
drwxr-xr-x 3 root root 4.0K Aug 24  2024 remote
drwxr-xr-x 3 root root 4.0K Jul 14  2024 ssd

$ ls -lh /mnt/hdd/
total 8K
d--------- 2 root root 4.0K Apr 14 15:58 10T-01234567
d--------- 2 root root 4.0K Apr 12 20:51 20T-12345678

$ ls -lh /mnt/nvme/
total 8K
d--------- 2 root root 4.0K Apr 14 16:00 1T-ABCDEFGH
d--------- 2 root root 4.0K Apr 14 23:24 1T-BCDEFGHI

$ ls -lh /mnt/remote/
total 8K
d--------- 2 root root 4.0K Apr 12 20:23 foo-sshfs
d--------- 2 root root 4.0K Apr 12 20:24 bar-nfs


# You can find the serial number of a drive using lsblk
$ lsblk -d -o NAME,PATH,SIZE,SERIAL
NAME    PATH           SIZE SERIAL
sda     /dev/sda       9.1T 01234567
sdb     /dev/sdb      18.2T 12345678
nvme0n1 /dev/nvme0n1 953.9G ABCDEFGH
nvme1n1 /dev/nvme1n1 953.9G BCDEFGHI
```


## 1. Drive Preparation

Before pooling, drives must be partitioned and formatted.

* **Note:** Identify your disks carefully using `lsblk` or `sudo fdisk -l`.
* **Strategy:** Use **EXT4** for simplicity and reliability on Ubuntu.

### **Formatting SSD as ext4**

```bash
# 1. Create a GPT partition table (Destructive!)
sudo wipefs -a /dev/sda

# 2. Create the partition (100% of disk)
sudo parted -a opt /dev/sdb mkpart primary ext4 0% 100%

# 3. Format to EXT4
# -m 0: Reserve 0% for root (max space for data)
# -L: Label the disk (disk1, disk2, parity1) to make it easy to identify
sudo mkfs.ext4 -m 0 -L disk1 /dev/sdb1

```
### **Formatting 3x HDDs as XFS** 
```bash
# Drive 1 (sdb)
# wipe fs
sudo wipefs -a /dev/sdb
# create partition
sudo parted -s /dev/sdb mkpart primary xfs 0% 100%
# list partition
sudo fdisk -l /dev/sdb
# format partition
sudo mkfs.xfs -f -L "disk1" /dev/sdb1

# Drive 2 (sdc)
sudo wipefs -a /dev/sdc
sudo parted -s /dev/sdc mkpart primary xfs 0% 100%
sudo fdisk -l /dev/sdc
sudo mkfs.xfs -f -L "disk2" /dev/sdc1

# Drive 3 (sdd)
sudo wipefs -a /dev/sdd
sudo parted -s /dev/sdd mkpart primary xfs 0% 100%
sudo fdisk -l /dev/sdd  
sudo mkfs.xfs -f -L "parity1" /dev/sdd1
```

*(Repeat for `disk2`, `parity1`, etc.)*

---

 - **Get UUIDs**
```bash
ls -l /dev/disk/by-uuid/
```

- **Output will be:**
```bash 
lrwxrwxrwx 1 root root 10 Dec  7 05:34 02de315f-8ced-4745-bb4f-2d24c46efa48 -> ../../sdb1
lrwxrwxrwx 1 root root 15 Dec  7 04:33 3c26dce0-7f10-426c-bd67-e9f2d77889ee -> ../../nvme0n1p2
lrwxrwxrwx 1 root root 10 Dec  7 05:34 603999da-c2dd-484d-80bf-ab4977680e90 -> ../../sdc1
lrwxrwxrwx 1 root root 10 Dec  7 05:34 844cae61-9599-4a57-9431-18c0a33904e4 -> ../../sda1
lrwxrwxrwx 1 root root 10 Dec  7 05:34 9860056e-b314-4cb9-acc5-30e76281795b -> ../../sdd1
lrwxrwxrwx 1 root root 10 Dec  7 04:33 d5ac7faa-876c-43ff-b27a-94c53bf79bb2 -> ../../dm-0
```
### **Mounting**
- **Create Mount points:**
```bash
# Must make the folder to mount drives to
sudo mkdir -p /mnt/disk1
sudo mkdir -p /mnt/disk2
sudo mkdir -p /mnt/parity1
sudo mkdir -p /mnt/appdata

# Mount Parition
sudo mount /dev/sdb1 /mnt/parity1
sudo mount /dev/sdc1 /mnt/disk1
sudo mount /dev/sdd1 /mnt/disk2

# Verify Mounting 
df -h 
# Filesystem      Size  Used Avail Use% Mounted on
# ...
# /dev/sda1       7.3T   93M  6.9T   1% /mnt/mydrive

```

- **Edit fstab**
- Scroll to bottom of fstab and add: 
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
- ##### ***NOTE:  MUST RUN AFTER EDITING fstab***
```bash
# verify syntax 
sudo findmnt --verify

# reload fstab
sudo systemctl daemon-reload
```
### Verify 
- **Test this BEFORE reboot** - no output is good
```bash
sudo mount -a
```

- **Check correct sizing:**
```bash
df -h
```

 You should see:

- `disk1`, `disk2`, `parity1`: ~11T
    
- `appdata`: ~220G
  
- **Check ownership:**
```bash
ls -ld /mnt/disk1 /mnt/disk2 /mnt/parity1 /mnt/appdata
```

- If owner listed as `root` (and doesnt need to be) run :
```bash 
sudo chown -R goose:goose /mnt/disk1 /mnt/disk2 /mnt/parity1 /mnt/appdata
```


## **2. Install MergerFS**

Read the official docs. There is a lot to go over with mergerFS. 

```bash
sudo apt update && sudo apt install mergerfs -y

```

**Configure Pool in Fstab**
Add this line to the bottom of `/etc/fstab`:

```bash
# Syntax: /mnt/disk* (all mounts starting with disk) -> /mnt/storage
/mnt/hdd/WD-B00WHLZD:/mnt/hdd/WD-B00WUHXD /mnt/storage fuse.mergerfs cache.files=off,category.create=mspmfs,func.getattr=newest,dropcacheonclose=false,minfreespace=100G,moveonenospc=true,fsname=mergerfs 0 0


```

* **minfreespace=50G:** Prevents filling a drive completely, which creates fragmentation issues.
* **cache.files=off:** Disables page caching (reduces RAM usage/complexity).
* **category.create=mspmfs:** Most Space, Path Most Free Space. Writes new files to the drive with the most free space, unless the path already exists on another drive.
* **func.getattr=newest:** Returns file attributes from the file with the newest mtime. Critical for apps like Plex/Kodi to detect changes.
* **minfreespace=200G:** Prevents filling a drive completely; moves to the next drive when 200GB remains.


**C. Activate**

```bash
sudo mkdir -p /mnt/storage
sudo mount -a

```

---

## 3. SnapRAID 

SnapRAID calculates parity (like RAID 5) but does it on a schedule (Snapshot), not in real-time. Ideally suited for media servers where files rarely change.

**A. Install SnapRAID**
Ubuntu repos often have outdated versions. It is safer to build from source or download the latest .deb.

```bash
# Check version in repo or download from official site
sudo apt install snapraid -y
snapraid --version

```

**B. Configure `/etc/snapraid.conf**`
Backup the default and create your own: `sudo nano /etc/snapraid.conf`

```text
# 1. Parity Location (The dedicated large drive)
parity /mnt/parity1/snapraid.parity

# 2. Content Files (Index of all your files)
# Save multiple copies on different physical disks!
content /var/snapraid.content
content /mnt/disk1/.snapraid.content
content /mnt/disk2/.snapraid.content

# 3. Data Disks ( The drives you want to protect)
data d1 /mnt/disk1/
data d2 /mnt/disk2/

# 4. Excludes (Temp files/Trash)
exclude *.unrecoverable
exclude /tmp/
exclude /lost+found/

```

**C. Initialize**
Run the first sync to calculate parity. This will take hours depending on data size.

```bash
sudo snapraid sync

```

---

### 5. Automation & Maintenance

SnapRAID is manual by default. You must automate it to stay protected.

**A. Basic Crontab**
Edit cron: `sudo crontab -e`
Add a nightly sync and weekly scrub (checks for bit-rot).

```bash
# Sync every night at 3 AM
0 3 * * * snapraid sync >> /var/log/snapraid.log

# Scrub (check data integrity) every Sunday at 5 AM
0 5 * * 0 snapraid scrub -p 5 >> /var/log/snapraid_scrub.log

```

* **-p 5:** Scrubs 5% of the array each week, ensuring a full check every ~5 months.

---

### Troubleshooting

* **"Disk is full" during sync:** Your parity drive must be equal to or larger than your single largest data drive.
* **Data Recovery:** If `disk1` fails, replace it, mount to `/mnt/disk1`, and run `snapraid fix -d d1`.
