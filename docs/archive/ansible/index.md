---
icon: simple/ansible
title: Ansible
---

Configuration management tool for automating provisioning and maintaining desired state across multiple hosts from a single control node.

<div class="grid cards" markdown>

-   :simple-ansible:{ .lg .middle } __[Overview](ansible.md)__

    ---

    What Ansible is, how it works, and initial SSH key setup.

-   :lucide-download:{ .lg .middle } __[Installation](ansibleinstallation.md)__

    ---

    Installing Ansible on the control node.

-   :lucide-circle-pile:{ .lg .middle } __[Inventory](ansibleinventory.md)__

    ---

    Defining managed hosts and running ad hoc commands.

-   :lucide-terminal:{ .lg .middle } __[Commands](ansiblecommands.md)__

    ---

    Elevated ad hoc commands for making changes on managed nodes.

-   :lucide-book:{ .lg .middle } __[Playbooks](ansibleplaybook.md)__

    ---

    Repeatable, ordered task definitions for configuring hosts.

-   :lucide-variable:{ .lg .middle } __[Variables](ansiblevars.md)__

    ---

    group_vars, host_vars, and vars_files for managing per-host and per-group configuration.

-   :lucide-package:{ .lg .middle } __[Roles](ansibleroles.md)__

    ---

    Structuring playbooks into reusable, portable units.

-   :lucide-tag:{ .lg .middle } __[Tags & Handlers](ansibletagshandlers.md)__

    ---

    Running targeted parts of a playbook and triggering tasks on change.

-   :lucide-lock:{ .lg .middle } __[Vault](ansiblevault.md)__

    ---

    Encrypting secrets and sensitive variables in playbooks and inventory.

</div>