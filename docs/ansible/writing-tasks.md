---
icon: lucide/list-checks
title: Writing Tasks
description: How Ansible tasks work — modules, loops, conditionals, register, handlers, and blocks.
---

# Writing Tasks

A **task** is a single unit of work — "install this package," "copy this file,"
"restart this service." Tasks live in a role's `tasks/main.yml` (or directly in a
playbook) and run **top to bottom**. This page is the "how do I actually write
Ansible" reference.

!!! tip "The golden rule: describe the end state, not the steps"

    Ansible is **declarative**. You don't say "run `apt install nginx`" — you say
    "nginx should be `present`." Ansible checks the current state and only changes
    what's needed. Run it twice and the second run reports `changed=0`. That
    property is called **idempotency**, and it's the whole point.

## Anatomy of a task

```yaml
- name: Install nginx            # (1) human-readable label (always add one)
  ansible.builtin.apt:           # (2) the module — the thing that does the work
    name: nginx                  # (3) module arguments
    state: present
  become: true                   # (4) run this task with sudo
  when: ansible_os_family == "Debian"   # (5) only run if this is true
  notify: restart nginx          # (6) fire a handler if this task changes anything
  tags: [web]                    # (7) label for --tags / --skip-tags
```

1. **`name`** — shows in the output. Unnamed tasks are hard to read; always name them.
2. **Module** — `apt`, `copy`, `template`, `service`, etc. Use the **fully
   qualified name** (`ansible.builtin.apt`) — see [Collections](collections.md).
3. **Arguments** — indented under the module.
4. **`become`** — privilege escalation (sudo). Can be set per-task, per-play, or
   globally in [ansible.cfg](configuration-files.md).
