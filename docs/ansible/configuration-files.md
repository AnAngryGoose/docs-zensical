# Ansible Configuration Files

---

## ansible.cfg

This tells Ansible where to find things and how to behave. Place at `~/ansible/ansible.cfg`.

```ini
[defaults]
inventory = inventory/hosts.ini
remote_user = username
private_key_file = ~/.ssh/ansible
roles_path = roles
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

**What each setting does:**

- `inventory`: Default inventory file so you don't have to pass `-i` every time.
- `remote_user`: SSH user on all managed nodes.
- `private_key_file`: The Ansible SSH key you generated.
- `roles_path`: Where to find roles.
- `host_key_checking = False`: Don't prompt on first SSH connection. Safe on a local VLAN.
- `retry_files_enabled = False`: Don't create `.retry` files on failure (they clutter the directory).
- `stdout_callback = yaml`: More readable output format.
- `become = True`: Use sudo by default (most tasks need root).
- `become_ask_pass = False`: Assumes username has passwordless sudo. If not, set this to True or configure sudoers.

**Important:** username needs passwordless sudo on all managed nodes. During OS install or after:

```bash
# Run this on each managed node
echo "username ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/username
sudo chmod 440 /etc/sudoers.d/username
```

Ref: https://docs.ansible.com/projects/ansible/latest/reference_appendices/config.html

---
