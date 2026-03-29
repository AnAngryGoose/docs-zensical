# Variables

Variables let you define values once and reuse them. Ansible loads them automatically based on the file location:

- `group_vars/all.yml` applies to every host.
- `group_vars/infra.yml` applies only to hosts in the `[infra]` group.
- `host_vars/nas.internal.yml` applies only to nas.

### inventory/group_vars/all.yml

Shared across all hosts.

```yaml
---
# User
main_user: username

# Timezone
timezone: America/Chicago

# Locale
locale: en_US.UTF-8

# DNS for Docker containers (LAB VLAN gateway)
docker_dns: "10.10.30.1"

# Packages to install on all hosts
base_packages:
  - sudo
  - curl
  - wget
  - vim
  - nano
  - tmux
  - git
  - htop
  - dnsutils
  - unzip
  - tar
  - rsync
  - tree
  - ncdu
  - jq
  - lsof
  - iotop
  - sysstat
  - python3-pip
  - ca-certificates
  - gnupg
  - lsb-release
  - borgbackup
  - cifs-utils

# Docker directory structure
docker_compose_dir: /opt/docker
appdata_dir: /mnt/appdata
backup_mount: /mnt/backups

# Borgmatic
borg_backup_targets:
  - /opt/docker
  - /mnt/appdata
  - /etc
borg_excludes:
  - "*.log"
  - "cache/"
  - "Cache/"
  - ".cache/"
  - "__pycache__/"
  - "node_modules/"

# NAS backup
nas_backup_host: "nas.internal"
nas_backup_share: "backups"
nas_creds_file: /etc/samba/nas-creds
nas_samba_user: username
# nas_samba_password is stored in ansible vault (see below)
```

### inventory/group_vars/infra.yml

Specific to ops-01.

```yaml
---
# Infra-specific stacks to deploy
infra_stacks:
  - npm
  - cloudflared
  - homeassistant
  - monitoring
  - homepage
  - code-server
  - vaultwarden
```

### inventory/group_vars/apps.yml

Specific to prod-deb-01.

```yaml
---
# App-specific stacks to deploy
app_stacks:
  - affine
  - vikunja
  - trilium
  - utilities
  - omada
  - paperless
  - forgejo
  - socket-proxy
```

### inventory/host_vars/nas.internal.yml

nas-specific overrides (not fully managed by Ansible, just borgmatic + cleanup).

```yaml
---
docker_compose_dir: /opt/docker
appdata_dir: /mnt/appdata
# nas doesn't mount NAS backups (it IS the NAS)
skip_nas_mount: true
borg_repo_path: /mnt/storage/backups/borg/nas
```

---

