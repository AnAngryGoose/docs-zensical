---
icon: lucide/search-code
title: Facts & Variable Precedence
description: System facts, magic variables, set_fact, and which variable wins when names collide.
---

# Facts & Variable Precedence

[Variables](variables.md) covered *where you define* values. This page covers the
variables Ansible provides **automatically** (facts and magic variables) and the
rules for **which value wins** when the same name is defined in more than one place.

## Facts — what Ansible discovers about a host

Before running tasks, Ansible connects to each host and gathers **facts**: OS,
IP addresses, CPU, memory, disks, and more. These are available as `ansible_*`
variables in tasks and templates.

```bash
# See every fact for a host
ansible ops-01.internal -m setup

# Filter to a subset
ansible ops-01.internal -m setup -a "filter=ansible_distribution*"
ansible ops-01.internal -m setup -a "filter=ansible_memtotal_mb"
```

Facts you'll reach for often:

| Fact | Example value |
|---|---|
| `ansible_distribution` | `Debian` |
| `ansible_distribution_major_version` | `12` |
| `ansible_distribution_release` | `bookworm` |
| `ansible_os_family` | `Debian` |
| `ansible_default_ipv4.address` | `10.10.30.20` |
| `ansible_hostname` | `ops-01` |
| `ansible_processor_vcpus` | `4` |
| `ansible_memtotal_mb` | `7900` |

They're used throughout this repo — e.g. the Docker role builds the apt repo line
with `{{ ansible_distribution_release }}` so the same task works on Debian 12 or 13.

!!! tip "Turn off fact-gathering to speed up simple plays"

    Gathering facts adds a round-trip per host. If a play doesn't need any
    `ansible_*` values, skip it:

    ```yaml
    - hosts: homelab
      gather_facts: false
    ```

## Magic variables — always available

These describe the run itself, not the machine:

| Variable | Meaning |
|---|---|
| `inventory_hostname` | The host's name as written in the inventory (`ops-01.internal`) |
| `inventory_hostname_short` | Just the first label (`ops-01`) — used for borg repo paths here |
| `group_names` | List of groups the current host belongs to |
| `groups` | Dict of every group → its hosts (`groups['apps']`) |
| `hostvars` | Access another host's variables: `hostvars['nas.internal'].borg_repo_path` |
| `ansible_play_hosts` | Hosts still active in the current play |

```yaml
- name: Show which groups this host is in
  ansible.builtin.debug:
    msg: "{{ inventory_hostname }} is in {{ group_names }}"

- name: Reference every app host
  ansible.builtin.debug:
    msg: "app servers: {{ groups['apps'] }}"
```

## Setting variables at runtime — `set_fact`

Create or compute a variable mid-play:

```yaml
- name: Compute half the RAM for a container limit
  ansible.builtin.set_fact:
    container_mem_mb: "{{ (ansible_memtotal_mb | int * 0.5) | int }}"

- name: Use it later
  ansible.builtin.debug:
    msg: "Limiting to {{ container_mem_mb }} MB"
```

`register` (from [Writing Tasks](writing-tasks.md#capturing-output-register)) is
the other runtime source — it captures a task's result.

## Variable precedence (which value wins)

When the same variable name is defined in multiple places, Ansible picks by a
fixed priority. From **lowest** to **highest** (higher wins), the parts you'll
actually touch:

1. Role defaults (`roles/x/defaults/main.yml`) — the weakest, meant to be overridden
2. `group_vars/all`
3. `group_vars/<specific group>`
4. `host_vars/<host>`
5. Facts / `set_fact` / registered vars
6. `-e` / `--extra-vars` on the command line — **always wins**

```mermaid
graph LR
  A[role defaults] --> B[group_vars/all]
  B --> C[group_vars/group]
  C --> D[host_vars/host]
  D --> E[set_fact / register]
  E --> F["--extra-vars (-e)"]
```

Practical takeaways:

- Put **safe fallbacks** in `roles/<role>/defaults/main.yml` — anything can override them.
- Put **shared settings** in `group_vars/all.yml`.
- **Host-specific** overrides go in `host_vars/<host>.yml` — they beat group vars
  (that's why `host_vars/nas.internal.yml` can override `docker_compose_dir`).
- Force a one-off value with `-e`:

  ```bash
  ansible-playbook playbooks/base.yml -e "timezone=UTC" --limit ops-01.internal
  ```

!!! warning "The full precedence list is longer"

    Ansible actually defines ~22 precedence levels. The six above cover everyday
    homelab use; see the reference if you hit a surprising override.

Ref: [Ansible — Variable precedence](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)
