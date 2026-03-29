# Samba Quick Reference 

---

# Samba Quick Reference

Config file: `/etc/samba/smb.conf`

---

## Commands

```bash
# Validate config for errors
testparm

# Show active connections and shares
smbstatus

# Restart after config changes
sudo systemctl restart smbd

# Check service status
sudo systemctl status smbd
```

## User Management

Samba users are separate from Linux users. A Linux account must exist first.

```bash
# Add samba password for existing Linux user
sudo smbpasswd -a username

# Change samba password
sudo smbpasswd username

# Disable a samba user
sudo smbpasswd -d username

# Enable a disabled user
sudo smbpasswd -e username

# Delete a samba user
sudo smbpasswd -x username

# List all samba users
sudo pdbedit -L

# List with details (SID, full name, etc.)
sudo pdbedit -Lv
```

## Share Configuration

Shares are defined in `/etc/samba/smb.conf`. Each share is a block:

```ini
[sharename]
    path = /path/to/folder
    read only = No
    valid users = username
    browseable = Yes
    create mask = 0644
    directory mask = 0755
```

Common options:

| Option | Purpose |
|--------|---------|
| `path` | Directory to share |
| `read only` | No = writable, Yes = read only |
| `valid users` | Users allowed access (space separated) |
| `browseable` | Show in network browse lists |
| `guest ok` | Allow anonymous access (avoid this) |
| `create mask` | Permissions for new files |
| `directory mask` | Permissions for new directories |
| `force user` | All access treated as this Linux user |
| `force group` | All access treated as this Linux group |
| `write list` | Users with write access (when read only = Yes) |

## Adding a New Share

1. Create the directory:

```bash
sudo mkdir -p /mnt/storage/newshare
sudo chown username:username /mnt/storage/newshare
```

2. Add to `/etc/samba/smb.conf`:

```ini
[newshare]
    path = /mnt/storage/newshare
    read only = No
    valid users = username
```

3. Validate and restart:

```bash
testparm
sudo systemctl restart smbd
```

## Mounting from a Client

```bash
# Install cifs-utils
sudo apt install cifs-utils

# Create mount point
sudo mkdir -p /mnt/sharename

# Create credentials file
sudo bash -c 'cat > /etc/samba/creds << EOF
username=username
password=yourpassword
EOF'
sudo chmod 600 /etc/samba/creds

# Add to /etc/fstab
//server.internal/sharename /mnt/sharename cifs credentials=/etc/samba/creds,uid=1000,gid=1000,iocharset=utf8 0 0

# Mount now without reboot
sudo mount -a
```

## Testing

```bash
# Test access from another machine
smbclient -L //server.internal -U username

# Connect to a specific share
smbclient //server.internal/sharename -U username

# Verify mount from client side
mount | grep cifs
df -h /mnt/sharename
```

## Troubleshooting

```bash
# Check logs
sudo tail -f /var/log/samba/log.smbd

# Permission denied on mount: verify samba password is set (smbpasswd -a)
# NT_STATUS_ACCESS_DENIED: check valid users in smb.conf
# Mount hangs: check firewall allows ports 139/445
# Stale mount after server reboot: sudo umount -f /mnt/sharename && sudo mount -a
```

Samba uses TCP ports 139 and 445. Both must be reachable from the client.