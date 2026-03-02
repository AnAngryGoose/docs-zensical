# Building Inventories & Running Ad Hoc Commands

[Official Docs :simple-ansible: ](https://docs.ansible.com/projects/ansible/latest/inventory_guide/index.html)

---

An **inventory** is a list of managed nodes, or hosts, that Ansible deploys and configures.

**Create an `inventory` file**

    nano inventory

Within the file add the IPs to be configured

    10.10.30.10
    10.10.30.20
    10.10.30.30

**Add the inventory file to version control**

    git add inventory

**Commit the changes**

    git commit -m "first version of the inventory file, added three hosts."

**Push commit to Github**

    git push origin master

**Test Ansible is working**

    ansible all --key-file ~/.ssh/ansible -i inventory -m ping
    # -i INVETORY 
    # -m MODULE_NAME (action to execute, in this case the ping command) - note that `ping` here is an actual ssh connection. Not a normal ping command. 


* This command will send a `ping` to all the hosts inside the `inventory` file
* If successful, this should return something like: 

```  
  10.10.30.20 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```
  
**Create ansible config file**

Inside your repository directory:

    nano ansible.cfg
 
    [defaults]
    inventory = inventory 
    private_key_file = ~/.ssh/ansible

**Now the ansible command can be simplified**

    ansible all -m ping

**List all of the hosts in the inventory**

    ansible all --list-hosts

**Gather facts about your hosts**

    ansible all -m gather_facts

This will be very long. You can search for something specific by filtering with `grep`

**Gather facts about your hosts, but limit it to just one host**

    ansible all -m gather_facts --limit 172.16.250.132