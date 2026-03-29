## Running Playbooks

This is the part where it all comes together. Run from `~/ansible/` on Control Node.

### First time setup (test connectivity)

```bash
# Ping all hosts to verify Ansible can reach them
ansible homelab -m ping

# Expected output for each host:
# ops-01.internal | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
```

If any host fails, check:
- SSH key is copied to the host
- Host is reachable (`ssh -i ~/.ssh/ansible username@ops-01.internal`)
- DNS resolves (`.internal` hostname works)

### Dry run (check mode)

See what would change without actually changing anything:

```bash
# Dry run the base playbook against all hosts
ansible-playbook playbooks/base.yml --check

# Dry run against a single host
ansible-playbook playbooks/base.yml --check --limit ops-01.internal
```

### Run for real

```bash
# Provision base system on all hosts
ansible-playbook playbooks/base.yml

# Deploy stacks only
ansible-playbook playbooks/stacks.yml

# Run everything (base + stacks)
ansible-playbook playbooks/site.yml

# Run only against ops-01
ansible-playbook playbooks/site.yml --limit ops-01.internal

# Run only against apps group
ansible-playbook playbooks/site.yml --limit apps
```

### Run specific roles with tags

Add tags to your playbooks for selective execution:

```yaml
# In playbooks/base.yml, add tags to each role:
roles:
  - { role: base, tags: ['base'] }
  - { role: docker, tags: ['docker'] }
  - { role: tailscale, tags: ['tailscale'] }
  - { role: mounts, tags: ['mounts'] }
  - { role: borgmatic, tags: ['borgmatic'] }
```

Then run only specific parts:

```bash
# Only run the docker role
ansible-playbook playbooks/base.yml --tags docker

# Run everything except borgmatic
ansible-playbook playbooks/base.yml --skip-tags borgmatic
```

---