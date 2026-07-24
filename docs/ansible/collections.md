---
icon: lucide/library
title: Collections & Galaxy
description: Where modules come from — namespaces, installing collections, and requirements.yml.
---

# Collections & Galaxy

Ever wondered why the roles in this repo write `community.general.timezone` and
`ansible.posix.mount` instead of just `timezone` and `mount`? That prefix is the
**collection** the module lives in. This page explains the namespace and how to
install the collections your playbooks depend on.

## FQCN — Fully Qualified Collection Name

Every module has a three-part name: `namespace.collection.module`.

| FQCN | Ships in |
|---|---|
| `ansible.builtin.apt` | **Built in** — always available |
| `ansible.builtin.copy` | Built in |
| `ansible.posix.mount` | The `ansible.posix` collection |
| `community.general.timezone` | The `community.general` collection |
| `community.docker.docker_container` | The `community.docker` collection |

!!! tip "Always use the full name"

    Short names (`apt`) still work for built-ins, but the FQCN (`ansible.builtin.apt`)
    is unambiguous, future-proof, and required for anything outside `ansible.builtin`.
    All the roles here already use it.

## What's already installed

The full `ansible` package (how it's installed in the
[Overview](ansible.md#installation)) bundles a large set of common collections —
including `community.general` and `ansible.posix` — so the roles in this repo work
out of the box. Check what you have:

```bash
# List installed collections and versions
ansible-galaxy collection list

# Is a specific module available?
ansible-doc community.general.timezone
```

If you'd installed the minimal `ansible-core` package instead, you'd get **only**
`ansible.builtin` and would need to add the rest yourself.

## Installing a collection

```bash
# Install the latest version
ansible-galaxy collection install community.docker

# Pin a version
ansible-galaxy collection install community.general:>=8.0.0

# Upgrade
ansible-galaxy collection install community.general --upgrade
```

Collections install to `~/.ansible/collections/` by default.

## requirements.yml — pin your dependencies

So a fresh control node can reproduce your setup, list dependencies in a
`requirements.yml` and install them all at once. This is the recommended pattern.

```yaml
# ~/ansible/requirements.yml
collections:
  - name: community.general
    version: ">=8.0.0"
  - name: ansible.posix
  - name: community.docker

# You can pull roles from Galaxy too:
roles:
  - name: geerlingguy.docker
```

```bash
# Install everything the project needs
ansible-galaxy install -r requirements.yml

# Collections only
ansible-galaxy collection install -r requirements.yml
```

!!! tip "Commit `requirements.yml`, not the collections"

    Keep `requirements.yml` in your Ansible git repo and run
    `ansible-galaxy install -r requirements.yml` on any new control node — same
    idea as `requirements.txt` for Python.

## Galaxy roles vs your own roles

- **Your roles** (`roles/base`, `roles/docker`, …) — written and maintained here.
- **Galaxy roles** — community-published roles you can pull in
  (e.g. `geerlingguy.docker`) instead of writing your own.

```bash
# Install a community role
ansible-galaxy role install geerlingguy.docker

# Use it in a playbook
# roles:
#   - geerlingguy.docker
```

For a homelab where you want to understand every task, hand-written roles (as in
this repo) are often clearer than large community roles — but Galaxy is there when
you don't want to reinvent something complex.

Ref: [Ansible — Using collections](https://docs.ansible.com/projects/ansible/latest/collections_guide/index.html)
 · [Ansible Galaxy](https://galaxy.ansible.com/)
