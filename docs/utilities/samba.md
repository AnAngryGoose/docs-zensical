# Samba

## Installation 

Server install: 

`sudo apt update && sudo apt install samba -y`

Client Install (this will be needed, install it)

`sudo apt install samba-client -y` 

Confirm it is running:

`sudo systemctl status smbd`

Create the dir to be used

`sudo mkdir -p /mnt/storage/backups`

Change ownership

`sudo chown goose:goose /mnt/storage/backups`

Edit the samba config

`sudo nano /etc/samba/smb.conf`

Add this block to the bottom

```ini
[backups]
   path = /mnt/storage/backups
   browseable = yes
   read only = no
   valid users = goose
```

Save and exit. Then set a samba password for your user.

`sudo smbpasswd -a goose`

Restart to apply the config 

`sudo systemctl restart smbd`

Verify share is visible

`smbclient -L localhost -U goose`
