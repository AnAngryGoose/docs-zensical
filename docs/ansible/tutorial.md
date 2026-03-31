# Provisioning a New Host with Ansible

A complete walkthrough of bringing a bare machine into a managed homelab using Ansible. This guide uses `dev-01` as the example, but the process applies to any new Debian host.

By the end, you will understand what Ansible does, how it works, how to provision a host from nothing, and how to use Ansible day-to-day for routine management.

Official Ansible documentation: https://docs.ansible.com/ansible/latest/getting_started/index.html

---

## What is Ansible

Ansible is a tool that configures remote machines over SSH. You describe what you want the machine to look like (packages installed, files in place, services running) and Ansible makes it happen. You do not install Ansible on the remote machines. It runs from a single control node and reaches out to each managed host via SSH.

The core idea is **declarative configuration**: you describe the desired state, not a sequence of steps. If a package is already installed, Ansible skips it. If a config file already matches the template, Ansible leaves it alone. This property is called **idempotency**, and it means you can safely run the same playbook over and over. The first run changes things. Subsequent runs confirm nothing drifted.

Ref: https://docs.ansible.com/ansible/latest/getting_started/get_started_ansible.html

---

## Core Concepts

Before touching any files, here is the vocabulary. Every term maps to a real file or action you will encounter in this guide.

**Control node** is the machine that runs Ansible.  Ansible is only installed here.

**Managed node** is a machine Ansible configures. dev-01, ops-01, prod-deb-01, and jupiter are all managed nodes. They do not need Ansible installed. They only need SSH access and sudo.

**Inventory** is a file that lists your managed nodes and organizes them into groups. When you run a playbook, Ansible reads the inventory to know which machines to target.

**Playbook** is a YAML file that says "run these roles against these hosts." It is the entry point for any Ansible run. You execute playbooks from the command line.

**Role** is a reusable bundle of work. The `base` role installs packages and hardens SSH. The `docker` role installs Docker. Each role has its own folder with tasks, templates, handlers, and files.

**Task** is a single action inside a role. "Install curl" is a task. "Copy the SSH config template" is a task. Tasks run in order, top to bottom.

**Handler** is a special task that only runs when triggered. For example, the SSH config template task notifies a handler called `restart sshd`. If the template did not change, the handler never fires. This prevents unnecessary service restarts.

**Template** is a file with placeholder variables (Jinja2 syntax, using `{{ variable_name }}`). Ansible fills in the variables and copies the result to the managed node. Your `sshd_config.j2` is a template.

**Module** is a built-in Ansible function. `ansible.builtin.apt` installs packages. `ansible.builtin.template` copies templates. `ansible.builtin.service` manages services. You do not write these yourself; you call them in your tasks.

**Facts** are information Ansible automatically collects about each managed node when a playbook runs. OS version, IP addresses, CPU count, disk layout, hostname. You can reference facts in templates and tasks. For example, `{{ ansible_distribution_release }}` resolves to `trixie` on Debian 13.

**Vault** is Ansible's built-in encryption for secrets. Passwords and tokens go in a vault-encrypted YAML file so they never sit in plain text on disk.

**Tags** are labels you attach to roles or tasks so you can run subsets of a playbook. `--tags docker` runs only the docker role. `--skip-tags borgmatic` runs everything except borgmatic.

Ref: https://docs.ansible.com/ansible/latest/getting_started/basic_concepts.html

---

## How It All Fits Together

Here is the flow when you run a playbook:

```
You run: ansible-playbook playbooks/base.yml --limit dev-01.internal

Ansible reads ansible.cfg
  -> finds inventory file (inventory/hosts.ini)
  -> finds dev-01.internal in the [standby] group
  -> [standby] is a child of [homelab], which matches hosts: homelab in the playbook
  -> loads variables from group_vars/all.yml (applies to all hosts)
  -> loads variables from group_vars/vault.yml (secrets, loaded via vars_files)
  -> loads variables from host_vars/dev-01.internal.yml (if it exists)
  -> connects to dev-01.internal via SSH using ~/.ssh/ansible key
  -> gathers facts about dev-01 (OS, hostname, IPs, etc.)
  -> runs each role in order: base, docker, tailscale, mounts, borgmatic
  -> within each role, runs tasks top to bottom
  -> if a task changes something, it may notify a handler
  -> handlers fire at the end of all tasks
  -> reports summary: ok=X, changed=Y, failed=Z
```

The `--limit` flag restricts which hosts the playbook targets. Without it, the playbook runs against every host that matches its `hosts:` directive.

Ref: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html

---

## The Existing Setup

Ansible is already installed and configured on your control node. The project lives at `~/ansible/` and currently manages ops-01. Here is the structure:

