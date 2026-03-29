---
icon: lucide/share-2
---

# Samba

**Samba** is a [free software](https://en.wikipedia.org/wiki/Free_software "Free software") implementation of the [SMB](https://en.wikipedia.org/wiki/Server_Message_Block "Server Message Block") [networking](https://en.wikipedia.org/wiki/Computer_network "Computer network") [protocol](https://en.wikipedia.org/wiki/Protocol_(computing) "Protocol (computing)"), and was originally developed by [Andrew Tridgell](https://en.wikipedia.org/wiki/Andrew_Tridgell "Andrew Tridgell").

Server Message Block (SMB) enables [file sharing](https://en.wikipedia.org/wiki/File_sharing "File sharing"), [printer sharing](https://en.wikipedia.org/wiki/Print_server "Print server"), network browsing, and [inter-process communication](https://en.wikipedia.org/wiki/Inter-process_communication "Inter-process communication") (through [named pipes](https://en.wikipedia.org/wiki/Named_pipe "Named pipe")) over a [computer network](https://en.wikipedia.org/wiki/Computer_network "Computer network").

I am primarily using Samba as my local file sharing as NFS causes issues with MergerFS, which is what my NAS uses. Samba also allows for easy sharing across Linux and Windows devices. Outside of these 2 issues, NFS is excellent, but Samba just works better in my situation. 

---

## Installation 

This is for creating a share on a NAS, then sharing that as a mount to other machines.

### Server install (on NAS) 

Server install

`sudo apt update && sudo apt install samba -y`

Client Install (this will be needed, install it)

`sudo apt install samba-client -y` 

Confirm it is running:

`sudo systemctl status smbd`

Create the dir to be used

`sudo mkdir -p /mnt/storage/backups`

Change ownership

`sudo chown username:username /mnt/storage/backups`

Edit the samba config

`sudo nano /etc/samba/smb.conf`

Add this block to the bottom

```ini
[backups]
   path = /mnt/storage/backups
   browseable = yes
   read only = no
   valid users = username
```

Save and exit. Then set a samba password for your user.

`sudo smbpasswd -a username`

Restart to apply the config 

`sudo systemctl restart smbd`

Verify share is visible

`smbclient -L localhost -U username`

### Mounting to machine

First, install the CIFS utilities on the machine

`sudo apt install cifs-utils -y`

Create mount point

`sudo mkdir -p /mnt/nas`

Now create a credentials file so the password isn't in fstab.

```bash
sudo nano /etc/samba/credentials
```

Add:
```bash
username=username
password=<your samba password>
```

Lock down permissions.

`sudo chmod 600 /etc/samba/credentials`

Add the mount to fstab

```bash
sudo nano /etc/fstab
```

Add this line at the bottom:
```
//10.10.30.30/backups /mnt/nas cifs credentials=/etc/samba/credentials,uid=1000,gid=1000,iocharset=utf8 0 0

# OR - Mount using hostname. Add _netdev to options so system waits for the network before mounting. 

//hostname.internal/backups /mnt/nas cifs credentials=/etc/samba/credentials,uid=1000,gid=1000,iocharset=utf8,_netdev 0 0
```

Save & Test

`sudo mount -a && df -h | grep nas`

You should see:

```
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
//nas.internal/backups           22T   17T  5.7T  74% /mnt/nas
```

reload deamon

`sudo systemctl daemon-reload`

Repeat this for any other machines you need to mount the share to. 

---

### Complete instruction for client

```bash
sudo apt install cifs-utils -y
sudo mkdir -p /mnt/nas
sudo nano /etc/samba/credentials
```

```bash
sudo chmod 600 /etc/samba/credentials
sudo nano /etc/fstab
```

fstab entry:
```
//hostname.internal/backups /mnt/nas cifs credentials=/etc/samba/credentials,uid=1000,gid=1000,iocharset=utf8,_netdev 0 0
```

Then:
```bash
sudo mount -a && sudo systemctl daemon-reload && df -h | grep nas
```

