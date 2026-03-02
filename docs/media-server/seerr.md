---
icon: lucide/list
---

# Seerr

[Seerr Docs](https://docs.seerr.dev/)

**Seerr** is a free and open source software application for managing requests for your media library. It integrates with the media server of your choice: [Jellyfin](https://jellyfin.org/), [Plex](https://plex.tv/), and [Emby](https://emby.media/). In addition, it integrates with your existing services, such as **[Sonarr](https://sonarr.tv/)**, **[Radarr](https://radarr.video/)**.

---

## Docker Installation (recommended)

---

!!!info
    An alternative Docker image is available on Docker Hub for this project. You can find it at [Docker Hub Repository Link](https://hub.docker.com/r/seerr/seerr)

    Our Docker images are available with the following tags:

    -   `latest`: Always points to the most recent stable release.
    -   Version tags (e.g., `v3.0.0`): For specific stable versions.
    -   `develop`: Rolling release/nightly builds for using the latest changes (use with caution).


### Docker Compose

```yaml
services:
  seerr:
    image: ghcr.io/seerr-team/seerr:latest
    init: true
    container_name: seerr
    environment:
      - LOG_LEVEL=debug
      - TZ=Asia/Tashkent
      - PORT=5055 #optional
    ports:
      - 5055:5055
    volumes:
      - /path/to/appdata/config:/app/config
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:5055/api/v1/status || exit 1
      start_period: 20s
      timeout: 3s
      interval: 15s
      retries: 3
    restart: unless-stopped
```

```bash 
# Create the appdata folder
mkdir /path/to/appdata/config
# Chown the folder as the container runs as the `node` user (UID 1000).
chown -R 1000:1000 /path/to/appdata/config
docker compose up -d
```

