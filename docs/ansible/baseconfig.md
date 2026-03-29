# Ansible Base Provisioning -- Homelab

Complete base provisioning of a new Debian host using Ansible. Covers system setup, Docker, Tailscale, NAS backup mounts, and automated backups with Borgmatic.

Ref: https://docs.ansible.com/projects/ansible/latest/getting_started/index.html

## Prerequisites

- Ansible installed on your control node (workstation/laptop)
- Dedicated Ansible SSH key
- Target host has Debian 13 (Trixie) installed
- Target host has your user account created during OS install
- Target host is reachable via SSH from the control node

## Pre-Ansible Setup on Target Host

SSH into the new host and install sudo (fresh Debian minimal doesn't include it):

```bash
ssh youruser@hostname
su -
apt update && apt install -y sudo
echo 'youruser ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/youruser
chmod 440 /etc/sudoers.d/youruser
exit
exit
```

Generate and copy the Ansible SSH key from the control node:

```bash
ssh-keygen -t ed25519 -C "ansible" -f ~/.ssh/ansible -N ""
ssh-copy-id -i ~/.ssh/ansible.pub youruser@hostname
```

## Project Structure

```
~/ansible/
  ansible.cfg
  .vault_pass
  inventory/
    hosts.ini
    group_vars/
      all.yml
      vault.yml          # encrypted with ansible-vault
  playbooks/
    base.yml
  roles/
    base/
      tasks/main.yml
      handlers/main.yml
      templates/sshd_config.j2
    docker/
      tasks/main.yml
      handlers/main.yml
      templates/daemon.json.j2
    tailscale/
      tasks/main.yml
    mounts/
      tasks/main.yml
    borgmatic/
      tasks/main.yml
      templates/
        borgmatic-config.yml.j2
        borgmatic.cron.j2
```

Create the skeleton:

```bash
mkdir -p ~/ansible
cd ~/ansible
mkdir -p inventory/group_vars inventory/host_vars
mkdir -p playbooks
mkdir -p roles/base/{tasks,handlers,templates}
mkdir -p roles/docker/{tasks,handlers,templates}
mkdir -p roles/tailscale/tasks
mkdir -p roles/mounts/{tasks,templates}
mkdir -p roles/borgmatic/{tasks,templates}
```

## Configuration Files

### ansible.cfg

```ini
[defaults]
inventory = inventory/hosts.ini
remote_user = youruser
private_key_file = ~/.ssh/ansible
roles_path = roles
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml
vault_password_file = .vault_pass

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

Settings breakdown:

- `inventory` -- Default inventory file so you don't pass `-i` every time.
- `remote_user` -- SSH user on all managed nodes.
- `private_key_file` -- The Ansible SSH key.
- `host_key_checking = False` -- Don't prompt on first SSH. Safe on a local network.
- `retry_files_enabled = False` -- Don't create `.retry` files on failure.
- `stdout_callback = yaml` -- More readable output format.
- `become = True` -- Use sudo by default.
- `vault_password_file` -- Points to the file containing your vault decryption password.

### inventory/hosts.ini

Organize hosts into groups by role. Adjust hostnames to match your environment.

```ini
[infra]
infra-01.example.local

[apps]
app-01.example.local

[standby]
dev-01.example.local

[nas]
nas-01.example.local

# Parent group containing all managed nodes
[homelab:children]
infra
apps
standby
```

When you target `homelab`, Ansible hits all three groups. When you target `infra`, only the infra hosts.

Ref: https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html

### inventory/group_vars/all.yml

Variables shared across all hosts. Adjust values to match your setup.

```yaml
---
# User account on all managed nodes
main_user: youruser

# Timezone and locale
timezone: America/Chicago
locale: en_US.UTF-8

# DNS server for Docker containers (your gateway/DNS server IP)
docker_dns: "10.0.0.1"

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

# Directory structure
docker_compose_dir: /opt/docker
appdata_dir: /mnt/appdata
backup_mount: /mnt/backups

# Borgmatic backup config
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

# NAS/backup server connection
nas_backup_host: "nas-01.example.local"
nas_backup_share: "backups"
nas_creds_file: /etc/samba/nas-creds
nas_samba_user: youruser
```

### Ansible Vault -- Storing Secrets

Passwords and secrets go in an encrypted vault file, never in plain text.

```bash
# Create a vault password file (stores the encryption password)
nano ~/ansible/.vault_pass
# Type a password, save
chmod 600 ~/ansible/.vault_pass

# Create the encrypted secrets file
ansible-vault create inventory/group_vars/vault.yml
```

Put your secrets in the vault file:

```yaml
---
vault_nas_samba_password: "your-nas-password"
```

The vault file is loaded explicitly in the playbook via `vars_files`.

Common vault commands:

```bash
# Edit secrets
ansible-vault edit inventory/group_vars/vault.yml

# View without editing
ansible-vault view inventory/group_vars/vault.yml

# Change the encryption password
ansible-vault rekey inventory/group_vars/vault.yml
```

Ref: https://docs.ansible.com/projects/ansible/latest/vault_guide/index.html

## Roles

### base

Installs packages, hardens SSH, sets timezone, creates directory structure.

#### roles/base/tasks/main.yml

```yaml
---
- name: Set timezone
  community.general.timezone:
    name: "{{ timezone }}"

- name: Set locale
  ansible.builtin.locale_gen:
    name: "{{ locale }}"
    state: present

- name: Update apt cache
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Install base packages
  ansible.builtin.apt:
    name: "{{ base_packages }}"
    state: present

- name: Create directory structure
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    owner: "{{ main_user }}"
    group: "{{ main_user }}"
    mode: "0755"
  loop:
    - "{{ docker_compose_dir }}"
    - "{{ appdata_dir }}"
    - "{{ backup_mount }}"

- name: Configure SSH daemon
  ansible.builtin.template:
    src: sshd_config.j2
    dest: /etc/ssh/sshd_config
    owner: root
    group: root
    mode: "0644"
    validate: "sshd -t -f %s"
  notify: restart sshd

- name: Ensure passwordless sudo for main user
  ansible.builtin.copy:
    content: "{{ main_user }} ALL=(ALL) NOPASSWD:ALL\n"
    dest: "/etc/sudoers.d/{{ main_user }}"
    owner: root
    group: root
    mode: "0440"
    validate: "visudo -cf %s"
```

#### roles/base/handlers/main.yml

```yaml
---
- name: restart sshd
  ansible.builtin.service:
    name: sshd
    state: restarted
```

#### roles/base/templates/sshd_config.j2

```
# Managed by Ansible
Port 22
AddressFamily any
ListenAddress 0.0.0.0
ListenAddress ::

PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

ChallengeResponseAuthentication no
UsePAM yes

X11Forwarding no
PrintMotd no

AcceptEnv LANG LC_*

Subsystem sftp /usr/lib/openssh/sftp-server
```

### docker

Installs Docker CE from the official repository and configures log rotation.

Ref: https://docs.docker.com/engine/install/debian/

#### roles/docker/tasks/main.yml

```yaml
---
- name: Remove old Docker packages
  ansible.builtin.apt:
    name:
      - docker
      - docker-engine
      - docker.io
      - containerd
      - runc
    state: absent

- name: Install Docker prerequisites
  ansible.builtin.apt:
    name:
      - ca-certificates
      - curl
      - gnupg
    state: present

- name: Create keyrings directory
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    mode: "0755"

- name: Add Docker GPG key
  ansible.builtin.shell: |
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    chmod a+r /etc/apt/keyrings/docker.gpg
  args:
    creates: /etc/apt/keyrings/docker.gpg

- name: Add Docker apt repository
  ansible.builtin.copy:
    content: |
      deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian {{ ansible_distribution_release }} stable
    dest: /etc/apt/sources.list.d/docker.list
    mode: "0644"

- name: Update apt cache after adding Docker repo
  ansible.builtin.apt:
    update_cache: yes

- name: Install Docker CE
  ansible.builtin.apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-compose-plugin
    state: present

- name: Add user to docker group
  ansible.builtin.user:
    name: "{{ main_user }}"
    groups: docker
    append: yes

- name: Configure Docker daemon
  ansible.builtin.template:
    src: daemon.json.j2
    dest: /etc/docker/daemon.json
    owner: root
    group: root
    mode: "0644"
  notify: restart docker

- name: Ensure Docker is running and enabled
  ansible.builtin.service:
    name: docker
    state: started
    enabled: yes
```

#### roles/docker/handlers/main.yml

```yaml
---
- name: restart docker
  ansible.builtin.service:
    name: docker
    state: restarted
```

#### roles/docker/templates/daemon.json.j2

Ref: https://docs.docker.com/engine/logging/drivers/json-file/

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

### tailscale

Installs Tailscale, leaves it stopped. Available as an emergency backdoor if the router is down.

Ref: https://tailscale.com/kb/1174/install-debian-bookworm

#### roles/tailscale/tasks/main.yml

```yaml
---
- name: Get Debian release name
  ansible.builtin.command: lsb_release -cs
  register: debian_release
  changed_when: false

- name: Add Tailscale GPG key
  ansible.builtin.shell: |
    curl -fsSL https://pkgs.tailscale.com/stable/debian/{{ debian_release.stdout }}.noarmor.gpg \
      -o /usr/share/keyrings/tailscale-archive-keyring.gpg
  args:
    creates: /usr/share/keyrings/tailscale-archive-keyring.gpg

- name: Add Tailscale apt repository
  ansible.builtin.copy:
    content: |
      deb [signed-by=/usr/share/keyrings/tailscale-archive-keyring.gpg] https://pkgs.tailscale.com/stable/debian {{ debian_release.stdout }} main
    dest: /etc/apt/sources.list.d/tailscale.list
    mode: "0644"
  register: tailscale_repo

- name: Update apt cache
  ansible.builtin.apt:
    update_cache: yes
  when: tailscale_repo.changed

- name: Install Tailscale
  ansible.builtin.apt:
    name: tailscale
    state: present

- name: Ensure Tailscale is stopped and disabled
  ansible.builtin.service:
    name: tailscaled
    state: stopped
    enabled: no
```

### mounts

Mounts a NAS backup share via CIFS. Skip this role if you don't have a NAS.

#### roles/mounts/tasks/main.yml

```yaml
---
- name: Create samba credentials directory
  ansible.builtin.file:
    path: /etc/samba
    state: directory
    owner: root
    group: root
    mode: "0700"

- name: Template NAS credentials file
  ansible.builtin.copy:
    content: |
      username={{ nas_samba_user }}
      password={{ vault_nas_samba_password }}
    dest: "{{ nas_creds_file }}"
    owner: root
    group: root
    mode: "0600"

- name: Mount NAS backup share
  ansible.posix.mount:
    path: "{{ backup_mount }}"
    src: "//{{ nas_backup_host }}/{{ nas_backup_share }}"
    fstype: cifs
    opts: "credentials={{ nas_creds_file }},uid=1000,gid=1000,iocharset=utf8"
    state: mounted
```

### borgmatic

Installs borgmatic, configures backups, initializes the Borg repository.

#### roles/borgmatic/tasks/main.yml

```yaml
---
- name: Install borgmatic via pip
  ansible.builtin.pip:
    name: borgmatic
    extra_args: --break-system-packages
    state: present

- name: Create borgmatic config directory
  ansible.builtin.file:
    path: /etc/borgmatic
    state: directory
    owner: root
    group: root
    mode: "0700"

- name: Template borgmatic config
  ansible.builtin.template:
    src: borgmatic-config.yml.j2
    dest: /etc/borgmatic/config.yaml
    owner: root
    group: root
    mode: "0600"

- name: Template borgmatic cron job
  ansible.builtin.template:
    src: borgmatic.cron.j2
    dest: /etc/cron.d/borgmatic
    owner: root
    group: root
    mode: "0644"

- name: Check if borg repo is initialized
  ansible.builtin.stat:
    path: "{{ backup_mount }}/{{ inventory_hostname_short }}/README"
  register: borg_repo_check

- name: Initialize borg repo
  ansible.builtin.command: >
    borg init --encryption=repokey
    {{ backup_mount }}/{{ inventory_hostname_short }}
  environment:
    BORG_PASSPHRASE: ""
  when: not borg_repo_check.stat.exists
```

#### roles/borgmatic/templates/borgmatic-config.yml.j2

```yaml
# Managed by Ansible
source_directories:
{% for dir in borg_backup_targets %}
  - {{ dir }}
{% endfor %}

repositories:
  - path: {{ backup_mount }}/{{ inventory_hostname_short }}
    label: nas

exclude_patterns:
{% for pattern in borg_excludes %}
  - {{ pattern }}
{% endfor %}

encryption_passphrase: ""

retention:
  keep_daily: 7
  keep_weekly: 4
  keep_monthly: 6

consistency:
  checks:
    - repository
    - archives
  check_last: 3
```

#### roles/borgmatic/templates/borgmatic.cron.j2

```
# Managed by Ansible
0 3 * * * root PATH=$PATH:/usr/bin:/usr/local/bin borgmatic --verbosity -1 --syslog-verbosity 1 --list --stats
```

## Playbook

### playbooks/base.yml

```yaml
---
- name: Base system configuration
  hosts: homelab
  become: yes
  vars_files:
    - ../inventory/group_vars/vault.yml
  roles:
    - { role: base, tags: ['base'] }
    - { role: docker, tags: ['docker'] }
    - { role: tailscale, tags: ['tailscale'] }
    - { role: mounts, tags: ['mounts'] }
    - { role: borgmatic, tags: ['borgmatic'] }
```

## Running It

```bash
# Test connectivity to all hosts
ansible homelab -m ping

# Dry run against a specific host (shows what would change)
ansible-playbook playbooks/base.yml --check --limit infra-01.example.local

# Run everything against a specific host
ansible-playbook playbooks/base.yml --limit infra-01.example.local

# Run only a specific role using tags
ansible-playbook playbooks/base.yml --tags docker --limit infra-01.example.local

# Run everything except one role
ansible-playbook playbooks/base.yml --skip-tags borgmatic --limit infra-01.example.local

# Run against all hosts
ansible-playbook playbooks/base.yml

# Verify idempotency (second run should show changed=0)
ansible-playbook playbooks/base.yml --limit infra-01.example.local

# Run ad-hoc commands against hosts
ansible homelab -a "uptime"
ansible infra-01.example.local -a "docker ps"
ansible homelab -a "df -h /mnt/appdata"
```

## What Gets Provisioned

After running the full base playbook, each host has:

- Base packages (curl, git, htop, jq, tmux, etc.)
- SSH hardened (key-only auth, no root login, no password auth)
- Passwordless sudo for your user
- Docker CE + compose plugin + log rotation
- Tailscale installed but stopped
- NAS backup mount via CIFS
- Borgmatic configured with Borg repo initialized, cron at 03:00 daily
- Directory structure: /opt/docker, /mnt/appdata, /mnt/backups

## Key Concepts

**Idempotency.** Running the same playbook twice produces the same result. The second run should show `changed=0`. If a task always shows `changed`, it's not idempotent and should be fixed with `creates:` or `when:` conditions.

**Tags.** Labels on roles or tasks that let you run subsets of a playbook. `--tags docker` runs only the docker role. `--skip-tags borgmatic` runs everything except borgmatic.

**Limit.** `--limit hostname` restricts execution to a single host or group. Without it, the playbook runs against all hosts matched by the `hosts:` directive.

**Check mode.** `--check` shows what would change without making changes. Always dry run before applying to production.

**Handlers.** Tasks that only run when notified. If the SSH config template changes, the handler restarts sshd. If the template is unchanged, sshd is not restarted. This prevents unnecessary service restarts.

**Vault.** Encrypted storage for secrets. The vault password file decrypts automatically. Never commit `.vault_pass` or unencrypted secrets to git.

## Gotchas

- Fresh Debian minimal does not include `sudo`. Install it manually before running Ansible.
- Don't use shell heredocs (`cat << EOF`) to write files containing Jinja2 `{{ }}` syntax. The shell escapes the braces. Use a text editor directly.
- Vault files in `group_vars/` are only auto-loaded if the filename matches a group name. Otherwise, load them explicitly with `vars_files` in the playbook.
- The `command` and `shell` modules always report `CHANGED` even for read-only commands. This is normal and doesn't mean the system was modified.
- Ansible connects via the SSH key specified in `ansible.cfg`. If you rotate keys, update `private_key_file`.