```
~/ansible/
  ansible.cfg                    # Global settings (SSH key, inventory path, sudo config)
  .vault_pass                    # Vault decryption password (chmod 600, never commit)
  inventory/
    hosts.ini                    # Host inventory with groups
    group_vars/
      all.yml                    # Variables shared across all hosts
      vault.yml                  # Encrypted secrets (samba password, tokens)
    host_vars/                   # Per-host variable overrides (empty for now)
  playbooks/
    base.yml                     # Provisions base system (packages, SSH, Docker, backups)
    stacks.yml                   # Deploys Docker Compose stacks per host role
    update.yml                   # Pulls and restarts containers
  roles/
    base/                        # Packages, SSH hardening, timezone, directories
    docker/                      # Docker CE installation and daemon config
    tailscale/                   # Tailscale install (left stopped)
    mounts/                      # CIFS mount to jupiter NAS
    borgmatic/                   # Borg backup config and cron
    codeserver/                  # Native code-server install (on infra hosts)
    stacks_infra/                # Docker stacks for ops-01 (NPM, monitoring, etc.)
    stacks_apps/                 # Docker stacks for prod-deb-01 (empty, future use)
    update_containers/           # Pull latest images and recreate
```

The key files that control behavior:

`ansible.cfg` tells Ansible where to find the inventory, which SSH key to use, which user to connect as, and that it should use sudo without a password prompt.

`inventory/hosts.ini` lists all managed hosts and their groups. Groups determine which playbooks and variables apply to which machines.

`inventory/group_vars/all.yml` defines shared variables like package lists, directory paths, and DNS settings. Every host gets these.

`inventory/group_vars/vault.yml` holds encrypted secrets (samba password, service tokens). Decrypted automatically at runtime using `.vault_pass`.

---

## Prepare the Network

Before the machine exists on the network, it needs two things in OPNsense: a DHCP static mapping so it always gets the same IP, and a dnsmasq host entry so `dev-01.internal` resolves across the lab.

### DHCP Static Mapping

In the OPNsense web UI:

1. Go to **Services > ISC DHCPv4 > [LAB interface]**
2. Scroll to **DHCP Static Mappings for this Interface**
3. Click **Add**
4. Set the MAC address of dev-01's NIC (find it on the machine's BIOS screen or from a live boot)
5. Set the IP address (pick the next available in 10.10.30.x)
6. Set the hostname to `dev-01`
7. Save and Apply

### dnsmasq Host Entry

1. Go to **Services > Dnsmasq DNS > Host Overrides**
2. Add an entry: Host = `dev-01`, Domain = `internal`, IP = the IP you just assigned
3. Save and Apply

### Why This Matters

Ansible connects to hosts by name. The inventory file says `dev-01.internal`, so DNS has to resolve that to an IP. The static DHCP mapping ensures the IP never changes. Without these two pieces, Ansible cannot reach the machine.

---

## Install Debian 13 (Trixie)

Install Debian 13 Trixie on the NVMe drive. During the installer:

- Set hostname to `dev-01`
- Create user `goose`
- Use the NVMe as the OS disk
- Choose minimal install (no desktop environment, no extra packages)

Since this is a test bed, skip setting up the SATA SSD for now. Ansible will create `/mnt/appdata` as a regular directory on the NVMe. If you later want a dedicated appdata disk, you can format the SSD, add an fstab entry, and re-run the playbook.

Ref: https://www.debian.org/releases/trixie/

---

## Pre-Ansible Setup on dev-01

Ansible needs two things to work: an SSH key that lets it log in without a password, and sudo that does not prompt for a password. Fresh Debian minimal installs do not include `sudo`, so this step is manual.

### Install sudo and configure passwordless sudo

SSH into dev-01 from your control node using whatever method works (password auth is still enabled at this point):

```bash
ssh goose@dev-01.internal
```

If DNS is not propagated yet, use the IP directly:

```bash
ssh goose@10.10.30.XX
```

Once logged in, switch to root (the root password was set during the Debian install):

```bash
su -
```

Install sudo:

```bash
apt update && apt install -y sudo
```

Configure passwordless sudo for goose:

```bash
echo 'goose ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/goose
chmod 440 /etc/sudoers.d/goose
```

Exit back to your normal user, then exit SSH entirely:

```bash
exit
exit
```

### Why passwordless sudo

Ansible runs most tasks as root (installing packages, writing config files to /etc, managing services). It connects as `goose` then uses `sudo` to elevate. If sudo prompted for a password, Ansible would hang waiting for input. The `become_ask_pass = False` setting in ansible.cfg relies on this sudoers config being in place.

Ref: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_privilege_escalation.html

### Copy the Ansible SSH key

Back on your control node, copy the dedicated Ansible key to dev-01:

```bash
ssh-copy-id -i ~/.ssh/ansible.pub goose@dev-01.internal
```

This adds the public key to `~goose/.ssh/authorized_keys` on dev-01. From this point forward, Ansible can log in without a password using the private key at `~/.ssh/ansible`.

Test it:

```bash
ssh -i ~/.ssh/ansible goose@dev-01.internal "hostname"
```

!!! warning
    Ensure you login with your ansible key, and that it is working before running the base playbook. The base playbook will disable password authentication and root login in SSH. If your key is not set up correctly, you will lock yourself out of the machine and have to fix it via a local console.

