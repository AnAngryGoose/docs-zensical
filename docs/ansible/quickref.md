# Ansible Quick Reference

All commands run from `~/ansible` on the control node.

---

## Deploying and Updating

```bash
# Deploy everything to all hosts
ansible-playbook playbooks/site.yml

# Deploy base config only (packages, SSH, Docker, borgmatic, etc.)
ansible-playbook playbooks/base.yml

# Deploy stacks only (compose services)
ansible-playbook playbooks/stacks.yml

# Target a single host - --limit filters which hosts
ansible-playbook playbooks/base.yml --limit ops-01.internal
ansible-playbook playbooks/stacks.yml --limit ops-01.internal

# Target a group - groups are defined in hosts.ini
ansible-playbook playbooks/base.yml --limit infra
ansible-playbook playbooks/stacks.yml --limit apps

# Run a specific role only
ansible-playbook playbooks/base.yml --tags docker --limit ops-01.internal
ansible-playbook playbooks/stacks.yml --tags monitoring --limit ops-01.internal

# Run everything except one role  - --skip-tags excludes a tag
ansible-playbook playbooks/base.yml --skip-tags borgmatic

# Dry run (preview changes without applying) - --check simulates, changes nothing
ansible-playbook playbooks/base.yml --check --limit ops-01.internal

# Dry run with file diffs shown
ansible-playbook playbooks/base.yml --check --diff --limit ops-01.internal

# Verbose output                          -v, -vv, -vvv for increasing detail
ansible-playbook playbooks/base.yml -v --limit ops-01.internal


```

---

## Ad-Hoc Commands

Run a single command without a playbook. Format: `ansible <target> -m <module> -a "<args>"`

```bash
# Ping (test connectivity + sudo)         -m ping uses the ping module
ansible homelab -m ping

# Run a shell command                     -a uses the command module by default
ansible ops-01.internal -a "docker ps"

# Run against all hosts
ansible homelab -a "uptime"

# Use a specific module                   -m service uses the service module
ansible ops-01.internal -m service -a "name=docker state=restarted"

# Copy a file                             -m copy uses the copy module
ansible homelab -m copy -a "src=/tmp/file.txt dest=/tmp/file.txt"

# Check a variable value                  -m debug prints variables
ansible ops-01.internal -m debug -a "var=ansible_distribution_release"
```

!!! note
     `-a` (command module) always reports CHANGED even for read-only commands. This is normal.

---

## Vault (Secrets)

```bash
# Create encrypted file                   opens editor, encrypts on save
ansible-vault create inventory/group_vars/vault.yml

# Edit existing vault                     decrypts, opens editor, re-encrypts
ansible-vault edit inventory/group_vars/vault.yml

# View without editing                    decrypts to stdout
ansible-vault view inventory/group_vars/vault.yml

# Change vault password
ansible-vault rekey inventory/group_vars/vault.yml
```

Vault files need `vars_files` in the playbook to load:

```yaml
vars_files:
  - ../inventory/group_vars/vault.yml
```

---

## Updating Docker Containers

```bash
# Pull latest images and recreate all containers on all hosts
ansible-playbook playbooks/update.yml

# Update only ops-01
ansible-playbook playbooks/update.yml --limit ops-01.internal

# Update only nas
ansible-playbook playbooks/update.yml --limit nas.internal

# Manual update of a single stack on a host
ansible ops-01.internal -a "docker compose -f /opt/docker/monitoring/docker-compose.yml pull"
ansible ops-01.internal -a "docker compose -f /opt/docker/monitoring/docker-compose.yml up -d"
```

---

## Docker Status

