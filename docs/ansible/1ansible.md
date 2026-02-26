# Ansible 

[Ansible :simple-ansible: ](https://docs.ansible.com/projects/ansible/latest/getting_started/index.html)

---

## Overview

--

Ansible automates the management of remote systems and controls their desired state.

Ansible provides open-source automation that reduces complexity and runs everywhere. Using Ansible lets you automate virtually any task. Here are some common use cases for Ansible:

* Eliminate repetition and simplify workflows

* Manage and maintain system configuration

* Continuously deploy complex software

* Perform zero-downtime rolling updates

Ansible uses simple, human-readable scripts called playbooks to automate your tasks. You declare the desired state of a local or remote system in your playbook. Ansible ensures that the system remains in that state.

![ansible-getting-started.png](https://docs.ansible.com/projects/ansible/latest/_images/ansible_inv_start.svg)

As shown in the preceding figure, most Ansible environments have three main components:

**Control node**

* A system on which Ansible is installed. You run Ansible commands such as `ansible` or `ansible-inventory` on a control node.

**Inventory**

* A list of managed nodes that are logically organized. You create an inventory on the control node to describe host deployments to Ansible.

**Managed node**

* A remote system, or host, that Ansible controls.

## Initial Setup 

---

This will cover the installation and structure required for using Ansible efficiently. 

Ansible be run from **one** computer (control node) to manage the configuration of **multiple** other servers (mananged nodes).

This requires a few things to work.

## SSH Keys

---

Generate an SSH on the **control node**. Add a passphrase, this is your main, secure key. 

    ssh-keygen -t ed25519 -C "controlhostname"

Copy that SSH key to the **mananged node(s)**.

    ssh-copy-id -i ~/.ssh/id_ed25519.pub <IP Address>

Generate an SSH key that is **specifically for Ansible**. No passphrase. This is ansible specific key.

    ssh-keygen -t ed25519 -C "ansible"

Copy that SSH key to the **mananged node(s)**.

    ssh-copy-id -i ~/.ssh/ansible.pub <IP Address>

Connecting using an SSH key. 

    ssh -i .ssh/<key_name> <IP Address>

Cache the passphrase

    eval $(ssh-agent)
    ssh-add

Bash alias for caching the passphrase

    alias ssha='eval $(ssh-agent) && ssh-add'


## Git Repository

---

**Check** if git is installed

    which git
    # This will return `/usr/bin/git` if installed. 

**Install** git

    sudo apt update
    sudo apt install git

**Create user config** for git

    git config --global user.name "First Last"
    git config --global user.email "somebody@somewhere.net"

**Check the status** of your git repository

    git status

**Stage the README.md** file (after making changes) to be included in the next git commit

    git add README.md

**Set up the README.md** file to be included in a commit

    git commit -m "Updated readme file, initial commit"

**Send the commit** to Github

    git push origin master

