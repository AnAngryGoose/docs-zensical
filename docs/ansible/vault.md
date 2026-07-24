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

## Encrypting a single value

You don't have to encrypt a whole file — you can drop one encrypted string into an
otherwise-plaintext vars file. Useful for a single secret alongside normal config.

```bash
ansible-vault encrypt_string 'super-secret-token' --name 'vault_api_token'
```

It prints a block you paste straight into a YAML file:

```yaml
vault_api_token: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      66386439653...
      33613463...
```

Ansible decrypts it automatically at runtime, just like a fully-encrypted file.

## If you don't use a password file

Without `vault_password_file` in [ansible.cfg](configuration-files.md), pass the
password at runtime:

```bash
ansible-playbook playbooks/base.yml --ask-vault-pass
```

## Best practices

!!! warning "Keep the key out of git"

    The whole point is defeated if the password leaks. Add these to your Ansible
    repo's `.gitignore`:

    ```gitignore
    .vault_pass
    *.vault_pass
    ```

    The **encrypted** `vault.yml` is safe to commit; the **password** file is not.

- **Prefix vaulted variables with `vault_`** (e.g. `vault_nas_samba_password`), then
  map them to normal names in `group_vars/all.yml`:
  `nas_samba_password: "{{ vault_nas_samba_password }}"`. This makes it obvious at a
  glance which values are secret and keeps `no_log` easy to reason about.
- Add `no_log: true` to tasks that handle secrets so they don't print to the
  terminal or logs.
- Store the vault password itself in a real password manager (e.g. Vaultwarden).

Ref: [Ansible — Protecting sensitive data with Vault](https://docs.ansible.com/projects/ansible/latest/vault_guide/index.html)
 · [Encrypting single values](https://docs.ansible.com/projects/ansible/latest/vault_guide/vault_encrypting_content.html#encrypting-individual-variables-with-ansible-vault)
