# Self-Hosted Obsidian LiveSync with Tailscale

[Github :simple-github:](https://github.com/vrtmrz/obsidian-livesync)

---
## Overview

This covers setting up a self-hosted synchronization backend for Obsidian using **CouchDB** and **Obsidian LiveSync**. I used the offical guide and it...did not go well, so here's an adjusted version 

**Tailscale** is used to secure the connection, allowing devices to sync remotely without opening ports on the router or exposing the database to the public internet.

---

## Installation

### 1. File Permissions (Critical)

The official CouchDB image runs as a specific internal user with ID `5984`. Docker typically creates volume directories as `root`, which causes the container to crash with `Permission denied` errors.

You must pre-create the directories and assign the correct ownership before starting the container.

```bash
mkdir -p ./couchdb-data ./couchdb-etc
sudo chown -R 5984:5984 ./couchdb-data
sudo chown -R 5984:5984 ./couchdb-etc

```

!!! note
If you need to edit files in `couchdb-etc` manually later, you may need to temporarily change ownership back to your user (`sudo chown -R $USER:$USER ...`) or use `sudo`.

### 2. Configuration File (`docker.ini`)

A configuration file must be injected to handle Large File Support (LFS), CORS headers (essential for remote access), and authentication limits.

Create the file at `./couchdb-etc/docker.ini`:

```ini
[couchdb]
single_node = true
max_document_size = 50000000

[chttpd]
bind_address = 0.0.0.0
port = 5984
require_valid_user = true
max_http_request_size = 4294967296
enable_cors = true

[chttpd_auth]
require_valid_user = true
authentication_redirect = /_utils/session.html

[httpd]
WWW-Authenticate = Basic realm="couchdb"
enable_cors = true

[cors]
# Wildcard required for Tailscale/VPN roaming where Client IPs change
origins = *
credentials = true
# Explicit headers required for Obsidian LiveSync Plugin PUT/Auth operations
headers = accept, authorization, content-type, origin, referer
methods = GET, PUT, POST, HEAD, DELETE
max_age = 3600

```

### 3. Docker Compose

Create the `compose.yaml` file.

!!! warning "Environment Variables"
This compose file uses `${COUCHDB_USER}` and `${COUCHDB_PASSWORD}`. You must either create a `.env` file in the same directory with these values or replace them directly in the file.

```yaml
services:
  couchdb:
    image: couchdb:latest
    container_name: couchdb-for-ols
    user: 5984:5984
    environment:
      - COUCHDB_USER=${COUCHDB_USER}  #Please change as you like.
      - COUCHDB_PASSWORD=${COUCHDB_PASS} #Please change as you like.
    volumes:
      - ./couchdb-data:/opt/couchdb/data
      - ./couchdb-etc:/opt/couchdb/etc/local.d
    ports:
      - 5984:5984
    restart: unless-stopped

```

Start the container:

```bash
docker compose up -d

```

---

## Configuration

### 1. Database Initialization

Once the container is running, the database structure must be initialized.

**Option A: Automatic Script**
Run this command on the host machine:

```bash
curl -s https://raw.githubusercontent.com/vrtmrz/obsidian-livesync/main/utils/couchdb/couchdb-init.sh | bash

```

* **Success:** The output will end with `<-- Configuring CouchDB by REST APIs Done!`.

**Option B: Manual Credential Injection**
If the script fails with errors like `ERROR: Hostname missing` (common in some Docker environments), run the command with specific credentials injected:

```bash
curl -s https://raw.githubusercontent.com/vrtmrz/obsidian-livesync/main/utils/couchdb/couchdb-init.sh | hostname=http://<YOUR SERVER IP>:5984 username=<INSERT USERNAME HERE> password=<INSERT PASSWORD HERE> bash

```

### 2. Generate Setup URI

The easiest way to configure devices is to generate a "Setup URI" that contains all connection strings and encryption keys. This can be run using `deno` on a desktop or server.

```bash
# Set variables
export hostname=https://you.tailscale.ip:5984 # Use Tailscale IP
export database=obsidiannotes 
export passphrase=dfsapkdjaskdjasdas # This is for encryption
export username=johndoe
export password=abc123 # Matches COUCHDB_PASSWORD in compose.yaml

# Run generator
deno run -A https://raw.githubusercontent.com/vrtmrz/obsidian-livesync/main/utils/flyio/generate_setupuri.ts

```

**Output:**

```text
obsidian://setuplivesync?settings=%5B%22tm2DpsOE74nJAryprZO2M93wF%2Fvg.......

Your passphrase of Setup-URI is: patient-haze
This passphrase is never shown again, so please note it in a safe place.

```

* **Save the Setup URI:** This is needed to configure every device.
* **Save the Passphrase:** This is needed to decrypt data on new devices.

---

## Client Setup

!!! danger "CRITICAL WARNING: Customization Sync"
    During setup, the plugin may ask to enable **"Customization Sync"** or "Configuration Sync" (plugins, themes, settings).
    **DO NOT ENABLE THIS.**
    Syncing configuration files between different platforms (e.g., Windows vs. Android) causes boot loops, UI corruption, and file path conflicts. **Decline all pop-ups** related to syncing hidden folders (`.obsidian`). Only sync markdown notes and media. 
    When I used this originally it was a **fucking nightmare** to get things syncing correctly, so I'd avoid it. 


### Network Configuration (Tailscale)

When connecting via mobile devices over Tailscale, mobile OSs often block cleartext HTTP traffic to hostnames.

* **Do NOT use:** `http://my-server-name:5984`
* **USE:** `http://100.x.y.z:5984` (The specific Tailscale IP of the server).

### Android Optimization

To prevent "Silent Drift" (where sync stops quietly in the background):

1. **Battery:** Set the Obsidian app to **Unrestricted** / No Optimization in Android settings.
2. **Plugin Settings:** In Obsidian LiveSync settings, switch the sync mode to **"Periodic"** (30s) or **"Adaptive"**.

### Primary Device Setup (First Time)

Perform these steps on the **first** device to initialize the database structure.

1. In Obsidian, open the LiveSync plugin.
2. Select **"I am setting this up for the first time"**.
3. Paste the **Setup URI** generated earlier.
4. Select **"Restart and initialize server"**.
5. **Confirmation:** Select **YES** on all overwrite prompts.
6. **Remote Config Fail:** If you see "Fetch remote config failed", select **"Skip and Proceed"** (this is normal for an empty database).
7. **Send Chunks:** Select **NO**.
8. **Config Doctor:** Select **NO** on all prompts.
9. **Enable:** Under Sync Settings, enable the LiveSync preset configuration.

### Adding Additional Devices

Perform these steps for additional phones, laptops, or tablets.

1. Open the LiveSync plugin.
2. Paste the **Setup URI** and enter the **Passphrase** (`patient-haze` in the example above).
3. Select **"Restart and fetch data"**.
4. **Reset Sync:** Choose **Merge** or **Discard Local** depending on preference.
5. **Optional Features:** Select **OK**.
6. **Config Doctor:** Select **NO** on all prompts.
7. **Enable:** Under Sync Settings, enable the LiveSync preset configuration.



---

## Maintenance

### Updating CouchDB

To update the database version, pull the latest image and restart the container.

```bash
docker compose pull
docker compose up -d

```

### Troubleshooting

* **Permission Denied:** If the container loops on startup, check that the `./couchdb-data` folder is owned by `5984:5984`.
* **CORS Errors:** If connection fails from a new device, ensure `origins = *` is set in the `docker.ini`.

---
