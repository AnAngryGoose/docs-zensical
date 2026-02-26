# Git Reference & Version Control

---

## Overview

**Git** is a distributed version control system used to track changes in source code. In a homelab context, it serves as the backbone of "Infrastructure as Code" (IaC). By storing `docker-compose.yml` and configuration files in Git, the entire server state is backed up, versioned, and easily reproducible.

This guide covers the initialization of a local repository, security practices for managing secrets, and daily workflows using both the Command Line Interface (CLI) and VS Code.

I don't know much about the other stuff in git. I just use the basics, but this gets me by. 

---

## 1. Initialization (One-Time Setup)

Perform these steps when setting up the machine for the first time to establish the repository.

### 1.1. Installation & Identity

First, ensure Git is installed and configure the user identity. This information is embedded in every commit.

```bash
# Install Git
sudo apt update && sudo apt install git -y

# Configure Global Identity
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Set default branch to 'main' (Modern Standard)
git config --global init.defaultBranch main 

```

### 1.2. Initialize Repo

Turn the Docker directory into a tracked repository.

```bash
cd ~/homelab/docker/
git init

```

### 1.3. The Ignore Rules (Critical)

A `.gitignore` file prevents sensitive data (secrets) and system clutter from being uploaded to the remote server.

**File Location:** `~/homelab/docker/.gitignore`

```bash
# --- Secrets (NEVER COMMIT) ---
**/.env
**/.env.*
**/secrets/
id_rsa
*.pem

# --- System Junk ---
.DS_Store
Thumbs.db
**/.git/   # Ignore nested git repos (prevents submodule errors)

# --- Docker Data (If mapped locally) ---
**/data/
**/mysql/
**/influxdb/ 

# --- Custom Ignores ---
**/.pastebin
**/.pastebin/

```

### 1.4. Connect to Remote (GitHub)

Link the local folder to a GitHub repository for off-site backup.

```bash
# 1. Link Remote
git remote add origin https://github.com/YourUser/homelab-backup.git

# 2. First Push
git add .
git commit -m "Initial homelab commit"
git push -u origin main

```

---

## 2. Daily Workflows (CLI)

### 2.1. The "Config Change" Routine

Execute this routine whenever a `compose.yaml` or configuration file is edited.

1. **Check Status:**
```bash
git status

```


* *Verify:* Ensure no `.env` files appear in the untracked list (Red).


2. **Stage & Commit:**
```bash
git add .
git commit -m "Added Tautulli to media stack"

```


3. **Push:**
```bash
git push

```



### 2.2. Disaster Recovery (Rebuild)

If the host OS drive fails, use this workflow to restore the stack on a fresh installation.

1. **Clone Configs:**
```bash
git clone https://github.com/YourUser/homelab-backup.git ~/homelab/docker

```


2. **Restore Secrets (Manual):**
Since `.env` files are ignored for security, they must be manually recreated from a password manager (e.g., Bitwarden).
```bash
cd ~/homelab/docker/core
nano .env 
# Paste API keys/passwords here

```


3. **Launch:**
```bash
docker compose up -d

```



---

## 3. Security: Secret Detection

*Objective: Prevent accidental leakage of passwords, API keys, or `.env` contents into the public Git history.*

### 3.1. Visual Inspection

Before staging files, review changes to ensure no hardcoded credentials were added.

| Command | Description |
| --- | --- |
| `git diff` | Shows line-by-line changes for **unstaged** files. |
| `git diff --staged` | Shows changes for files **already staged** but not committed. |

### 3.2. Automated Scanning (Gitleaks via Docker)

Use **Gitleaks** to scan the directory for high-entropy strings (random characters that look like API keys) without installing new software on the host.

```bash
# Run inside ~/homelab/docker
docker run --rm \
  -v $(pwd):/path \
  zricethezav/gitleaks:latest \
  detect --source="/path" -v

```

* **Success:** "No leaks found."
* **Failure:** Output lists the specific secret and file location.

---

## 4. VS Code Integration

### 4.1. Visualizing Diffs

To view diffs directly in VS Code from the terminal:

**One-off:**

```bash
git diff | code -

```

**Permanent Configuration:**

```bash
git config --global diff.tool vscode
git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'
# Usage:
git difftool

```

### 4.2. File Explorer Indicators

Understanding the colored badges in the VS Code file explorer.

| Badge | Color (Gruvbox) | Meaning | Action Needed |
| --- | --- | --- | --- |
| **U** | **Green** | **Untracked** | File is new. Needs `git add`. |
| **M** | **Blue** | **Modified** | File is edited but not staged. |
| **A** | **Green** | **Added** | File is staged and ready to commit. |
| **D** | **Red** | **Deleted** | File removed locally. |
| **-** | **Gray** | **Ignored** | Matches `.gitignore` rule. |

### 4.3. GUI Workflow

The "Happy Path" using the Source Control Tab (`Ctrl+Shift+G`):

1. **Edit & Save:** Make changes and save (`Ctrl+S`).
2. **Review:** Click the file under **Changes** to see the Side-by-Side Diff.
3. **Stage:** Click the **`+` (Plus)** icon next to the file.
4. **Commit:** Type a message in the input box  Click **Commit**.
5. **Sync:** Click **Sync Changes** (Auto Pulls & Pushes).

---

## 5. Troubleshooting

### 5.1. "Submodule" Error (Gray Folder on GitHub)

**Symptom:** A project (like a theme) was `git clone`d inside the main repo. GitHub sees the nested `.git` folder and treats it as a dead link (submodule).

**Fix:**

```bash
# 1. Remove the nested .git folder
rm -rf path/to/subfolder/.git

# 2. Remove the folder from Git's index (cache) only
git rm --cached path/to/subfolder

# 3. Re-add the folder as standard files
git add .
git commit -m "Converted submodule to standard files"

```

### 5.2. Credentials Issues (HTTPS)

**Symptom:** Git prompts for a username/password on every push.

**Fix (Cache Credentials):**

```bash
# Cache credentials in memory for 1 hour (3600 seconds)
git config --global credential.helper 'cache --timeout=3600'

```

---

## 6. Command Cheat Sheet

### Basic Operations

| Action | Command |
| --- | --- |
| **Check State** | `git status` (Always run this first) |
| **View Changes** | `git diff` |
| **Stage Files** | `git add .` (Stage all) or `git add <filename>` |
| **Commit** | `git commit -m "Your message here"` |
| **Push** | `git push origin main` |
| **Pull Updates** | `git pull` |
| **Undo File Edit** | `git restore <file>` (Discards local changes) |
| **View History** | `git log --oneline --graph --all` |

### Emergency Fixes

| Scenario | Workflow / Command |
| --- | --- |
| **Typos in last commit** | `git add .`  `git commit --amend -m "New Message"`  `git push -f` |
| **Leaked Secret** | `git rm --cached .env`  `echo ".env" >> .gitignore`  `git commit -m "rm secret"` |
| **Reset to Remote** | `git fetch origin`  `git reset --hard origin/main` (Destructive: makes local match cloud) |

---
