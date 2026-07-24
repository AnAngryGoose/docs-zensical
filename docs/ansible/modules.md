---
icon: lucide/blocks
title: Common Modules
description: A reference of the Ansible modules used most often, with examples.
---

# Common Modules

A **module** is the code that actually does the work in a [task](writing-tasks.md).
There are thousands; you'll use maybe fifteen day to day. This is that shortlist,
with copy-paste examples. Names use the **fully qualified** form
(`ansible.builtin.apt`) — see [Collections](collections.md) for why.

!!! tip "Read the docs for any module"

    `ansible-doc ansible.builtin.copy` prints every option and examples right in
    your terminal. `ansible-doc -l` lists all installed modules.

## Packages

=== "apt (Debian/Ubuntu)"

    ```yaml
    - name: Install one package
      ansible.builtin.apt:
        name: htop
        state: present          # present | latest | absent

    - name: Install a list (single transaction — faster than a loop)
      ansible.builtin.apt:
        name: "{{ base_packages }}"
        state: present

    - name: Update the cache, but only if it's older than an hour
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Full upgrade
      ansible.builtin.apt:
        upgrade: dist
    ```

=== "package (distro-agnostic)"

    ```yaml
    - name: Install regardless of distro (uses apt/dnf/etc automatically)
      ansible.builtin.package:
        name: git
        state: present
    ```

=== "pip (Python)"

    ```yaml
    - name: Install a Python package
      ansible.builtin.pip:
        name: borgmatic
        extra_args: --break-system-packages   # needed on modern Debian (PEP 668)
    ```

## Files & directories

| Task | Module |
|---|---|
| Create/remove dirs, set perms, symlinks | `ansible.builtin.file` |
| Copy a file from the control node | `ansible.builtin.copy` |
| Render a Jinja2 template | `ansible.builtin.template` |
| Edit one line in an existing file | `ansible.builtin.lineinfile` |
| Download a file from a URL | `ansible.builtin.get_url` |
| Extract a tarball/zip | `ansible.builtin.unarchive` |

```yaml
- name: Ensure a directory exists with the right owner
  ansible.builtin.file:
    path: /opt/docker/app
    state: directory          # directory | file | touch | link | absent
    owner: "{{ main_user }}"
    group: "{{ main_user }}"
    mode: "0755"

- name: Copy a static file
  ansible.builtin.copy:
    src: motd.txt             # from the role's files/ dir
    dest: /etc/motd
    mode: "0644"

- name: Render a template (variables get filled in)
  ansible.builtin.template:
    src: config.yml.j2        # from the role's templates/ dir
    dest: /etc/app/config.yml
    validate: "app --test-config %s"   # optional: validate before saving

- name: Ensure a single line is present in a file
  ansible.builtin.lineinfile:
    path: /etc/sysctl.conf
    line: "net.ipv4.ip_forward = 1"
    state: present

- name: Download a file only if it isn't already there
  ansible.builtin.get_url:
    url: https://example.com/tool.tar.gz
    dest: /tmp/tool.tar.gz

- name: Extract an archive to a remote path
  ansible.builtin.unarchive:
    src: /tmp/tool.tar.gz
    dest: /opt/tool
    remote_src: true          # the archive is already on the managed node
```

!!! note "`copy` vs `template`"

    `copy` places a file **verbatim**. `template` runs it through
    [Jinja2](templates-jinja2.md) first, filling in `{{ variables }}`. If the file
    has variables, use `template` and name it `*.j2`.

## Services

```yaml
- name: Start now and enable at boot
  ansible.builtin.service:
    name: docker
    state: started            # started | stopped | restarted | reloaded
    enabled: true

- name: systemd-specific (daemon-reload after a unit change)
  ansible.builtin.systemd_service:
    name: myapp
    state: restarted
    daemon_reload: true
```

## Users, groups & cron

```yaml
- name: Create a user
  ansible.builtin.user:
    name: appuser
    groups: docker
    append: true              # ADD to groups (without this, it REPLACES them)
    shell: /bin/bash

- name: Create a group
  ansible.builtin.group:
    name: appusers

- name: Add a cron job
  ansible.builtin.cron:
    name: "nightly backup"    # unique identifier — edits in place on re-run
    minute: "0"
    hour: "3"
    job: "/usr/local/bin/backup.sh"
```

## Commands (last resort)

Use these only when no module fits — they aren't idempotent on their own.

```yaml
- name: Run a command (no shell — no pipes/redirects/globs)
  ansible.builtin.command:
    cmd: docker compose up -d
    chdir: /opt/docker/app    # cd here first
    creates: /opt/app/.installed   # SKIP if this path already exists (idempotency)

- name: Run through a shell (needed for | > < && $VAR)
  ansible.builtin.shell: |
    curl -fsSL https://example.com/key | gpg --dearmor -o /etc/apt/keyrings/x.gpg
  args:
    creates: /etc/apt/keyrings/x.gpg
```

!!! warning "`command` vs `shell`"

    `command` does **not** run in a shell, so `|`, `>`, `&&`, `*`, and `$VAR`
    won't work — it's safer. Use `shell` only when you need those. Add `creates:`
    (or a `when:`) so they don't report `changed` every run. See
    [Writing Tasks → changed_when](writing-tasks.md#controlling-changed-and-failed).

## Filesystem mounts (ansible.posix)

```yaml
- name: Mount the NAS backup share and add it to fstab
  ansible.posix.mount:
    path: /mnt/backups
    src: "//nas.internal/backups"
    fstype: cifs
    opts: "credentials=/etc/samba/nas-creds,uid=1000,gid=1000"
    state: mounted            # mounted | present | unmounted | absent
```

## Debugging & facts

```yaml
- name: Print a message or variable
  ansible.builtin.debug:
    msg: "Deploying to {{ inventory_hostname }}"

- name: Print a variable's value
  ansible.builtin.debug:
    var: base_packages

- name: Gather system facts on demand
  ansible.builtin.setup:
    filter: ansible_distribution*
```

See [Facts & Variables](facts.md) for what `setup` collects.

## Where these come from

- `ansible.builtin.*` — ship with Ansible, always available.
- `ansible.posix.*` (e.g. `mount`) and `community.general.*` (e.g. `timezone`) —
  live in **collections** you install. See [Collections & Galaxy](collections.md).

Ref: [Ansible — Module Index](https://docs.ansible.com/projects/ansible/latest/collections/index_module.html)