5. **`when`** — a conditional (below).
6. **`notify`** — triggers a [handler](#handlers).
7. **`tags`** — for running subsets ([ad-hoc & check mode](ad-hoc.md)).

## Variables in tasks

Reference variables with `{{ }}`. They come from
[group_vars/host_vars, facts, or register](variables.md).

```yaml
- name: Create the app directory
  ansible.builtin.file:
    path: "{{ docker_compose_dir }}/myapp"   # value from group_vars/all.yml
    state: directory
    owner: "{{ main_user }}"
    mode: "0755"
```

!!! warning "Quote values that start with `{{`"

    YAML reads a leading `{` as the start of a dictionary. If a value **starts**
    with a variable, wrap the whole thing in quotes: `path: "{{ x }}/sub"`, not
    `path: {{ x }}/sub`.

## Conditionals — `when`

Run a task only when an expression is true. No `{{ }}` needed inside `when` — it's
already a Jinja2 expression.

```yaml
- name: Install a package only on Debian
  ansible.builtin.apt:
    name: htop
  when: ansible_os_family == "Debian"

- name: Multiple conditions (AND — all must be true)
  ansible.builtin.command: /opt/setup.sh
  when:
    - ansible_distribution == "Debian"
    - ansible_distribution_major_version | int >= 12

- name: OR condition
  ansible.builtin.debug:
    msg: "small box"
  when: ansible_memtotal_mb < 2048 or ansible_processor_vcpus < 2

- name: Run only if a variable is defined / not defined
  ansible.builtin.debug:
    msg: "custom path set"
  when: custom_path is defined

- name: Run based on a previous task's result
  ansible.builtin.service:
    name: docker
    state: restarted
  when: config_file.changed          # see 'register' below
```

## Loops — `loop`

Repeat a task over a list. The current item is `item`.

```yaml
- name: Install several packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop:
    - curl
    - git
    - htop

- name: Create several directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
  loop: "{{ borg_backup_targets }}"   # loop over a variable that holds a list

- name: Loop over a list of dictionaries
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
  loop:
    - { name: alice, groups: sudo }
    - { name: bob,   groups: docker }
```

!!! tip "Many modules take a list directly — no loop needed"

    `apt`, `package`, and others accept a list for `name:`, which is faster (one
    transaction) than looping:

    ```yaml
    - name: Install base packages (single apt transaction)
      ansible.builtin.apt:
        name: "{{ base_packages }}"
        state: present
    ```

## Capturing output — `register`

Store a task's result in a variable to use later.

```yaml
- name: Check if a file exists
  ansible.builtin.stat:
    path: /etc/borgmatic/config.yaml
  register: borg_cfg

- name: Only initialize if it doesn't exist yet
  ansible.builtin.command: borgmatic init --encryption repokey
  when: not borg_cfg.stat.exists

- name: Run a command and capture output
  ansible.builtin.command: docker --version
  register: docker_version
  changed_when: false            # a read-only command shouldn't report "changed"

- name: Show what we captured
  ansible.builtin.debug:
    msg: "{{ docker_version.stdout }}"
```

Common fields on a registered result: `.stdout`, `.stdout_lines`, `.rc` (return
code), `.changed`, `.failed`, and (for `stat`) `.stat.exists`.

## Controlling "changed" and "failed"

`command`/`shell` can't know whether they changed anything, so they **always**
report `changed`. Tell Ansible the truth:

```yaml
- name: This never changes state — just reads
  ansible.builtin.command: docker ps
  changed_when: false

- name: Only "changed" when output contains a marker
  ansible.builtin.shell: /opt/deploy.sh
  register: deploy
  changed_when: "'DEPLOYED' in deploy.stdout"

- name: Don't fail the play if this returns non-zero
  ansible.builtin.command: /opt/optional-check.sh
  register: check
  failed_when: false            # or: failed_when: check.rc not in [0, 2]
```

!!! tip "Prefer a real module over `command`/`shell`"

    A dedicated module (`apt`, `copy`, `file`, `service`, `lineinfile`) is
    idempotent for free and reports `changed` accurately. Reach for
    `command`/`shell` only when no module fits. See [Modules](modules.md).

## Handlers

A **handler** is a task that only runs when **notified**, and only **once** at the
end of the play — perfect for "restart the service, but only if its config
actually changed."

```yaml
# roles/base/tasks/main.yml
- name: Configure SSH daemon
  ansible.builtin.template:
    src: sshd_config.j2
    dest: /etc/ssh/sshd_config
    validate: "sshd -t -f %s"    # test the config before saving — avoids lockout
  notify: restart sshd           # fires ONLY if the file changed
```

```yaml
# roles/base/handlers/main.yml
- name: restart sshd
  ansible.builtin.service:
    name: sshd
    state: restarted
```

If the template renders identical to what's already there, the task reports
`ok` (not `changed`), the handler is **not** notified, and sshd is left alone.

## Grouping tasks — `block`

A `block` groups tasks so they share directives (`when`, `become`, tags) and can
have error handling — Ansible's version of try/except.

```yaml
- name: Set up the app (with rollback on failure)
  block:
    - name: Deploy compose file
      ansible.builtin.template:
        src: app-compose.yml.j2
        dest: /opt/docker/app/docker-compose.yml
    - name: Start the stack
      ansible.builtin.command:
        cmd: docker compose up -d
        chdir: /opt/docker/app
  rescue:
    - name: Roll back on failure
      ansible.builtin.debug:
        msg: "Deploy failed — check the compose file"
  always:
    - name: This runs no matter what
      ansible.builtin.command: docker ps
  become: true                   # applies to every task in the block
```

## Putting it together

A realistic task file reads like a checklist of desired states:

```yaml
---
- name: Install Docker
  ansible.builtin.apt:
    name: docker-ce
    state: present

- name: Create app directory
  ansible.builtin.file:
    path: "{{ docker_compose_dir }}/myapp"
    state: directory
    owner: "{{ main_user }}"

- name: Deploy compose file
  ansible.builtin.template:
    src: myapp-compose.yml.j2
    dest: "{{ docker_compose_dir }}/myapp/docker-compose.yml"
  register: compose_file

- name: Start the stack only if the compose file changed
  ansible.builtin.command:
    cmd: docker compose up -d
    chdir: "{{ docker_compose_dir }}/myapp"
  when: compose_file.changed
```

## Next

- [Common Modules](modules.md) — the toolbox of things tasks can do
- [Templates & Jinja2](templates-jinja2.md) — the `.j2` files tasks deploy
- [Ad-hoc Commands & Check Mode](ad-hoc.md) — test tasks safely before running

Ref: [Ansible — Tasks & Playbooks](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_intro.html)
