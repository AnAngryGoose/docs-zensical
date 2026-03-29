# Troubleshooting

### "Permission denied" on SSH

```bash
# Verify the key works manually
ssh -i ~/.ssh/ansible -v username@ops-01.internal

# Check the key is in authorized_keys on the remote host
ssh username@ops-01.internal "cat ~/.ssh/authorized_keys"
```

### "Missing sudo password"

The username user needs passwordless sudo. On the managed node:

```bash
echo "username ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/username
sudo chmod 440 /etc/sudoers.d/username
```

### "Could not resolve host"

Check DNS. From Control Node:

```bash
dig ops-01.internal
```

If it doesn't resolve, add the host in OPNsense dnsmasq.

### Template errors

If a template fails, check the variable exists:

```bash
# Show all variables for a host
ansible ops-01.internal -m debug -a "var=hostvars[inventory_hostname]"
```

### "changed" on every run

If a task shows "changed" every time you run it, it's not idempotent. Common causes:
- Using `ansible.builtin.command` or `ansible.builtin.shell` (these always show changed). Add `creates:` or `when:` conditions.
- Templates that include timestamps or dynamic content.

### View verbose output

Add `-v` flags for more detail:

```bash
ansible-playbook playbooks/base.yml -v     # some detail
ansible-playbook playbooks/base.yml -vv    # more detail
ansible-playbook playbooks/base.yml -vvv   # SSH-level debug
```