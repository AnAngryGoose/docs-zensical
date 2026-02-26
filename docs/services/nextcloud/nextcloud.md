# Nextcloud Storage

[NextCloud Github :simple-github: ](https://github.com/nextcloud/server) 

[Nextcloud linuxserverio image :simple-docker: ](https://hub.docker.com/r/linuxserver/nextcloud)

## Overview

**Nextcloud** is a self-hosted, open-source productivity platform that functions as a private alternative to services like Google Drive or Dropbox. It provides file storage, synchronization, and sharing, but can be expanded via "apps" to include contacts, calendars, office suites, and more.

While the core functionality manages files, the architecture requires a web server, PHP runtime, a database for metadata, and a caching mechanism for performance.

---

## Installation

### Docker Compose

While there are multiple ways to install Nextcloud, this guide uses Docker Compose for easier configuration and maintenance.

!!! note "Image Choice"
    I read a lot of people having issues with the official nextcloud docker image so I used the **[LinuxServer.io](https://hub.docker.com/r/linuxserver/nextcloud)** image. It is apparently generally more stable. 

#### Compose Configuration

Create a `compose.yaml` file. You should also create a `.env` file in the same directory to store your secrets (passwords and users).

```yaml
services:
  nextcloud:
    image: lscr.io/linuxserver/nextcloud:latest
    container_name: nextcloud
    environment:
      - PUID=1000   # Change to your user ID (run 'id' in terminal)
      - PGID=1000   # Change to your group ID
      - TZ=America/Chicago
    volumes:
      - /mnt/appdata/nextcloud/config:/config # Configuration files (appdata ssd)
      - /mnt/storage/nextcloud:/data  # This is where your actual files/photos will live
    ports:
      # - 443:443 # Default HTTPS port (uncomment if you want to use it)
      - 4043:443 
    depends_on:
      - nextcloud_db
      - nextcloud_redis
    restart: unless-stopped

  nextcloud_db:
    image: postgres:15-alpine
    container_name: nextcloud_db
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}  # CHANGE THIS in .env!
      - POSTGRES_DB=${POSTGRES_DB} 
    volumes:
      - /mnt/appdata/nextcloud/db_data:/var/lib/postgresql/data
    restart: unless-stopped

  nextcloud_redis:
    image: redis:alpine
    container_name: nextcloud_redis
    restart: unless-stopped

volumes:
  db_data:
```

---

## Initial Setup

After creating your `compose.yaml` and `.env` files, run `docker compose up -d` to start the containers.

Navigate to `https://your.server.ip:4043` to access the setup wizard.

1.  **Create Admin Account:** Enter your desired username and password.
2.  **Database Configuration:** Click "Storage & Database" to expand the options.
3.  **Select Database:** Choose **PostgreSQL**.

!!! warning "Database Hostname"
    When asked for "Database host", do not use `localhost` or an IP. Use the container name: `nextcloud_db`.
    
    Ensure the user, password, and database name match exactly what you defined in your `.env` file.

---

## Database & Caching

### PostgreSQL

**Why use it:**
By default, Nextcloud can run on SQLite (file-based). However, SQLite performs poorly with concurrent connections and slows down significantly as your file count grows. PostgreSQL is an enterprise-class database that handles concurrent connections efficiently, providing stability for metadata operations.

**How to configure:**
This is handled during the initial setup wizard (see above).

### Redis

**Why use it:**
Redis is an in-memory data structure store used for:
1.  **Memory Caching:** Speeds up the web interface by storing frequently accessed data in RAM.
2.  **Transactional File Locking:** Prevents file corruption when multiple devices (phone, desktop) access the same file simultaneously.

**How to configure:**
The `nextcloud_redis` container is running, but Nextcloud needs to be told to use it.

1.  Locate your `config.php` file (mapped in your volume, e.g., `/mnt/appdata/nextcloud/config/www/nextcloud/config/config.php`).
2.  Add or modify the following lines inside the `$CONFIG = array ( ... );` block:

```php
  'memcache.local' => '\\OC\\Memcache\\APCu',
  'memcache.distributed' => '\\OC\\Memcache\\Redis',
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  'redis' => 
  array (
    'host' => 'nextcloud_redis',
    'port' => 6379,
  ),
```

---

## Performance Tuning

### A. PHP Memory Limit
By default, the container often limits PHP to 512MB, which causes sluggishness or crashes during heavy photo uploads.

**The Fix:**
Set the limit via environment variables in your `compose.yaml`:

```yaml
services:
  nextcloud:
    # ... existing config ...
    environment:
      - PHP_MEMORY_LIMIT=2G
      - PHP_UPLOAD_LIMIT=16G # Adjust based on your needs
```
*Run `docker compose up -d` to apply changes.*

### B. Preview Generator
Nextcloud generates image thumbnails "on the fly" when you open a folder. If you open a folder with 1,000 photos, the server will freeze trying to generate them all at once.

**The Fix:**
Install the **Preview Generator** app to pre-render thumbnails in the background.

1.  **Install App:** Go to **Apps** (top right menu) > Search "Preview Generator" > **Download & Enable**.
2.  **Run Initial Scan:**
    
    !!! tip
        This command can take a long time. It is recommended to run this inside a `screen` or `tmux` session so it doesn't fail if your SSH connection drops.

    ```bash
    docker exec -it nextcloud occ preview:generate-all -vvv
    ```

3.  **Add to Cron:**
    To ensure new photos get thumbnails automatically, open your host crontab (`crontab -e`) and add this line to run every 10 minutes:

    ```bash
    */10 * * * * docker exec -u abc nextcloud occ preview:pre-generate
    ```

---
*Sources:*

* [Nextcloud Database Configuration](https://docs.nextcloud.com/server/latest/admin_manual/configuration_database/linux_database_configuration.html)

* [Nextcloud Caching & Locking](https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/caching_configuration.html)