If that prints `dev-01` without asking for a password, you are good. If it prompts for a password, something went wrong with the key copy.

---

## Add dev-01 to the Ansible Inventory

Now we tell Ansible that dev-01 exists. This happens in one file on your control node.

### Edit inventory/hosts.ini

Open `~/ansible/inventory/hosts.ini` on your control node. It currently looks like this:

```ini
[infra]
ops-01.internal

[apps]
prod-deb-01.internal

[standby]
dev-01.internal

[nas]
jupiter.internal

[homelab:children]
infra
apps
standby
```

dev-01 is already listed in the `[standby]` group. The `[homelab:children]` section includes `standby`, which means dev-01 is part of the `homelab` group. Since `playbooks/base.yml` targets `hosts: homelab`, dev-01 is already included. No changes needed here.

If dev-01 were not already in the inventory, you would add it under the appropriate group. The group name determines which group_vars apply and which playbooks target it.

### How groups and variable loading work

For this repo's inventory layout, think of variable loading like this (later entries override earlier ones):

```
group_vars/all.yml          <- applies to every host, loaded first
group_vars/<group>.yml      <- applies to hosts in that group
host_vars/<hostname>.yml    <- applies to one specific host, loaded last (wins)
```

The full Ansible variable precedence rules are more extensive than this (there are 22 levels). For this repo, the simplified model above covers every case you will encounter.

Ref: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence

For dev-01:
- `group_vars/all.yml` loads because all.yml applies to everything
- `group_vars/standby.yml` would load if it existed (it does not currently)
- `host_vars/dev-01.internal.yml` would load if it existed (it does not currently)

Since dev-01 is a test bed with no special configuration, the defaults in `all.yml` are sufficient. You do not need to create any host_vars or group_vars files for it.

Ref: https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html

### Verify Ansible can reach dev-01

From `~/ansible/` on your control node:

```bash
ansible dev-01.internal -m ping
```

Expected output:

```
dev-01.internal | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

This `ping` is not an ICMP ping. It is an Ansible module that connects via SSH, runs a tiny Python script on the remote host, and returns "pong" if everything works. It validates: SSH connectivity, key authentication, Python availability on the remote host, and sudo access.

!!! info "If it fails" 

    Check:

    - **DNS resolves**: `dig dev-01.internal` from your control node
    - **SSH works manually**: `ssh -i ~/.ssh/ansible goose@dev-01.internal`
    - **`sudo` works**: `ssh -i ~/.ssh/ansible goose@dev-01.internal "sudo whoami"` should print `root`

---

## Dry Run

Before making any changes to dev-01, do a dry run. This shows you exactly what Ansible would do without actually doing it.

```bash
cd ~/ansible
ansible-playbook playbooks/base.yml --check --diff --limit dev-01.internal
```

The `--check` flag is "check mode." Ansible connects to dev-01, evaluates every task, and reports what would change. Nothing is actually modified on the remote host.

The `--diff` flag shows a line-by-line comparison of what changed in templated files. Combined with `--check`, you see exactly what Ansible will modify without actually modifying it.

The `--limit` flag restricts execution to dev-01 only. Without it, the playbook would target every host in the `homelab` group (ops-01, prod-deb-01, and dev-01).



You will see output like:

```
TASK [base : Set timezone] ****************************************************
changed: [dev-01.internal]

TASK [base : Install base packages] *******************************************
changed: [dev-01.internal]