```bash
# All containers on a host
ansible ops-01.internal -a "docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'"

# Running containers only
ansible ops-01.internal -a "docker ps --format 'table {{.Names}}\t{{.Status}}'"

# Check a specific container's logs (last 50 lines)
ansible ops-01.internal -a "docker logs --tail 50 homeassistant"

# Follow logs live (run directly, not via Ansible)
ssh username@ops-01.internal "docker logs -f homeassistant"

# Restart a specific container
ansible ops-01.internal -a "docker restart homeassistant"

# Stop a specific container
ansible ops-01.internal -a "docker stop homeassistant"

# Start a specific container
ansible ops-01.internal -a "docker start homeassistant"

# Check all containers across all hosts
ansible homelab -a "docker ps --format 'table {{.Names}}\t{{.Status}}'"

# Docker networks
ansible ops-01.internal -a "docker network ls"

# Docker volumes (check for orphans)
ansible ops-01.internal -a "docker volume ls"

# Prune unused images
ansible ops-01.internal -a "docker image prune -f"

# Prune everything (images, containers, networks, volumes)
# CAREFUL: removes all stopped containers and unused volumes
ansible ops-01.internal -a "docker system prune --volumes -f"
```

---

## Editing a Stack

Workflow: edit template on Control Node, push with Ansible.

```bash
# 1. Edit the compose template
micro ~/ansible/roles/stacks_infra/templates/monitoring-compose.yml.j2

# 2. Push the change
ansible-playbook playbooks/stacks.yml --limit ops-01.internal

# Never edit compose files directly on the hosts.
# Ansible overwrites them on next run.
```

## Checking Host Status

```bash
# Ping all hosts (tests SSH + sudo)
ansible homelab -m ping

# Uptime
ansible homelab -a "uptime"

# OS and kernel version
ansible homelab -a "uname -r"
ansible homelab -a "cat /etc/os-release"

# Who is logged in
ansible homelab -a "who"

# System load
ansible homelab -a "cat /proc/loadavg"

# Memory usage
ansible homelab -a "free -h"
```

## Checking Disk

```bash
# Disk usage on all hosts
ansible homelab -a "df -h"

# Appdata disk specifically
ansible homelab -a "df -h /mnt/appdata"

# NAS backup mount
ansible homelab -a "df -h /mnt/backups"

# Largest directories in appdata
ansible ops-01.internal -a "du -sh /mnt/appdata/* --max-depth=0"

# Check NAS backup repos
ansible ops-01.internal -a "ls -la /mnt/backups/"

# Docker disk usage
ansible ops-01.internal -a "docker system df"
```

---

## Backups

```bash
# Run borgmatic backup manually on a host
ansible ops-01.internal -a "borgmatic --verbosity 1 --list --stats"

# Check latest backup archives
ansible ops-01.internal -a "borgmatic rlist --last 5"

# Check backup repo info
ansible ops-01.internal -a "borgmatic info"

# Verify backup integrity
ansible ops-01.internal -a "borgmatic check"

# Check backup cron is in place
ansible ops-01.internal -a "cat /etc/cron.d/borgmatic"

# See what's in the backup repo on the NAS
ansible ops-01.internal -a "ls -la /mnt/backups/ops-01/"
```

## Service Management

```bash
# Check Docker daemon status
ansible ops-01.internal -m service -a "name=docker"

# Restart Docker daemon
ansible ops-01.internal -m service -a "name=docker state=restarted"

# Check if Tailscale is running
ansible ops-01.internal -m service -a "name=tailscaled"

# Start Tailscale if needed
ansible ops-01.internal -m service -a "name=tailscaled state=started"

# Check SSH daemon
ansible ops-01.internal -m service -a "name=sshd"

# Check code-server
ansible ops-01.internal -m service -a "name=code-server@username"
```

## Network

```bash
# Check IP addresses
ansible ops-01.internal -a "ip -brief addr"

# Check listening ports
ansible ops-01.internal -a "ss -tlnp"

# Test DNS resolution
ansible ops-01.internal -a "dig prod-deb-01.internal +short"

# Check NAS mount is alive
ansible ops-01.internal -a "mountpoint /mnt/backups"

# Test connectivity to another host
ansible ops-01.internal -a "ping -c 2 nas.internal"
```

---


## Vault (Secrets)

