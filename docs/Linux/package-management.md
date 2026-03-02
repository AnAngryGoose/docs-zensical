---
title: Package Management
description: Installing, updating, and managing packages with apt on Debian and Ubuntu.
---

# Package Management

> **Reference:** [Debian Wiki — apt](https://wiki.debian.org/Apt) · [Ubuntu Docs — apt](https://help.ubuntu.com/community/AptGet/Howto) · [Debian Policy Manual](https://www.debian.org/doc/debian-policy/)

---

## apt — Core Commands

`apt` is the primary package manager for both Debian and Ubuntu. It's a higher-level frontend to `dpkg`.

```bash
# Update package index (always run before install/upgrade)
sudo apt update

# Upgrade all installed packages
sudo apt upgrade

# Full upgrade — also handles dependency changes and removals
sudo apt full-upgrade

# Install a package
sudo apt install packagename

# Install multiple packages
sudo apt install nginx curl git htop

# Remove a package (keep config files)
sudo apt remove packagename

# Remove a package and its config files
sudo apt purge packagename

# Remove unused dependencies
sudo apt autoremove

# Remove unused dependencies and purge configs
sudo apt autoremove --purge

# Clean downloaded package cache
sudo apt clean          # removes all cached .deb files
sudo apt autoclean      # removes only outdated cached .deb files
```

---

## Searching & Inspecting

```bash
# Search for a package by name or description
apt search keyword

# Show package details (version, dependencies, description)
apt show packagename

# List installed packages
apt list --installed

# List upgradable packages
apt list --upgradable

# Find which package provides a file
dpkg -S /usr/bin/curl

# List files installed by a package
dpkg -L packagename

# Check if a package is installed
dpkg -l packagename
```

---

## Pinning & Held Packages

Pinning prevents a package from being upgraded — useful when a newer version breaks something.

```bash
# Hold a package at current version
sudo apt-mark hold packagename

# Unhold a package
sudo apt-mark unhold packagename

# List all held packages
apt-mark showhold
```

For more granular pinning, create a preferences file:

```bash
# /etc/apt/preferences.d/pin-nginx
Package: nginx
Pin: version 1.24.*
Pin-Priority: 1001
```

---

## Sources & Repositories

```bash
# View current sources
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/

# Add a repository (Ubuntu — adds to sources.list.d/)
sudo add-apt-repository ppa:user/repo    # Ubuntu PPA
sudo add-apt-repository "deb https://repo.example.com/apt stable main"

# Add a repository manually (Debian / Ubuntu without add-apt-repository)
echo "deb https://repo.example.com/apt stable main" | \
    sudo tee /etc/apt/sources.list.d/example.list

# Add a GPG signing key (modern method — keyring file)
curl -fsSL https://repo.example.com/gpg.key | \
    sudo gpg --dearmor -o /usr/share/keyrings/example-keyring.gpg

# Reference keyring in source
echo "deb [signed-by=/usr/share/keyrings/example-keyring.gpg] \
    https://repo.example.com/apt stable main" | \
    sudo tee /etc/apt/sources.list.d/example.list
```

!!! note "Debian vs Ubuntu"
    Ubuntu supports PPAs (`ppa:user/repo`) via `add-apt-repository`. Debian does not — all repos must be added manually. The `add-apt-repository` command is available on Debian but PPA support is not.

---

## Unattended Upgrades

Automatically install security updates — useful for servers.

```bash
# Install
sudo apt install unattended-upgrades

# Configure
sudo dpkg-reconfigure -plow unattended-upgrades

# Config file location
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Key settings in `50unattended-upgrades`:

```
// Auto-reboot after kernel updates (set to false on servers unless planned)
Unattended-Upgrade::Automatic-Reboot "false";

// Email notifications
Unattended-Upgrade::Mail "you@example.com";

// Remove unused dependencies automatically
Unattended-Upgrade::Remove-Unused-Dependencies "true";
```

!!! note "Debian vs Ubuntu"
    On Ubuntu, unattended-upgrades is pre-installed and pre-configured for security updates. On Debian, it must be installed and configured manually.

---

## dpkg — Low-Level Package Tool

`dpkg` is the underlying package manager. Use it when dealing with `.deb` files directly.

```bash
# Install a .deb file
sudo dpkg -i package.deb

# Fix broken dependencies after manual dpkg install
sudo apt install -f

# Remove a package
sudo dpkg -r packagename

# List all installed packages
dpkg -l

# Query package status
dpkg -s packagename

# Extract contents of a .deb without installing
dpkg -x package.deb /tmp/extracted/
```

---

## Useful Combinations

```bash
# Full system update in one line
sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y

# Install a package non-interactively (scripts/automation)
DEBIAN_FRONTEND=noninteractive sudo apt install -y packagename

# Simulate an install without actually doing it
sudo apt install --dry-run packagename

# Download a package without installing
apt download packagename
```

---

## Package Version Management

```bash
# Install a specific version
sudo apt install nginx=1.24.0-1

# List all available versions of a package
apt-cache policy packagename

# Show what version is currently installed vs available
apt-cache showpkg packagename
```