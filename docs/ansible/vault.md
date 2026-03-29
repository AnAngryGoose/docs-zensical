# Sensitive Data -- Ansible Vault

Passwords and secrets should not be in plain text. Ansible Vault encrypts them.

```bash
# Create a vault password file (so you don't type it every time)
echo "your-vault-password-here" > ~/ansible/.vault_pass
chmod 600 ~/ansible/.vault_pass
```

Add to `ansible.cfg` under `[defaults]`:

```ini
vault_password_file = .vault_pass
```

Create encrypted variables:

```bash
# Create an encrypted vars file for secrets
ansible-vault create inventory/group_vars/vault.yml
```

Put your secrets in there:

```yaml
---
vault_nas_samba_password: "your-samba-password"
vault_vaultwarden_admin_token: "generate-a-long-random-string"
vault_cloudflared_token: "your-cloudflare-tunnel-token"
vault_forgejo_secret_key: "generate-a-long-random-string"
```

Reference vault variables in other files with:

```yaml
nas_samba_password: "{{ vault_nas_samba_password }}"
```

Commands:

```bash
# Edit the vault file
ansible-vault edit inventory/group_vars/vault.yml

# View without editing
ansible-vault view inventory/group_vars/vault.yml

# Re-encrypt with a new password
ansible-vault rekey inventory/group_vars/vault.yml
```

Ref: https://docs.ansible.com/projects/ansible/latest/vault_guide/index.html
