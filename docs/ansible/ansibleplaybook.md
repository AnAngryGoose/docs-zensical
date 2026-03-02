# Playbooks

[Official Docs :simple-ansible: ](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_intro.html)

---

**Ansible Playbooks** provide a repeatable, reusable, simple configuration management and multimachine deployment system that is well suited to deploying complex applications. If you need to execute a task with Ansible more than once, you can write a playbook and put the playbook under source control. You can then use the playbook to push new configurations or confirm the configuration of remote systems.

Playbooks allow you to perform the following actions:

* Declare configurations.

* Orchestrate steps of any manual ordered process on multiple sets of machines in a defined order.

* Launch tasks synchronously or asynchronously.
  
## Creating a playbook

---

Examples below. Create this in the same directory as your `inventory` file. 


**install_apache.yml**

```yaml
 --- # be sure to add these 3 hyphens
 
 - hosts: all # Note the - (hyphen) at beginning of block. 
   become: true # run as sudo
   tasks: # begin list of tasks
 
   - name: install apache2 package # keep descriptive, what task does
     apt: # module to run
       name: apache2 # what to install 
```

**Run the playbook**

    ansible-playbook --ask-become-pass install_apache.yml

**install_apache.yml (second version)**

```yaml
 ---
 
 - hosts: all
   become: true
   tasks:
 
   - name: update repository index
     apt:
       update_cache: yes
 
   - name: install apache2 package
     apt:
       name: apache2
```
**install_apache.yml (third version)**

```yaml
 ---
 
 - hosts: all
   become: true
   tasks:
 
   - name: update repository index
     apt:
       update_cache: yes
 
   - name: install apache2 package
     apt:
     name: apache2
 
   - name: add php support for apache
     apt:
       name: libapache2-mod-php
```
**install_apache.yml (fourth version)**

```yaml
 ---
 
 - hosts: all
   become: true
   tasks:
 
   - name: update repository index
     apt:
       update_cache: yes
 
   - name: install apache2 package
     apt:
       name: apache2
       state: latest
 
   - name: add php support for apache
     apt:
       name: libapache2-mod-php
       state: latest
```
**remove_apache.yml**

```yaml
 ---
 
 - hosts: all
   become: true
   tasks:
 
   - name: remove apache2 package
     apt:
       name: apache2
       state: absent
 
   - name: remove php support for apache
     apt:
       name: libapache2-mod-php
       state: absent
```