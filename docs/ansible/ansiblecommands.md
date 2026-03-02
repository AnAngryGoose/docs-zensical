# Elevated Commands

[Official Docs :simple-ansible: ](https://docs.ansible.com/projects/ansible/latest/command_guide/index.html)

https://www.youtube.com/watch?v=FPU9_KDTa8A&list=PLT98CRl2KxKEUHie1m24-wkyHpEsa4Y70&index=5

---

Unlike the commands listed in the [inventory and ad hoc guide](../ansibleinventory/), elevated commands will make system changes on the managed nodes.

**Tell ansible to use sudo (become)**

This is needed in order to actually become a `sudo` user on the managed node. 

The `--become --ask-become-pass` is what makes the preceding command possible to run. 

    ansible all -m apt -a update_cache=true --become --ask-become-pass

When prompted for password, input the password for the `sudo` account user. 

!!!note
    By default, `become` will be using `sudo`. There are different approaches when using become, but that is not covered here. 

**Install a package via the apt module**

    ansible all -m apt -a name=vim-nox --become --ask-become-pass


**Install a package via the apt module, and also make sure it’s the latest version available**

    ansible all -m apt -a "name=nano state=latest" --become --ask-become-pass

**Upgrade all the package updates that are available**

    ansible all -m apt -a upgrade=dist --become --ask-become-passxx

