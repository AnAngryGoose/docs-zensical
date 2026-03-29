# Ansible 

---

This page covers installation, project structure, every role config, and how to run it all.

This doubles as a quick reference. Use the table of contents to jump to what you need.

Official Ansible docs: https://docs.ansible.com/projects/ansible/latest/getting_started/index.html

---

## How It Works

```
hostname (control node)  --SSH-->  ops-01, prod-01, dev-01 (managed nodes)
        |
   ~/ansible/
   ansible.cfg          # how Ansible behaves (user, key, inventory path)
   inventory/hosts.ini  # what hosts exist and how they're grouped
   group_vars/all.yml   # variables shared by all hosts
   group_vars/vault.yml # encrypted secrets (passwords, tokens)
   playbooks/base.yml   # what to do (calls roles in order)
   roles/base/          # how to do it (tasks, templates, handlers)
```

Ansible reads the **playbook**, which targets **hosts** from the **inventory**, applies **roles** containing **tasks**, and fills in **templates** using **variables**. It connects via SSH and runs everything through sudo.

---

## Table of Contents

1. [Concepts](#concepts)
2. [Installation](#installation)
3. [SSH Key Setup](#ssh-key-setup)
4. [Project Structure](#project-structure)
5. [Configuration Files](./configuration-files)
6. [Inventory](./inventory)
7. [Variables](./variables)
8. [Vault](./vault)
9. [Roles](./roles)
10. [Playbooks](./playbooks)
11. [Running Playbooks](./playbookrunning)
12. [Compose Templates](./compose-templates)
13. [Base Configuration](./baseconfig)
14. [Troubleshooting](./troubleshooting)

---

## Concepts

**Control node** -- The machine that runs Ansible.

**Managed nodes** -- The machines Ansible configures. For you: prod-01, ops-01, dev-01 (and optionally nas).

**Inventory** -- A file listing your managed nodes and grouping them.

**Playbook** -- The entry point. Says "run these roles against these hosts." You execute this.

**Role** -- A folder containing everything needed for one job (install Docker, configure backups). Keeps things modular and reusable.

**Task** -- A single action inside a role. "Install this package." "Copy this file." "Start this service." Runs top to bottom.

**Template** -- A config file with `{{ variables }}` that Ansible fills in before placing on the host. Turns one file into many host-specific versions.

**Handler** -- A task that only fires when notified. "Restart sshd, but only if the config actually changed." Prevents unnecessary restarts.

**Idempotent** -- Running the same playbook twice produces the same result. Ansible only changes things that need changing.

```
You run a playbook
  - playbook targets hosts and calls roles
    - role runs tasks in order (tasks/main.yml)
      - task uses a template (.j2) to place a config file
        - template pulls values from group_vars/host_vars
          - task notifies a handler if something changed
            - handler runs once at the end (e.g., restart service)
```

**On disk:**

```
playbooks/base.yml              # you run this
  calls -> roles/docker/
             tasks/main.yml      # what to do, in order
             templates/daemon.json.j2  # config files with variables
             handlers/main.yml   # triggered reactions
```

**Playbook** calls **role**. **Role** runs **tasks**. **Tasks** use **templates**. **Tasks** notify **handlers**.

Ref: [https://docs.ansible.com/projects/ansible/latest/getting_started/get_started_ansible.html](https://docs.ansible.com/projects/ansible/latest/getting_started/get_started_ansible.html)

--- 

## Installation

Run on **Control Node** only. Managed nodes do not need Ansible installed.

```bash
# Install Ansible from the official PPA
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible

# Verify
ansible --version
```

Ref: https://docs.ansible.com/projects/ansible/latest/installation_guide/installation_distros.html#installing-ansible-on-ubuntu

---

## SSH Key Setup

Ansible uses SSH to connect to managed nodes. You need two keys: your personal key (with passphrase) and a dedicated Ansible key (no passphrase, for automation).

```bash
# Generate a dedicated Ansible key (no passphrase)
ssh-keygen -t ed25519 -C "ansible" -f ~/.ssh/ansible -N ""

# Copy it to each managed node (do this once per host)
ssh-copy-id -i ~/.ssh/ansible.pub username@ops-01.internal
ssh-copy-id -i ~/.ssh/ansible.pub username@prod-deb-01.internal
ssh-copy-id -i ~/.ssh/ansible.pub username@dev-01.internal
ssh-copy-id -i ~/.ssh/ansible.pub username@nas.internal
```

Test that it works without a password prompt:

```bash
ssh -i ~/.ssh/ansible username@ops-01.internal "hostname"
```

If that prints the hostname without asking for a password, you're good.


Test connectivity with Ansible:

```bash
cd ~/ansible
ansible ops-01.internal -m ping
```

!!! warning "Debian Install Issue?"

    Fresh Debian minimal install doesn't include `sudo` by default. You need to install it and configure it. SSH into ops-01 as root:

    ```bash
    ssh root@ops-01.internal "apt update && apt install -y sudo && echo 'username ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/username && chmod 440 /etc/sudoers.d/username"
    ```

    If root SSH is disabled, log in as username and use `su`:

    ```bash
    ssh username@ops-01.internal
    su -
    apt update && apt install -y sudo
    echo 'username ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/username
    chmod 440 /etc/sudoers.d/username
    exit
    exit
    ```

    Then test again:

    ```bash
    ansible ops-01.internal -m ping
    ```

---

## Project Structure

Create this on Control Node. This is where all your Ansible code lives.

```bash
mkdir -p ~/ansible
cd ~/ansible
```

Final structure:

```
~/ansible
├── ansible.cfg
├── inventory
│   ├── group_vars
│   │   ├── all.yml
│   │   └── vault.yml
│   ├── hosts.ini
│   └── host_vars
├── playbooks
│   ├── base.yml
│   ├── stacks.yml
│   └── update.yml
└── roles
    ├── base
    │   ├── files
    │   ├── handlers
    │   │   └── main.yml
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       └── sshd_config.j2
    ├── borgmatic
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       ├── borgmatic-config.yml.j2
    │       └── borgmatic.cron.j2
    ├── codeserver
    │   └── tasks
    │       └── main.yml
    ├── docker
    │   ├── handlers
    │   │   └── main.yml
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       └── daemon.json.j2
    ├── mounts
    │   └── tasks
    │       ├── main.yml
    │       └── templates
    ├── stacks_apps
    │   ├── tasks
    │   └── templates
    ├── stacks_infra
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       ├── cloudflared-compose.yml.j2
    │       ├── homeassistant-compose.yml.j2
    │       ├── homepage-compose.yml.j2
    │       ├── homepage-env.j2
    │       ├── monitoring-compose.yml.j2
    │       ├── npm-compose.yml.j2
    │       └── vaultwarden-compose.yml.j2
    ├── tailscale
    │   └── tasks
    │       └── main.yml
    └── update_containers
        └── tasks
            └── main.yml

```

Create the skeleton:

```bash
cd ~/ansible
mkdir -p inventory/group_vars inventory/host_vars
mkdir -p playbooks
mkdir -p roles/base/{tasks,handlers,templates,files}
mkdir -p roles/docker/{tasks,handlers,templates}
mkdir -p roles/borgmatic/{tasks,templates}
mkdir -p roles/tailscale/tasks
mkdir -p roles/mounts/tasks/templates
mkdir -p roles/stacks_infra/{tasks,templates}
mkdir -p roles/stacks_apps/{tasks,templates}
```
Verify it looks right:

```bash
tree ~/ansible -L 3
```

Ref: [Ansible Docs](https://docs.ansible.com/projects/ansible/latest/user_guide/playbooks_reuse_roles.html#directory-layout-for-roles)