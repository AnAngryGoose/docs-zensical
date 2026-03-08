---
icon: simple/borgbackup
---

# Borg / Borgmatic

borgmatic is simple, configuration-driven backup software for servers and workstations. Protect your files with client-side encryption. Backup your databases too. Monitor it all with integrated third-party services.

https://torsion.org/borgmatic/

---

## Installation 

To install borgmatic, first [install Borg](https://borgbackup.readthedocs.io/en/stable/installation.html), at least version 1.1. (borgmatic does not install Borg automatically so as to avoid conflicts with existing Borg installations.)

`sudo apt install borgbackup`

Then, [install pipx](https://pypa.github.io/pipx/installation/) as the root user (with `sudo`) to make installing borgmatic easier without impacting other Python applications on your system. If you have trouble installing pipx with pip, then you can install a system package instead. E.g. on Ubuntu or Debian, run:

```bash
sudo apt update
sudo apt install pipx
```
### Root Install 

If you want to run borgmatic on a schedule with privileged access to your files, then you should install borgmatic as the root user by running the following commands:

```bash
sudo pipx ensurepath
sudo pipx install borgmatic
```

Check whether this worked with:

```bash
sudo su -
borgmatic --version
```

If borgmatic is properly installed, that should output your borgmatic version. And if you'd also like sudo borgmatic to work, keep reading!

### Non-Root Install 

!!! success "Preferred - run as sudo"
If you only want to run borgmatic as a non-root user (without privileged file access) or you want to make `sudo borgmatic` work so borgmatic runs as root, then install borgmatic as a non-root user by running the following commands as that user:

```bash
pipx ensurepath
pipx install borgmatic
```
Reload your shell

```bash
source ~/.zshrc # or ~/.bashrc
```
Verify 

```bash
borgmatic --version # to verify
```

This should work even if you've also installed borgmatic as the root user.

If borgmatic is properly installed, that should output your borgmatic version. You can also try `sudo borgmatic --version` if you intend to run borgmatic with sudo. If that doesn't work, you may need to update your sudoers secure_path option.

### Other ways to Install

You  can also install via docker/podman. The official docs cover this. I chose to just run it natively as it is a root privledged system backup that manages files. Running via a container can work fine, just a bit extra work. I like to run infrastructue services outside of a container, if those services manage containers. Either way works. 


## Configuration

Generate the defualt config:

`sudo borgmatic config generate --destination /etc/borgmatic/config.yaml`

!!!note
    If you are met with a "`sudo: borgmatic: command not found`" message, and wish to run it as sudo, you may need to update your sudoers secure_path option.

    ```bash
    sudo visudo
    ```

    Find the line that says `Defaults secure_path=...` and add `/home/user/.local/bin` to it, for example:

    ```bash 
    Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/home/user/.local/bin"
    ```
You should see:

```
Generating a configuration file at: /etc/borgmatic/config.yaml

This includes all available configuration options with example values, the few
required options as indicated. Please edit the file to suit your needs.

If you ever need help: https://torsion.org/borgmatic/#issues

summary:
Generate successful
```

### Edit Configuration

Edit the config file.

`sudo micro /etc/borgmatic/config.yaml`

I setup a mounted SMB/CIFS drive to my NAS. This is what I'll be using to backup. You can find info on that [here](samba)

For a basic config with a local/mounted share you can add:

```yaml
source_directories:
  - /opt/docker

repositories:
  - path: /mnt/nas/prod-deb-01
    label: nas

keep_daily: 7
```

#### Create a passphrase

You can add a plaintext passphrase, but I prefer a seperate file. 

Create the passphrase file:

```bash
sudo mkdir -p /etc/borgmatic
sudo nano /etc/borgmatic/passphrase
```

Add a strong passphrase, then lock permissions:

```bash
sudo chmod 600 /etc/borgmatic/passphrase
```

Then in the config set:

```yaml 
encryption_passcommand: cat /etc/borgmatic/passphrase
```

Validate the config file:

`sudo borgmatic config validate`

Intialize repo (run on current machine TO BE backed up)

`sudo borgmatic init --encryption repokey-blake2`

!!!note 
    `repokey-blake2` stores the key inside the repo itself - key was auto created, stored at /mnt/nas/hostname. Passphrase will unlock it.

    As a precaution you can export the key now. If the NAS dies you'll lose both repo and key. You can export it via:

    ```bash
    sudo borg key export /mnt/nas/hostname /etc/borgmatic/repo-key-backup
    sudo chmod 600 /etc/borgmatic/repo-key-backup
    ```

## Run first backup

```bash
sudo borgmatic create --verbosity 1 --list --stats 
``` 

The `--verbosity` flag makes borgmatic show the steps it's performing. 

The `--list` flag lists each file that's new or changed since the last backup. 

And `--stats` shows summary information about the created archive. All of these flags are optional.



!!!note
    This can take awhile depending on size of dir to be backed up. 

## Automate Backups

### Cron 

I prefer `cron` to automate my services. Systemd is also an option, read [here](https://torsion.org/borgmatic/how-to/set-up-backups/#systemd).

Download the sample cron file.

```bash
sudo curl -o /etc/cron.d/borgmatic https://projects.torsion.org/borgmatic-collective/borgmatic/raw/branch/main/sample/cron/borgmatic
sudo chmod +x /etc/cron.d/borgmatic
```

Edit the path 

```bash
sudo nano /etc/cron.d/borgmatic
```
Change `/root/.local/bin/borgmatic` to `/home/hostname/.local/bin/borgmatic.`

Save and exit. Machine is good to go. 

## Notifications

Borgmatic supports this natively via `ntfy` or generic `webhook` hooks. Official way is via **Apprise**, which borgmatic natively supports since version 1.8.4. It supports Discord among many other services. Since you installed borgmatic with pipx, you need to reinstall it with the Apprise extra:

```bash
pipx uninstall borgmatic # no sudo as this was installed as non-root
pipx install 'borgmatic[Apprise]'
```

Then add this to your `/etc/borgmatic/config.yaml`:

```yaml
apprise:
  services:
    - url: discord://webhook_id/webhook_token
      label: discord
  send_logs: true # for logs in notification
  states:
    - finish
    - fail
```

The Discord URL format is `discord://webhook_id/webhook_token` — you get both values from your Discord webhook URL which looks like `https://discord.com/api/webhooks/{webhook_id}/{webhook_token}`.
