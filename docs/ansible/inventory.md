# Ansible Inventory

Ansible automates tasks on managed nodes or “hosts” in your infrastructure by using a list or group of lists known as inventory. Ansible composes its inventory from one or more ‘inventory sources’. While one of these sources can be the list of host names you pass at the command line, most Ansible users create inventory files. Your inventory defines the managed nodes you automate and the variables associated with those hosts. You can also specify groups. Groups allow you to reference multiple associated hosts to target for your automation or to define variables in bulk. Once you define your inventory, you use [patterns](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_patterns.html#intro-patterns) to select the hosts or groups you want Ansible to run against.

---

## inventory/hosts.ini

```ini
[infra]
ops-01.internal

[apps]
prod-deb-01.internal

[standby]
dev-01.internal

[nas]
nas.internal

# Parent group containing all managed homelab nodes (not NAS)
[homelab:children]
infra
apps
standby
```

**How groups work:** When you target `homelab`, it hits ops-01, prod-deb-01, and dev-01. When you target `infra`, it only hits ops-01. You can also target individual hosts by name.

Ref: https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html

---