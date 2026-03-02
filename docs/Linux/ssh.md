---
title: SSH Hardening & Key Management
description: Secure SSH configuration, key-based authentication, and multi-host management.
---

# SSH Hardening & Key Management

> **Reference:** [OpenSSH Docs](https://www.openssh.com/manual.html) · [Debian Wiki — SSH](https://wiki.debian.org/SSH) · [Ubuntu Docs — OpenSSH](https://help.ubuntu.com/community/SSH/OpenSSH/Configuring)

---

## Key-Based Authentication

Password authentication is convenient but weak. Key-based auth should be used for all server access.

### Generate a Key Pair

```bash
# Generate ED25519 key (preferred — smaller, faster, more secure than RSA)
ssh-keygen -t ed25519 -C "your-label"

# RSA if ED25519 isn't supported by a target system
ssh-keygen -t rsa -b 4096 -C "your-label"
```

Keys are saved to `~/.ssh/` by default:

- `~/.ssh/id_ed25519` — private key (never share or copy this)
- `~/.ssh/id_ed25519.pub` — public key (this gets added to servers)

---

### Copy Public Key to a Server

```bash
# Preferred method
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server

# Manual method if ssh-copy-id isn't available
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Verify correct permissions on the server after:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

### SSH Config File (`~/.ssh/config`)

Saves typing for hosts you connect to regularly:

```bash
# ~/.ssh/config

Host appserver
    HostName 192.168.1.10
    User youruser
    IdentityFile ~/.ssh/id_ed25519
    Port 22

Host nas
    HostName 192.168.1.20
    User youruser
    IdentityFile ~/.ssh/id_ed25519

Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

Now connect with just `ssh appserver` instead of the full command.

---

## Hardening `sshd_config`

The server-side SSH config lives at `/etc/ssh/sshd_config`. Edit with:

```bash
sudo nano /etc/ssh/sshd_config
```

### Key Settings

```bash
# Change default port (reduces automated scanning noise)
Port 2222

# Disable root login — always
PermitRootLogin no

# Disable password auth — only after keys are working
PasswordAuthentication no

# Disable empty passwords
PermitEmptyPasswords no

# Limit to specific users
AllowUsers youruser

# Limit to specific groups
AllowGroups sshusers

# Disable unused authentication methods
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication no

# Reduce login grace time
LoginGraceTime 30

# Limit concurrent unauthenticated connections
MaxStartups 3:50:10

# Disconnect idle sessions after 5 minutes (300s × 3 attempts)
ClientAliveInterval 300
ClientAliveCountMax 3
```

!!! warning "Disable passwords only after confirming key auth works"
    Test key-based login in a **separate terminal session** before setting `PasswordAuthentication no`. Locking yourself out requires console access to fix.

### Apply Changes

```bash
# Test config for syntax errors before reloading
sudo sshd -t

# Reload (no active sessions dropped)
sudo systemctl reload ssh

# Debian uses 'ssh', Ubuntu sometimes uses 'sshd'
sudo systemctl reload sshd   # if above fails
```

!!! note "Debian vs Ubuntu"
    The service name differs slightly:

    - Debian: `sudo systemctl reload ssh`
    - Ubuntu: `sudo systemctl reload ssh` (recent) or `sudo systemctl reload sshd` (older)

    Check with `systemctl status ssh` or `systemctl status sshd`.

---

## SSH Agent

The SSH agent holds your decrypted private key in memory so you don't re-enter your passphrase every connection.

```bash
# Start agent (usually auto-started by desktop environments)
eval "$(ssh-agent -s)"

# Add key to agent
ssh-add ~/.ssh/id_ed25519

# List loaded keys
ssh-add -l

# Remove all keys from agent
ssh-add -D
```

---

## Agent Forwarding

Forward your local SSH agent to a remote host — useful for hopping between servers without copying private keys.

```bash
# Connect with forwarding
ssh -A user@jumphost

# Or set in ~/.ssh/config
Host jumphost
    ForwardAgent yes
```

!!! warning
    Only forward your agent to hosts you fully trust. Anyone with root on the remote host can use your forwarded agent.

---

## Useful SSH Commands

```bash
# Copy files securely
scp file.txt user@server:/destination/
scp -r directory/ user@server:/destination/

# Mount remote filesystem locally (requires sshfs)
sshfs user@server:/remote/path /local/mountpoint

# Run a remote command without interactive shell
ssh user@server "sudo systemctl restart myservice"

# Create a SOCKS proxy through a remote host
ssh -D 1080 user@server

# Port forward — access remote port 8080 locally on 8080
ssh -L 8080:localhost:8080 user@server

# Check which keys are authorized on a server
cat ~/.ssh/authorized_keys
```

---

## Permissions Reference

SSH is strict about permissions — wrong permissions cause silent auth failures.

| Path | Required Permission |
|------|-------------------|
| `~/.ssh/` | `700` |
| `~/.ssh/authorized_keys` | `600` |
| `~/.ssh/id_ed25519` (private) | `600` |
| `~/.ssh/id_ed25519.pub` (public) | `644` |
| `~/.ssh/config` | `600` |

```bash
# Fix all at once
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys ~/.ssh/id_ed25519 ~/.ssh/config
chmod 644 ~/.ssh/id_ed25519.pub
```