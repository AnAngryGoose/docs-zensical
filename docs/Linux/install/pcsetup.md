# Server Setup: App Server

* [Debian 12 Bookworm Release Notes :simple-debian:](https://www.debian.org/releases/bookworm/releasenotes)
* [Debian Network Setup :material-lan:](https://wiki.debian.org/NetworkConfiguration)
* [Debian SSH Documentation :material-console:](https://wiki.debian.org/SSH)

---

## 1. Preparation & Media Creation

**A. Download ISO**
Get the **[Small Installation Image (Netinst)](https://www.debian.org/distrib/netinst)**.

!!! note "Architecture"
For Lenovo M920q/M70q nodes, ensure you download the **amd64** ISO.

**B. Flash to USB**
Use **BalenaEtcher**, **Rufus**, or `dd` to write the ISO to a USB drive (4GB+).

---

## 2. BIOS / UEFI Configuration

Mini PCs often ship with settings optimized for Windows. Adjust these for a Linux server.

1. **Enter BIOS:** Power on and rapidly tap the setup key (`F1` for Lenovo).
2. **Secure Boot:** Set to **Disabled**.
* *Reasoning:* While Debian supports Secure Boot, disabling it prevents headaches with third-party drivers (Nvidia) or unsigned kernel modules later.


3. **Power Behavior:** Set "After Power Loss" to **Power On**.
4. **Boot Order:** Prioritize the USB drive.

---

## 3. Installation Process

Boot from USB. The Debian installer uses a classic text-based interface. Navigate with `Arrow Keys`, select with `Enter`, and toggle options with `Space`.

**A. Initial Settings**

* **Install:** Select `Graphical Install` (easier) or `Install` (text-only).
* **Language/Location/Keyboard:** Select defaults.
* **Hostname:** `vanth-node-01` (or your preference).
* **Domain Name:** Leave blank (unless you have a local domain).

**B. User & Password (Crucial)**

!!! warning "Sudo Configuration"
**Leave the 'Root password' field BLANK.**

```
If you leave the root password blank, the installer will automatically install `sudo` and add your new user to the sudo group. If you set a root password now, you will have to manually configure sudo later.

```

* **Full name / Username:** Enter your details (e.g., `goose`).
* **Password:** Set a strong password for this user.

**C. Partitioning**

* **Method:** "Guided - use entire disk and set up LVM".
* **Partition Scheme:** "All files in one partition" (easiest for beginners).
* **Confirm:** Select "Finish partitioning and write changes to disk" -> **Yes**.

**D. Software Selection (Tasksel)**

The installer will install the base system and then ask for additional software.

* **Scan extra media?** No.
* **Package Manager:** Select a mirror close to you (e.g., `deb.debian.org`).
* **Software selection:**
* [ ] Debian desktop environment (Uncheck this)
* [ ] GNOME (Uncheck this)
* [x] **SSH server** (Check this)
* [x] **Standard system utilities** (Check this)



---

## 4. Post-Install Configuration

Remove the USB and reboot. Log in via the physical terminal one last time to get the IP.

### A. Network Configuration (Static IP)

Debian uses `/etc/network/interfaces` by default, not Netplan.

1. **Check Interface Name:**
```bash
ip link
# Look for eno1, eth0, or enp3s0

```


2. **Edit Config:**
```bash
sudo nano /etc/network/interfaces

```


3. **Modify:**
Replace `allow-hotplug eno1` and `iface eno1 inet dhcp` with:
```bash
# The primary network interface
auto eno1
iface eno1 inet static
    address 192.168.1.50/24
    gateway 192.168.1.1
    # DNS servers
    dns-nameservers 1.1.1.1 8.8.8.8

```


4. **Apply:**
```bash
sudo systemctl restart networking

```



### B. Connect via SSH

Switch to your main PC terminal:

```bash
ssh goose@192.168.1.50

```

### C. Update System

```bash
sudo apt update && sudo apt upgrade -y

```

---

## 5. Quality of Life & Security

### A. Install QEMU Guest Agent (If VM)

If this is running as a VM inside Proxmox:

```bash
sudo apt install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent

```

### B. Prevent Sleep

Prevent the Mini PC from suspending when idle:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target

```

### C. Install Vital Tools

Debian "Standard Utilities" is very barebones.

```bash
sudo apt install curl wget git htop vim tmux -y

```

---

## Troubleshooting

* **"Username is not in the sudoers file":** You likely set a root password during install.
* *Fix:* Switch to root (`su -`), then run `usermod -aG sudo your_username`. Reboot.


* **SSH Connection Refused:** Check if the service is running (`sudo systemctl status ssh`). If missing, `sudo apt install openssh-server`.
* **DNS failures:** Check `/etc/resolv.conf`. It should contain `nameserver 1.1.1.1`. If not, your static IP config in `/etc/network/interfaces` might have a syntax error.