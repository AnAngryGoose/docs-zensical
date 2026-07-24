---
icon: lucide/file-code-2
title: Templates & Jinja2
description: Writing .j2 template files — Jinja2 variables, filters, loops, and conditionals.
---

# Templates & Jinja2

A **template** is a config file with placeholders that Ansible fills in per host
before placing it on the machine. It's how one `docker-compose.yml.j2` becomes a
correct, host-specific compose file on every node. Templates use the **Jinja2**
templating language and are conventionally named `*.j2` and kept in a role's
`templates/` directory.

```yaml
# In a task — render templates/borgmatic-config.yml.j2 to the host
- name: Template borgmatic config
  ansible.builtin.template:
    src: borgmatic-config.yml.j2
    dest: /etc/borgmatic/config.yaml
    mode: "0600"
```

## The three Jinja2 delimiters

| Syntax | Purpose | Example |
|---|---|---|
| `{{ ... }}` | **Print** a value | `server: {{ docker_dns }}` |
| `{% ... %}` | **Logic** (loops, if) | `{% for d in dirs %}` |
| `{# ... #}` | **Comment** (not rendered) | `{# TODO: revisit #}` |

## Variables

Anything available to the task — [group_vars/host_vars](variables.md),
[facts](facts.md), registered vars — is usable in the template.

```jinja
# templates/app.conf.j2
timezone = {{ timezone }}
data_dir = {{ appdata_dir }}/app
hostname = {{ inventory_hostname }}
cpus = {{ ansible_processor_vcpus }}          {# a fact #}
```

Access nested values with dots or brackets:

```jinja
first_backup = {{ borg_backup_targets[0] }}
os = {{ ansible_distribution }} {{ ansible_distribution_version }}
```

## Filters — transform a value with `|`

Filters are the workhorses of Jinja2. Chain them with `|`.

```jinja
{# Provide a fallback when a variable might be undefined #}
port = {{ app_port | default(8080) }}

{# Fail loudly if a required var is missing #}
token = {{ vault_api_token | mandatory }}

{# Text #}
name = {{ main_user | upper }}
slug = {{ app_name | lower | replace(' ', '-') }}

{# Lists #}
packages = {{ base_packages | join(', ') }}
count = {{ base_packages | length }}
first = {{ base_packages | first }}

{# Numbers / bytes #}
half = {{ ansible_memtotal_mb | int // 2 }}

{# JSON / YAML output (great for config files) #}
config = {{ my_dict | to_nice_json }}
```

| Filter | Does |
|---|---|
| `default(x)` | Use `x` if the variable is undefined |
| `default(x, true)` | Use `x` if undefined **or empty/false** |
| `mandatory` | Error out if undefined |
| `join(', ')` | List → string |
| `length`, `first`, `last` | List helpers |
| `upper` / `lower` / `replace(a,b)` | Text |
| `int` / `float` | Type conversion |
| `to_nice_json` / `to_nice_yaml` | Serialize a dict/list |
| `bool` | Interpret as true/false |

## Loops

Repeat lines for each item in a list — the pattern behind the borgmatic and
compose templates in this repo.

```jinja
# templates/borgmatic-config.yml.j2
source_directories:
{% for dir in borg_backup_targets %}
  - {{ dir }}
{% endfor %}
```

With a list of dictionaries:

```jinja
{% for user in app_users %}
  - name: {{ user.name }}
    role: {{ user.role | default('member') }}
{% endfor %}
```

`loop.index` (1-based) and `loop.first` / `loop.last` are handy inside loops:

```jinja
{% for host in groups['apps'] %}
server{{ loop.index }} = {{ host }}{% if not loop.last %},{% endif %}
{% endfor %}
```

## Conditionals

```jinja
{% if ansible_memtotal_mb > 8000 %}
worker_processes = 4
{% elif ansible_memtotal_mb > 4000 %}
worker_processes = 2
{% else %}
worker_processes = 1
{% endif %}

{# Inline conditional #}
log_level = {{ 'debug' if debug_enabled | default(false) else 'info' }}

{# Only include a block when a variable is set #}
{% if custom_dns is defined %}
dns = {{ custom_dns }}
{% endif %}
```

## Whitespace control

`{% for %}`/`{% if %}` lines leave blank lines behind in the output. Add a `-` to
trim surrounding whitespace — important for clean YAML/config files.

```jinja
repositories:
{% for repo in repos -%}
  - {{ repo }}
{% endfor -%}
```

- `{%-` trims whitespace **before** the tag.
- `-%}` trims the newline **after** the tag.

!!! tip "Preview a template before trusting it"

    Run the play in [check + diff mode](ad-hoc.md) to see exactly what the rendered
    file will look like without writing it:

    ```bash
    ansible-playbook playbooks/base.yml --check --diff --limit ops-01.internal
    ```

## A real example

```jinja
# templates/homepage-env.j2  — Managed by Ansible, do not edit on the host
HOMEPAGE_ALLOWED_HOSTS={{ inventory_hostname }}.internal,{{ ansible_default_ipv4.address }}
PUID={{ puid | default(1000) }}
PGID={{ pgid | default(1000) }}
TZ={{ timezone }}
{% for var in extra_env | default([]) %}
{{ var.key }}={{ var.value }}
{% endfor %}
```

Always mark generated files so future-you doesn't hand-edit them:

```jinja
# Managed by Ansible -- changes here are overwritten on the next run
```

Ref: [Ansible — Templating (Jinja2)](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_templating.html)
 · [Jinja2 docs](https://jinja.palletsprojects.com/)