TASK [docker : Install Docker CE] *********************************************
changed: [dev-01.internal]
```

Everything shows `changed` because dev-01 is a fresh install with none of this configured yet. That is expected.

!!!info 
    Some tasks may show `skipped` or `failed` in check mode. 
    
    This is normal. For example, the Docker repo task cannot actually check whether packages are available because it has not really added the repo yet. Check mode is a best-effort preview, not a guarantee.

Ref: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html

---

## Run the Playbook

This is where Ansible actually configures dev-01.

!!! warning

    Ensure you login with your ansible key, and that it is working before running the base playbook. The base playbook will disable password authentication and root login in SSH. If your key is not set up correctly, you will lock yourself out of the machine and have to fix it via a local console.

```bash
ansible-playbook playbooks/base.yml --limit dev-01.internal
```

Ansible will connect to dev-01 and execute each role in the order defined in the playbook. Here is what each role does and why.

### Role: base

**What it does:** Sets the timezone, generates the locale, updates the apt cache, installs all base packages (curl, git, htop, jq, tmux, rsync, etc.), creates the standard directory structure (`/opt/docker`, `/mnt/appdata`, `/mnt/backups`), deploys a hardened SSH config, and ensures passwordless sudo is configured.

**Why:** Every host in the lab needs the same foundation. Instead of manually running `apt install` on each machine and hoping you remember the same list, the role guarantees consistency. The SSH hardening (key-only auth, no root login) is enforced on every Ansible run, so even if someone manually edits `sshd_config`, the next run reverts it.

**How it works internally:** The `ansible.builtin.apt` module checks each package. If already installed, it skips it. If not, it installs it. The `ansible.builtin.template` module compares the rendered template against the file on disk. If they match, no change. If they differ, it writes the new version and notifies the `restart sshd` handler.

### Role: docker

**What it does:** Removes any old Docker packages (distro-provided ones that conflict), adds Docker's official GPG key and apt repository, installs Docker CE, the CLI, containerd, and the compose plugin. Adds `goose` to the `docker` group so you can run docker commands without sudo. Configures log rotation (10MB max, 3 files) via `/etc/docker/daemon.json`.

**Why:** Docker from the official repo is more current than what Debian ships. The daemon.json log config prevents container logs from filling up the disk, which is a common problem on long-running hosts.

**How the `creates` argument works:** The GPG key task uses `args: creates: /etc/apt/keyrings/docker.gpg`. This tells Ansible "if this file already exists, skip the task." It is a simple idempotency guard for shell commands that would otherwise always show as `changed`.

Ref: https://docs.docker.com/engine/install/debian/

### Role: tailscale

**What it does:** Adds the Tailscale apt repository and installs tailscale. Then explicitly stops and disables the service.

**Why:** Tailscale is an emergency backdoor. If OPNsense goes down and you lose access to the VLANs, you can SSH into a host via Tailscale's overlay network. It is installed but not running so it does not consume resources or create unexpected network paths during normal operation. To use it: `sudo tailscale up`.

### Role: mounts

**What it does:** Creates `/etc/samba/` with restricted permissions, writes a credentials file containing the NAS samba username and password (pulled from the vault), and mounts `//jupiter.internal/backups` at `/mnt/backups` via CIFS.

**Why:** Borgmatic needs somewhere to write backup archives. Jupiter's samba share is the target. The credentials file has mode 0600 so the password is not exposed. The `ansible.posix.mount` module both mounts the filesystem and adds it to `/etc/fstab` so it persists across reboots. This module is from the `ansible.posix` collection (not `ansible-core`), included with the full `ansible` package.

Ref: https://docs.ansible.com/ansible/latest/collections/ansible/posix/mount_module.html

### Role: borgmatic

**What it does:** Installs `uv` (a fast Python package manager), then uses `uv tool install` to install borgmatic as recommended by upstream. Creates the config directory, templates the borgmatic config (backup sources, exclude patterns, retention policy), sets up a cron job to run at 03:00 daily, checks if a Borg repo exists on the NAS, and initializes one if not.

**Why:** Automated daily backups of `/opt/docker` (compose files), `/mnt/appdata` (container data), and `/etc` (system config). The retention policy keeps 7 daily, 4 weekly, and 6 monthly archives. The `--encryption=repokey` flag with an empty passphrase means the repo is deduplicated and checksummed - the passphrase is set via `BORG_PASSPHRASE: "{{ borg_passphrase }}"` which is a vault variable. 

**Note on install method:** Borgmatic's upstream documentation now recommends `uv tool install borgmatic` over pip. The `uv` tool isolates borgmatic into its own virtual environment automatically, avoiding conflicts with system Python packages. This replaces the older `pip install --break-system-packages borgmatic` approach.

Ref: https://torsion.org/borgmatic/how-to/install-borgmatic/

---

## Verify

After the playbook completes, verify everything is in place. You can do this with Ansible ad-hoc commands from your control node, which is a good skill to practice.

### Check base packages

```bash
ansible dev-01.internal -a "which htop jq tmux git"
```

