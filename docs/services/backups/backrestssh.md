# Backrest - Using SSH (SFTP) Remotes with Docker Compose

---

### [Step 1: Create a Local Directory for SSH Config](https://garethgeorge.github.io/backrest/cookbooks/ssh-remote#step-1-create-a-local-directory-for-ssh-config)

First, create a directory to store your SSH key and configuration files. This keeps your Backrest-related files organized.

```
mkdir -p ./backrest/ssh
```

### [Step 2: Generate an SSH Key](https://garethgeorge.github.io/backrest/cookbooks/ssh-remote#step-2-generate-an-ssh-key)

Next, generate a new SSH key pair specifically for Backrest.

```
ssh-keygen -t ed25519 -f ./backrest/ssh/id_rsa -C "backrest-backup-key"
```

When prompted for a passphrase, you can leave it empty by pressing Enter. Using a passphrase adds another layer of security but requires more complex setup to use with an automated tool like Backrest.

### [Step 3: Copy the Public Key to Your Remote Server](https://garethgeorge.github.io/backrest/cookbooks/ssh-remote#step-3-copy-the-public-key-to-your-remote-server)

Copy the public key to your remote server's `authorized_keys` file. The `ssh-copy-id` command is the easiest way to do this.

```
# Replace your-username and example.com with your remote server's details
ssh-copy-id -i ./backrest/ssh/id_rsa.pub your-username@example.com
```


### [Step 4: Create the SSH Config and Known Hosts Files](https://garethgeorge.github.io/backrest/cookbooks/ssh-remote#step-4-create-the-ssh-config-and-known-hosts-files)

Create an SSH configuration file that Restic (inside the container) will use to connect.

```back
# Create the config file
cat > ./backrest/ssh/config << EOF
Host backrest-remote
    HostName example.com
    User your-username
    IdentityFile /root/.ssh/id_rsa
    Port 22
EOF

# Add the server's fingerprint to known_hosts
ssh-keyscan -H example.com >> ./backrest/ssh/known_hosts
```

Copy to clipboard

**Important:**

-   **`Host backrest-remote`**: This is a custom alias. You will use this name in the Backrest UI.
-   **`HostName`**: The actual IP address or hostname of your remote server.
-   **`User`**: The username on the remote server.
-   **`IdentityFile`**: This **must be `/root/.ssh/id_rsa`**. This is the path _inside_ the container where the key will be mounted.
-   **`Port`**: The SSH port of your remote server.

### [Step 5: Set Secure Permissions](https://garethgeorge.github.io/backrest/cookbooks/ssh-remote#step-5-set-secure-permissions)

SSH requires that key and configuration files have strict permissions. These permissions need to be set as **root**. To ensure this is read properly via backrest, you can run them via the container itself 

Fix Ownership: Make root the owner of the files inside the container.

```Bash

docker exec -u 0 backrest chown -R root:root /root/.ssh
```
Fix Directory Permissions: Ensure the folder is only accessible by the owner.

```Bash

docker exec -u 0 backrest chmod 700 /root/.ssh
```
Fix File Permissions: Ensure the config and keys are read-only for the owner.

```Bash

docker exec -u 0 backrest chmod 600 /root/.ssh/config /root/.ssh/id_rsa
```

### [Step 6: Mount the SSH Directory in Docker Compose](https://garethgeorge.github.io/backrest/cookbooks/ssh-remote#step-6-mount-the-ssh-directory-in-docker-compose)

Now, edit your `docker-compose.yml` to mount the `backrest/ssh` directory into the container. We mount it as read-only (`:ro`) for better security.

```yaml
version: "3.8"
services:
  backrest:
    image: garethgeorge/backrest:latest
    container_name: backrest
    # ... other configuration ...
    volumes:
      - ./backrest/data:/data
      - ./backrest/config:/config
      - ./backrest/cache:/cache
      # ... other volumes ...
      - ./backrest/ssh:/root/.ssh:ro # Add this line
      - ./backrest/ssh:/.ssh:ro # Add this line if running rootless
    # ... rest of configuration ...
```


After saving the file, restart your container for the changes to take effect:

```
docker compose up -d --force-recreate
```


### [Step 7: Add the Repository in Backrest](https://garethgeorge.github.io/backrest/cookbooks/ssh-remote#step-7-add-the-repository-in-backrest)

!!!important
    Ensure your user has write permissions to the **destination** repo. 

1.  In the Backrest WebUI, navigate to **Repositories** and click **Add Repository**.
2.  For the **Type**, select **Remote/Cloud**.
3.  For the **URL**, enter `sftp:backrest-remote:/path/to/your/repo`.
    -   Replace `backrest-remote` with the `Host` alias you defined in `backrest/ssh/config`.
    -   Replace `/path/to/your/repo` with the absolute path on the remote server where you want to store backups.
4.  Enter a secure **Password** to encrypt your backup data. This is a new password for the repository itself, not your SSH key password.
5.  Click **Initialize Repository**.

## [Troubleshooting](https://garethgeorge.github.io/backrest/cookbooks/ssh-remote#troubleshooting)

-  **Connection Errors:** First, test your SSH connection from the host machine to isolate issues. This command uses the exact same configuration files that the container will use. Running this via the container itself verifies access. Official docs have this run locally, which did not work for me. 


```
docker exec -it backrest ssh -F /root/.ssh/config backrest-remote
```


    
    
-   **Permission Denied:**
    -   Double-check the file permissions set in Step 5.
    -   Ensure the user on the remote server has write permissions to the repository path.
-   **Check Logs:** Review the Backrest application logs for detailed error messages from Restic.