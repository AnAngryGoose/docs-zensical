---
title: Users & Permissions
description: Managing users, groups, and file permissions on Linux.
---

# Users & Permissions

> **Reference:** [Debian Wiki — UserAccounts](https://wiki.debian.org/UserAccounts) · [Ubuntu Docs — User Management](https://help.ubuntu.com/community/AddUsersHowto) · [Linux man-pages: chmod, chown, passwd](https://www.kernel.org/doc/man-pages/)

---

## User Management

### Create a User

```bash
# Create user with home directory
sudo useradd -m -s /bin/bash username

# Create user and add to a group immediately
sudo useradd -m -s /bin/bash -G sudo username

# Set password
sudo passwd username
```

!!! note "Debian vs Ubuntu"
    On Ubuntu, `adduser` (a friendlier wrapper) is preferred and walks you through creation interactively. On Debian it's available but `useradd` is more common in scripts.

    ```bash
    sudo adduser username   # Ubuntu preferred
    ```

---

### Modify a User

```bash
# Add existing user to a group
sudo usermod -aG groupname username

# Change user's shell
sudo usermod -s /bin/bash username

# Change username
sudo usermod -l newname oldname

# Lock / unlock account
sudo usermod -L username   # lock
sudo usermod -U username   # unlock
```

!!! warning
    Always use `-aG` (append) when adding groups. Without `-a`, `usermod -G` **replaces** all existing groups.

---

### Delete a User

```bash
# Remove user (keep home directory)
sudo userdel username

# Remove user and their home directory
sudo userdel -r username
```

---

### View User Info

```bash
id username          # UID, GID, and all groups
whoami               # current user
w                    # who is logged in and what they're doing
cat /etc/passwd      # all users on system
getent passwd username  # single user entry
```

---

## Group Management

```bash
# Create a group
sudo groupadd groupname

# Delete a group
sudo groupdel groupname

# List all groups a user belongs to
groups username

# List all groups on system
cat /etc/group
```

---

## File Permissions

Permissions are displayed as `rwxrwxrwx` — three sets of three for **owner**, **group**, and **other**.

```bash
ls -la /path/to/file
# -rw-r--r-- 1 user group 1234 Jan 1 12:00 file.txt
#  ^ owner   ^ group  ^ other
```

### chmod — Change Permissions

```bash
# Symbolic
chmod u+x script.sh        # add execute for owner
chmod g-w file.txt         # remove write for group
chmod o=r file.txt         # set other to read-only
chmod a+r file.txt         # add read for all (a = all)

# Octal (most common in practice)
chmod 755 script.sh        # rwxr-xr-x  (owner full, group/other read+execute)
chmod 644 file.txt         # rw-r--r--  (owner read/write, group/other read)
chmod 600 .env             # rw-------  (owner only — use this for secrets)
chmod 700 private-dir/     # rwx------  (owner only, directory)
```

| Octal | Binary | Meaning |
|-------|--------|---------|
| 7 | 111 | read + write + execute |
| 6 | 110 | read + write |
| 5 | 101 | read + execute |
| 4 | 100 | read only |
| 0 | 000 | no permissions |

```bash
# Recursive — apply to directory and all contents
chmod -R 755 /var/www/html
```

---

### chown — Change Ownership

```bash
# Change owner
sudo chown username file.txt

# Change owner and group
sudo chown username:groupname file.txt

# Change group only
sudo chown :groupname file.txt

# Recursive
sudo chown -R username:groupname /opt/myapp
```

---

### umask — Default Permission Mask

`umask` controls the default permissions applied to newly created files and directories.

```bash
umask           # view current umask (commonly 022)
umask 027       # set temporarily for session
```

With `umask 022`: new files get `644`, new directories get `755`.
With `umask 027`: new files get `640`, new directories get `750` — group has no write, other has nothing.

To persist, add `umask 027` to `~/.bashrc` or `/etc/profile`.

---

## sudo

```bash
# Run single command as root
sudo command

# Open root shell
sudo -i

# Run as a different user
sudo -u otheruser command

# Edit sudoers safely (never edit /etc/sudoers directly)
sudo visudo
```

### Grant sudo to a User

```bash
# Quickest way — add to sudo group
sudo usermod -aG sudo username

# Verify
groups username
```

!!! note "Debian vs Ubuntu"
    On Ubuntu, the `sudo` group grants full sudo access by default.
    On Debian, the `sudo` group is the same but the `sudo` package may not be installed on minimal installs — install it first: `apt install sudo`.

### Custom sudoers Rule

```bash
sudo visudo
# Add line:
username ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myservice
# Allows user to restart a specific service without password
```

---

## Special Permission Bits

```bash
# Setuid — run file as owner, not caller (e.g. passwd)
chmod u+s /usr/bin/somebinary

# Setgid — new files in dir inherit group
chmod g+s /shared/directory

# Sticky bit — only owner can delete their files (e.g. /tmp)
chmod +t /shared/directory

# View special bits
ls -la   # 's' in owner execute = setuid/setgid, 't' in other execute = sticky
```