```bash
# Create encrypted file                   opens editor, encrypts on save
ansible-vault create inventory/group_vars/vault.yml

# Edit existing vault                     decrypts, opens editor, re-encrypts
ansible-vault edit inventory/group_vars/vault.yml

# View without editing                    decrypts to stdout
ansible-vault view inventory/group_vars/vault.yml

# Change vault password
ansible-vault rekey inventory/group_vars/vault.yml
```

Vault files need `vars_files` in the playbook to load:

```yaml
vars_files:
  - ../inventory/group_vars/vault.yml
```

---

## Troubleshooting

```bash
# Verbose playbook output
ansible-playbook playbooks/base.yml -v --limit ops-01.internal    # some detail
ansible-playbook playbooks/base.yml -vvv --limit ops-01.internal  # SSH debug

# List what hosts a playbook would target
ansible-playbook playbooks/stacks.yml --list-hosts

# List what tasks would run
ansible-playbook playbooks/stacks.yml --list-tasks

# Show all Ansible facts for a host (OS, IP, RAM, disks, etc.)
ansible ops-01.internal -m setup

# Show specific facts
ansible ops-01.internal -m setup -a "filter=ansible_distribution*"
ansible ops-01.internal -m setup -a "filter=ansible_memtotal_mb"

# Check a variable value
ansible ops-01.internal -m debug -a "var=docker_compose_dir"

# Test SSH manually
ssh -i ~/.ssh/ansible username@ops-01.internal "hostname"

# Check sudoers
ansible ops-01.internal -a "cat /etc/sudoers.d/username"

# Failed systemd units
ansible ops-01.internal -a "systemctl --failed"

# Journal for a service
ansible ops-01.internal -a "journalctl -u docker --no-pager -n 30"
```

---


## Key Concepts

**Idempotent** -- Running twice gives the same result. Second run = `changed=0`. If a task always shows changed, add `creates:` or `when:` to fix it.

**Roles** -- Reusable bundles. Each role has `tasks/main.yml` (what to do), `templates/` (config files with variables), `handlers/main.yml` (actions triggered by changes).

**Handlers** -- Only run when notified. If a template changes, the handler fires (e.g., restart sshd). If nothing changes, the handler is skipped.

**Tags** -- Labels on roles. Defined in the playbook: `{ role: docker, tags: ['docker'] }`. Filter with `--tags` or `--skip-tags`.

**Templates** -- Files ending in `.j2`. Ansible replaces `{{ variable }}` with values from group_vars/host_vars. Loops use `{% for item in list %}`.

**Register** -- Captures task output into a variable for later use:

```yaml
- name: Get something
  command: hostname
  register: result          # stores output in 'result'
- name: Use it
  debug:
    msg: "{{ result.stdout }}"  # access with .stdout
```

---

## Quick Health Check (All Hosts)

Run these in sequence for a full overview:

```bash
ansible homelab -m ping
ansible homelab -a "uptime"
ansible homelab -a "free -h"
ansible homelab -a "df -h /mnt/appdata"
ansible homelab -a "docker ps --format '{{.Names}}: {{.Status}}'" 
ansible homelab -a "mountpoint /mnt/backups"
ansible homelab -a "systemctl --failed"
```

---

## Inventory Targeting

```ini
# hosts.ini
[infra]           # group name
ops-01.internal   # host in group

[apps]
prod-deb-01.internal

[homelab:children] # parent group containing other groups
infra
apps
```

```bash
ansible infra -m ping              # all hosts in infra group
ansible homelab -m ping            # all hosts in homelab (infra + apps)
ansible ops-01.internal -m ping    # single host
ansible all -m ping                # everything in inventory
```

---

## Debugging

```bash
# List hosts a playbook would target
ansible-playbook playbooks/base.yml --list-hosts

# List tasks a playbook would run
ansible-playbook playbooks/base.yml --list-tasks

# Show all facts about a host (OS, IP, RAM, etc.)
ansible ops-01.internal -m setup

# Show a specific fact
ansible ops-01.internal -m setup -a "filter=ansible_distribution*"

# Test a playbook without running it
ansible-playbook playbooks/base.yml --check --diff --limit ops-01.internal
#                                    ^dry run  ^show file diffs
```