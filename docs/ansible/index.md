---
icon: simple/ansible
title: Ansible
---

Configuration management tool for automating provisioning and maintaining desired state across multiple hosts from a single control node.

<div class="grid cards" markdown>

-   :simple-ansible:{ .lg .middle } __[Overview](ansible.md)__

    ---

    What Ansible is, how it works, and initial SSH key setup.

-   :lucide-download:{ .lg .middle } __[Installation](ansible.md#installation)__

    ---

    Installing Ansible on the control node.

-   :lucide-list-checks:{ .lg .middle } __[Writing Tasks](writing-tasks.md)__

    ---

    How tasks work — modules, loops, conditionals, register, handlers, and blocks.

-   :lucide-blocks:{ .lg .middle } __[Common Modules](modules.md)__

    ---

    The toolbox — apt, copy, template, file, service, and the rest, with examples.

-   :lucide-file-code-2:{ .lg .middle } __[Templates & Jinja2](templates-jinja2.md)__

    ---

    Writing `.j2` files — Jinja2 variables, filters, loops, and conditionals.

-   :lucide-search-code:{ .lg .middle } __[Facts & Variables](facts.md)__

    ---

    System facts, magic variables, and which value wins when names collide.

-   :lucide-library:{ .lg .middle } __[Collections & Galaxy](collections.md)__

    ---

    Where modules come from, FQCNs, and installing collections.

-   :lucide-terminal-square:{ .lg .middle } __[Ad-hoc & Safe Runs](ad-hoc.md)__

    ---

    One-off commands, check mode, `--diff`, tags, and limiting scope.

-   :lucide-circle-pile:{ .lg .middle } __[Inventory](inventory.md)__

    ---

    Defining managed hosts and running ad hoc commands.

-   :lucide-variable:{ .lg .middle } __[Variables](variables.md)__

    ---

    Variables let you define values once and reuse them.

-   :lucide-lock:{ .lg .middle } __[Vault](vault.md)__

    ---

    Encrypting secrets and sensitive variables in playbooks and inventory.



-   :lucide-package:{ .lg .middle } __[Roles](roles.md)__

    ---

    Structuring playbooks into reusable, portable units.

-   :lucide-book:{ .lg .middle } __[Playbooks](playbooks.md)__

    ---

    Repeatable, ordered task definitions for configuring hosts.

-   :simple-docker:{ .lg .middle } __[Compose Templates](composetemplates.md)__

    ---

    Templates for running Docker containers with Ansible.


</div>