The `-a` flag runs a raw command. This is the `command` module (Ansible's default). It runs the command on the remote host and returns the output.

### Check Docker

```bash
ansible dev-01.internal -a "docker --version"
ansible dev-01.internal -a "docker compose version"
```

### Check Docker is running

```bash
ansible dev-01.internal -m service_facts
```

This returns JSON with every service's state. To check just Docker:

```bash
ansible dev-01.internal -a "systemctl is-active docker"
```

### Check the NAS mount

```bash
ansible dev-01.internal -a "df -h /mnt/backups"
ansible dev-01.internal -a "ls /mnt/backups"
```

### Check borgmatic

```bash
ansible dev-01.internal --become -a "/root/.local/bin/borgmatic --version"
ansible dev-01.internal -a "cat /etc/borgmatic/config.yaml"
ansible dev-01.internal -a "cat /etc/cron.d/borgmatic"
```

### Check Tailscale is installed but stopped

```bash
ansible dev-01.internal -a "tailscale version"
ansible dev-01.internal -a "systemctl is-active tailscaled"
```

The first should print a version number. The second should print `inactive`.

### Check SSH hardening

```bash
ansible dev-01.internal -a "grep PasswordAuthentication /etc/ssh/sshd_config"
ansible dev-01.internal -a "grep PermitRootLogin /etc/ssh/sshd_config"
```

Expected: `PasswordAuthentication no` and `PermitRootLogin no`.

### Check directory structure

```bash
ansible dev-01.internal -a "ls -la /opt/docker /mnt/appdata /mnt/backups"
```

### Verify idempotency

Run the playbook again:

```bash
ansible-playbook playbooks/base.yml --limit dev-01.internal
```

This time, the summary should show `changed=0` (or close to it). A few tasks may always show `changed` (like shell commands without `creates`), but the majority should be `ok`. This confirms the playbook is idempotent: running it twice does the same thing as running it once.

---

## Test Deploying a Stack

Since dev-01 is a test bed, let's deploy a simple container to practice the compose workflow. This mimics what `stacks_infra` does on ops-01 but in a manual, one-off way so you can see the mechanics.

### The pattern for Ansible-managed compose stacks

Every stack follows the same four steps:

1. Create the compose directory at `/opt/docker/<stack-name>/`
2. Create the appdata directories at `/mnt/appdata/<service>/`
3. Template the compose file from a `.j2` template to the compose directory
4. Run `docker compose up -d` if the compose file changed

The compose templates use Jinja2 variables like `{{ appdata_dir }}` and `{{ docker_dns }}` so the same template works on any host. Ansible fills in the values from your `group_vars/all.yml`.

### Create a test stack template

On your control node, create a simple compose template. This deploys IT-Tools, a lightweight utility container that is harmless and easy to verify:

Create the file at `~/ansible/roles/stacks_infra/templates/test-compose.yml.j2`:

```yaml
# Managed by Ansible -- test stack for dev-01
services:
  it-tools:
    image: corentinth/it-tools:latest
    container_name: it-tools
    restart: unless-stopped
    ports:
      - "8071:80"
    dns:
      - {{ docker_dns }}
```

### Add tasks to deploy it

You could add this to the existing `stacks_infra` role, but since it is a test, let's run it as an ad-hoc set of commands to understand what each step does:

```bash
# Step 1: Create the compose directory on dev-01
ansible dev-01.internal -m file -a "path=/opt/docker/test state=directory owner=goose group=goose"

# Step 2: Copy the rendered template to dev-01
ansible dev-01.internal -m template -a "src=roles/stacks_infra/templates/test-compose.yml.j2 dest=/opt/docker/test/docker-compose.yml owner=goose group=goose"

# Step 3: Start the stack
ansible dev-01.internal -a "docker compose -f /opt/docker/test/docker-compose.yml up -d"
```

Each of these is an ad-hoc command using a different module:

- `file` creates directories (like `mkdir -p`)
- `template` renders a Jinja2 template and copies it to the remote host
- The default command module runs docker compose

### Verify it is running

```bash
ansible dev-01.internal -a "docker ps"
```

You should see `it-tools` running and mapped to port 8071. You can also visit `http://dev-01.internal:8071` in your browser.

### Clean up when done

```bash
ansible dev-01.internal -a "docker compose -f /opt/docker/test/docker-compose.yml down"
ansible dev-01.internal -m file -a "path=/opt/docker/test state=absent"
```

The `state=absent` on a `file` module deletes the directory.

---

## Day-to-Day Ansible Usage

Now that dev-01 is provisioned, here is how you use Ansible for routine management.

### Checking on your hosts

```bash
# Are all hosts reachable?
ansible homelab -m ping

# What is the uptime on all hosts?
ansible homelab -a "uptime"

# How much disk space is left on all hosts?
ansible homelab -a "df -h /"

# What containers are running on ops-01?
ansible ops-01.internal -a "docker ps"

# Check memory usage on dev-01
ansible dev-01.internal -a "free -h"
```

These are **ad-hoc commands**. They run a single task against one or more hosts without needing a playbook. The `-m` flag specifies a module (defaults to `command` if omitted). The `-a` flag passes arguments.

Ref: https://docs.ansible.com/ansible/latest/command_guide/intro_adhoc.html

### Gathering host information (facts)

Ansible collects a large set of system facts every time a playbook runs. You can query them directly:

```bash
# All facts for a host (long output)
ansible dev-01.internal -m setup

# Filter to specific facts
ansible dev-01.internal -m setup -a "filter=ansible_distribution*"
ansible dev-01.internal -m setup -a "filter=ansible_memtotal_mb"
ansible dev-01.internal -m setup -a "filter=ansible_default_ipv4"
ansible dev-01.internal -m setup -a "filter=ansible_hostname"
```

Useful facts you can reference in templates and tasks:

| Fact | Example Value | Use Case |
|------|---------------|----------|
| `ansible_hostname` | `dev-01` | Setting container hostnames |
| `ansible_distribution_release` | `trixie` | Docker apt repo URL |
| `ansible_default_ipv4.address` | `10.10.30.XX` | Network config templates |
| `ansible_memtotal_mb` | `16014` | Conditional tasks based on RAM |
| `ansible_processor_vcpus` | `6` | Tuning thread counts |

Ref: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html

### Editing a compose file and pushing it

This is the main workflow once services are deployed. The typical cycle:

**1. Edit the template on your control node:**

```bash
cd ~/ansible
micro roles/stacks_infra/templates/npm-compose.yml.j2
```

**2. See what would change (dry run):**

```bash
ansible-playbook playbooks/stacks.yml --check --diff --limit ops-01.internal
```

The `--diff` flag shows a line-by-line comparison of what changed in templated files. Combined with `--check`, you see exactly what Ansible will modify without actually modifying it.

**3. Push the change:**

```bash
ansible-playbook playbooks/stacks.yml --limit ops-01.internal
```

If the compose file changed, the `stacks_infra` role's task for that stack detects it (via `register: npm_compose`) and runs `docker compose up -d` to apply the change. If the file did not change, Docker is not touched.

**4. Verify:**

```bash
ansible ops-01.internal -a "docker ps"
```

### Pulling updated container images

To update containers across hosts, use the update playbook:

```bash
# Update all containers on ops-01
ansible-playbook playbooks/update.yml --limit ops-01.internal

# Update on all hosts
ansible-playbook playbooks/update.yml
```

Or do it ad-hoc for a specific stack:

```bash
# Pull latest images for the monitoring stack
ansible ops-01.internal -a "docker compose -f /opt/docker/monitoring/docker-compose.yml pull"

# Recreate containers with the new images
ansible ops-01.internal -a "docker compose -f /opt/docker/monitoring/docker-compose.yml up -d"
```

### Running a manual backup

```bash
# Trigger borgmatic on a specific host
ansible dev-01.internal -a "borgmatic --verbosity 1 --list --stats" --become

# Trigger on all hosts
ansible homelab -a "borgmatic --verbosity 1 --list --stats" --become
```

The `--become` flag ensures the command runs as root, which borgmatic needs to read `/etc` and other protected directories.

### Re-converging after a drift

If someone manually changes something on a host (edits sshd_config by hand, removes a package, deletes a cron job), re-running the playbook fixes it:

```bash
ansible-playbook playbooks/base.yml --limit dev-01.internal
```

Ansible compares the current state to the declared state and corrects any differences. This is the power of idempotency. You do not need to figure out what changed. Just re-run and Ansible puts everything back.

---

## Understanding the Playbook File

Here is `playbooks/base.yml` with annotations:

```yaml
---
# The triple dash marks the start of a YAML document

- name: Base system configuration        # Human-readable name, shows in output
  hosts: homelab                          # Target group from inventory
  become: yes                             # Use sudo for all tasks
  vars_files:                             # Additional variable files to load
    - ../inventory/group_vars/vault.yml   # Load encrypted secrets
  roles:                                  # Roles to apply, in order
    - { role: base, tags: ['base'] }
    - { role: docker, tags: ['docker'] }
    - { role: tailscale, tags: ['tailscale'] }
    - { role: mounts, tags: ['mounts'] }
    - { role: borgmatic, tags: ['borgmatic'] }
```

The `vars_files` line is included by design. In this repo, `vault.yml` is loaded explicitly so secrets are clearly separated from regular variables. Ansible auto-loads files in `group_vars/` that match a group or host name, but `vault.yml` does not match any group, so it requires explicit loading.

Ref: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html

### Using tags

Tags let you run specific parts of the playbook:

```bash
# Only run the docker role
ansible-playbook playbooks/base.yml --tags docker --limit dev-01.internal

# Run everything except borgmatic
ansible-playbook playbooks/base.yml --skip-tags borgmatic --limit dev-01.internal

# Run base and docker only
ansible-playbook playbooks/base.yml --tags base,docker --limit dev-01.internal
```

This is useful when you are iterating on a single role and do not want to wait for all five roles to run every time.

Ref: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_tags.html

---

## Understanding a Role in Detail

Let's walk through the `base` role task by task to see how roles work internally. This is `roles/base/tasks/main.yml`:

```yaml
- name: Set timezone
  community.general.timezone:
    name: "{{ timezone }}"
```

**Module:** `community.general.timezone`. Sets the system timezone. Like `locale_gen`, this module is from the `community.general` collection, not `ansible-core`. It is included with the full `ansible` package.
**Variable:** `{{ timezone }}` comes from `group_vars/all.yml` where it is set to `America/Chicago`.
**Idempotency:** If the timezone is already `America/Chicago`, this task reports `ok` and changes nothing.

Ref: https://docs.ansible.com/ansible/latest/collections/community/general/timezone_module.html

```yaml
- name: Set locale
  community.general.locale_gen:
    name: "{{ locale }}"
    state: present
```

**Module:** `community.general.locale_gen`. Generates the specified locale so programs can use it. Note: the role file may reference this as `ansible.builtin.locale_gen`, which works because Ansible resolves short module names. The technically correct fully qualified collection name (FQCN) is `community.general.locale_gen` since this module is part of the `community.general` collection, not `ansible-core`.

Ref: https://docs.ansible.com/ansible/latest/collections/community/general/locale_gen_module.html

```yaml
- name: Update apt cache
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600
```

**Module:** `apt`. The `cache_valid_time: 3600` means "only run `apt update` if the cache is older than 1 hour." This prevents hitting the apt mirrors on every single run.

```yaml
- name: Install base packages
  ansible.builtin.apt:
    name: "{{ base_packages }}"
    state: present
```

**Variable:** `{{ base_packages }}` is a list defined in `group_vars/all.yml`. The apt module accepts a list and installs everything in one transaction. `state: present` means "make sure it is installed." It does not upgrade if already present. Use `state: latest` if you want upgrades (but that makes runs less predictable).

```yaml
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
```

**The `loop` keyword:** This runs the task once for each item in the list. `{{ item }}` refers to the current item. So this creates three directories: `/opt/docker`, `/mnt/appdata`, and `/mnt/backups`.

```yaml
- name: Configure SSH daemon
  ansible.builtin.template:
    src: sshd_config.j2
    dest: /etc/ssh/sshd_config
    owner: root
    group: root
    mode: "0644"
    validate: "sshd -t -f %s"
  notify: restart sshd
```

**Module:** `template`. Renders `roles/base/templates/sshd_config.j2` with variables filled in and places it at `/etc/ssh/sshd_config`.
**`validate`:** Before writing the file, Ansible runs `sshd -t -f <tempfile>` to check the config syntax. If validation fails, the file is not written and the task fails. This prevents locking yourself out with a bad SSH config.
**`notify`:** If the file changed, the `restart sshd` handler is triggered. If the file was already identical, no restart happens.

```yaml
- name: Ensure passwordless sudo for main user
  ansible.builtin.copy:
    content: "{{ main_user }} ALL=(ALL) NOPASSWD:ALL\n"
    dest: "/etc/sudoers.d/{{ main_user }}"
    owner: root
    group: root
    mode: "0440"
    validate: "visudo -cf %s"
```

**Module:** `copy` with inline `content` instead of a source file. Writes a single line to `/etc/sudoers.d/goose`. The `validate` runs `visudo -cf` to check sudoers syntax before committing.

---

## Working with Ansible Vault

The vault stores secrets like the NAS samba password and service tokens. Here are the commands you will use:

```bash
cd ~/ansible

# View current secrets (read-only)
ansible-vault view inventory/group_vars/vault.yml

# Edit secrets (opens in your $EDITOR)
ansible-vault edit inventory/group_vars/vault.yml

# Change the vault encryption password
ansible-vault rekey inventory/group_vars/vault.yml
```

To add a new secret, edit the vault file and add a new key:

```yaml
---
vault_nas_samba_password: "existing-password"
vault_some_new_secret: "new-secret-value"
```

Then reference it in your roles or templates as `{{ vault_some_new_secret }}`.

The vault password file at `~/ansible/.vault_pass` should be `chmod 600` and never committed to version control.

Ref: https://docs.ansible.com/ansible/latest/vault_guide/index.html

---

## Useful Flags and Options

A reference for common command-line flags.

| Flag | Purpose | Example |
|------|---------|---------|
| `--limit` | Restrict to specific host(s) or group(s) | `--limit dev-01.internal` |
| `--check` | Dry run, show what would change | `--check` |
| `--diff` | Show file-level diffs for template changes | `--diff` |
| `--tags` | Run only tasks/roles with these tags | `--tags docker,base` |
| `--skip-tags` | Run everything except these tags | `--skip-tags borgmatic` |
| `-v` | Verbose output (add more v's for more detail) | `-vvv` for SSH debug |
| `--list-hosts` | Show which hosts a playbook would target | `--list-hosts` |
| `--list-tasks` | Show which tasks would run | `--list-tasks` |
| `--become` | Use sudo (already default in our cfg, but needed for ad-hoc) | `--become` |

Combining flags:

```bash
# Dry run with diffs against one host, verbose
ansible-playbook playbooks/base.yml --check --diff --limit dev-01.internal -v
```

Ref: https://docs.ansible.com/ansible/latest/cli/ansible-playbook.html

---

## Troubleshooting

### "Permission denied" on SSH

Verify the key works manually:

```bash
ssh -i ~/.ssh/ansible -v goose@dev-01.internal
```

The `-v` flag shows the SSH handshake. Look for which keys are being offered and whether the server accepts them. If the key is not being offered, check that `private_key_file` in ansible.cfg points to the right path.

### "Missing sudo password"

Run on the managed node:

```bash
echo "goose ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/goose
sudo chmod 440 /etc/sudoers.d/goose
```

### "Could not resolve host"

Check DNS from your control node:

```bash
dig dev-01.internal
```

If it does not resolve, the dnsmasq entry in OPNsense is missing or has not propagated. You can also temporarily use the IP directly by adding to your inventory:

```ini
[standby]
dev-01.internal ansible_host=10.10.30.XX
```

### A task shows "changed" on every run

Tasks using `ansible.builtin.command` or `ansible.builtin.shell` always report `changed` because Ansible has no way to know if the command actually modified anything. Fix this by adding `creates:`, `when:`, or `changed_when: false` conditions:

```yaml
# Before (always changed)
- name: Get release name
  ansible.builtin.command: lsb_release -cs
  register: debian_release

# After (never shows changed, because it is read-only)
- name: Get release name
  ansible.builtin.command: lsb_release -cs
  register: debian_release
  changed_when: false
```

### Template rendering errors

Check that the variable exists:

```bash
ansible dev-01.internal -m debug -a "var=hostvars[inventory_hostname]"
```

This dumps every variable available for that host. Search for the one your template references.

### View verbose output

```bash
ansible-playbook playbooks/base.yml -v      # task results
ansible-playbook playbooks/base.yml -vv     # task input parameters
ansible-playbook playbooks/base.yml -vvv    # SSH connection debug
```

---

## Quick Reference

### Playbook commands

```bash
# Provision base system on one host
ansible-playbook playbooks/base.yml --limit dev-01.internal

# Provision all hosts
ansible-playbook playbooks/base.yml

# Deploy stacks on ops-01
ansible-playbook playbooks/stacks.yml --limit ops-01.internal

# Run everything (base + stacks)
ansible-playbook playbooks/site.yml

# Dry run with diffs
ansible-playbook playbooks/base.yml --check --diff --limit dev-01.internal

# Run only docker role
ansible-playbook playbooks/base.yml --tags docker --limit dev-01.internal
```

### Ad-hoc commands

```bash
# Ping all hosts
ansible homelab -m ping

# Run a command on all hosts
ansible homelab -a "uptime"

# Run a command on one host
ansible dev-01.internal -a "docker ps"

# Check disk space
ansible homelab -a "df -h /"

# Restart a service
ansible dev-01.internal -m service -a "name=docker state=restarted" --become

# Copy a file
ansible dev-01.internal -m copy -a "src=/tmp/test.txt dest=/tmp/test.txt"

# Install a package
ansible dev-01.internal -m apt -a "name=tree state=present" --become

# Gather facts
ansible dev-01.internal -m setup -a "filter=ansible_distribution*"
```

### Vault commands

```bash
ansible-vault view inventory/group_vars/vault.yml
ansible-vault edit inventory/group_vars/vault.yml
ansible-vault rekey inventory/group_vars/vault.yml
```

---

## Gotchas and Lessons Learned

**Fresh Debian minimal does not include sudo.** You must install it manually before Ansible can connect. This is the only manual step on a new host.

**Jinja2 braces in heredocs.** Never use `cat << EOF` in shell scripts to write files containing `{{ }}` syntax. The shell interprets the braces and breaks everything. Use a text editor (micro, nano, vim) instead, or use Ansible's `template` or `copy` module.

**Vault file naming.** In this repo, `group_vars/vault.yml` is loaded explicitly via `vars_files` in the playbook. This is by design. Ansible auto-loads files in `group_vars/` that are named after a group in the inventory (e.g., `group_vars/homelab.yml` would auto-load for hosts in the `homelab` group). Since `vault.yml` does not match any group name, it must be loaded explicitly.

Ref: https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html

**Docker apt repo on new Debian releases.** The Docker role uses `{{ ansible_distribution_release }}` to build the repo URL. Docker officially supports Debian 13 Trixie, so this works today. However, when future Debian releases come out (e.g. Forky), Docker may not have packages ready on day one. In that case, you would temporarily hardcode the previous release name in the template until Docker catches up.

Ref: https://docs.docker.com/engine/install/debian/

**The command module always shows changed.** Read-only commands like `lsb_release -cs` still report `changed`. Use `changed_when: false` to suppress this.

**Ansible connects via the key in ansible.cfg.** If you rotate SSH keys, update `private_key_file` in ansible.cfg and re-copy the new public key to all hosts.

**The `--limit` flag is your safety net.** When testing changes, always use `--limit` to target a single host. Running against the whole lab when you meant to test on dev-01 can cause unintended changes.


## Checklist: dev-01 from Nothing to Complete

For quick reference, here is the entire process as a condensed checklist:

- [ ] Create DHCP static mapping in OPNsense for dev-01
- [ ] Create dnsmasq host entry for dev-01.internal
- [ ] Install Debian 13 Trixie on dev-01 (hostname: dev-01, user: goose)
- [ ] SSH in, install sudo, configure passwordless sudo
- [ ] Copy Ansible SSH key from your control node: `ssh-copy-id -i ~/.ssh/ansible.pub goose@dev-01.internal`
- [ ] Verify dev-01.internal is already in `inventory/hosts.ini` under [standby]
- [ ] Test connectivity: `ansible dev-01.internal -m ping`
- [ ] Dry run: `ansible-playbook playbooks/base.yml --check --limit dev-01.internal`
- [ ] Run for real: `ansible-playbook playbooks/base.yml --limit dev-01.internal`
- [ ] Verify: Docker running, NAS mounted, borgmatic configured, SSH hardened
- [ ] Verify idempotency: re-run playbook, expect changed=0
- [ ] (Optional) Deploy a test stack to practice the compose workflow
- [ ] (Optional) Clean up the test stack when done