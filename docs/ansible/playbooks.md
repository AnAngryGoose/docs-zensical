## Playbooks

Playbooks are what you actually run. They map roles to host groups.

### playbooks/base.yml

Provisions the base system on all hosts.

```yaml
---
- name: Base system configuration
  hosts: homelab
  become: yes
  roles:
    - base
    - docker
    - tailscale
    - mounts
    - borgmatic
```

### playbooks/stacks.yml

Deploys Docker stacks per host role.

```yaml
---
- name: Deploy infrastructure stacks
  hosts: infra
  become: yes
  roles:
    - stacks_infra

- name: Deploy application stacks
  hosts: apps
  become: yes
  roles:
    - stacks_apps
```

### playbooks/site.yml

The master playbook. Runs everything.

```yaml
---
- import_playbook: base.yml
- import_playbook: stacks.yml
```

---