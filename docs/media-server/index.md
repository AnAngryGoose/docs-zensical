---
icon: lucide/play
title: Media Server
---

# Media Server 

This section covers how to setup and configure a media server. This covers running Plex, as well as downloaders, VPN, and arr stack integration. This is entirely run via a single docker compose file. The file is included below. 

Further configuration of most of these services is covered in depth here: [TRaSH-Guides](https://trash-guides.info/). 


**This setup includes:**

<div class="grid cards" markdown>


-    :simple-plex:{ .lg .middle } __[**Plex Media Server**](https://www.plex.tv/media-server-downloads/)__ 

    ---

    Media Player and streaming server.

-    :simple-jellyfin:{ .lg .middle } __[**Jellyfin**](https://jellyfin.org/)__ 

    ---

    Open-source media server alternative to Plex.

-    :simple-qbittorrent:{ .lg .middle } __[**qBittorrent**](https://www.qbittorrent.org/)__ 

    ---

    Torrent client for downloading media.

-    :simple-qbittorrent:{ .lg .middle } __[**Qbit Manage**](https://github.com/StuffAnThings/qbit_manage)__ 

    ---

    Web interface for managing qBittorrent.

-    :material-vpn:{ .lg .middle } __[**Gluetun**](https://github.com/qdm12/gluetun)__ 

    ---

    VPN client for secure torrenting.

-    :simple-radarr:{ .lg .middle } __[**Radarr**](https://radarr.video/)__

    ---

    Movie downloader and organizer.

-    :simple-sonarr:{ .lg .middle } __[**Sonarr**](https://sonarr.tv/)__ 

    ---

    TV show downloader and organizer.

-   :lucide-paw-print:{ .lg .middle } __[**Prowlarr**](https://prowlarr.com/)__ 

    ---

    Indexer manager for Radarr and Sonarr.

-    :lucide-list:{ .lg .middle } __[**Seerr**](https://seerr.dev/)__ 

    ---

    Media request and management interface. Allows for autodownload via watchlists.
</div>

## Docker Compose

!!! note

    No configuration change should be needed to the compose file. All configuration should be able to be accomplished via the `.env` file. Adjust those settings as needed and then run `docker compose up -d` to start the media server stack.


### `compose.yaml`

```yaml
services:

### ---------------------------------- ###
### ---------- Streaming ------------- ###
### ---------------------------------- ###

### --- Plex Media Server --- ###
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    network_mode: host
    environment:
      - PUID=${PUID}
      - PGID=${GUID}
      - TZ=${TZ}
      - VERSION=docker
      - PLEX_CLAIM=${PLEX_CLAIM}
    ports:
      - 32400:32400
    devices:
     - /dev/dri:/dev/dri #Required for plex HW transcoding / QuickSync (Intel 8th gen or newer)
    volumes:
      - ${APPDATA_PATH}/plex/config:/config  # config data - home nvme 
      - ${TORRENT_PATH}:/data/torrents
      - ${APPDATA_PATH}/plex/transcode:/transcode #t ranscode temp data - appdata SSD
    # - /dev/shm:/transcode - # transcode to ram (if available)
      - ${MEDIA_DATA}/media:/data/media # media access only - mergerfs pool
    restart: unless-stopped

 ### -------------------------------- ###
 ### ----------- Arr Stack ---------- ###
 ### -------------------------------- ###  

  #Radarr - used to find movies automatically
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=${PUID}
      - PGID=${GUID}
      - TZ=${TZ}
    volumes:
      - ${APPDATA_PATH}/radarr/config:/config
      - ${MEDIA_DATA}:/data #Access to the entire data
    ports:
      - 7878:7878
    restart: unless-stopped
    
  #Sonarr - used to find tv shows automatically
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=${PUID}
      - PGID=${GUID}
      - TZ=${TZ}
    volumes:
      - ${APPDATA_PATH}/sonarr/config:/config
      - ${MEDIA_DATA}:/data #Access to the entire data
    ports:
      - 8989:8989
    restart: unless-stopped
  
  #Prowlarr - manages your Sonarr, Radarr and download client
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=${PUID}
      - PGID=${GUID}
      - TZ=${TZ}
    volumes:
      - ${APPDATA_PATH}/prowlarr/config:/config
    ports:
      - 9696:9696
    restart: unless-stopped
  
  # Seerr - allows users to request media on their own [Supercedes Overseerr]
  seerr:
    image: ghcr.io/seerr-team/seerr:latest
    init: true
    container_name: seerr
    environment:
      - PUID=${PUID}
      - PGID=${GUID}
      - TZ=${TZ}
      - LOG_LEVEL=debug
      - PORT=5055 # Optional
    volumes:
      - ${APPDATA_PATH}/seerr/config:/app/config
      - ${MEDIA_DATA}:/data #Access to the entire data
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:5055/api/v1/status || exit 1
      start_period: 20s
      timeout: 3s
      interval: 15s
      retries: 3
    ports:
      - 5055:5055
    restart: unless-stopped


### -------------------------------------- ###
### ----------- Download W/ VPN ---------- ###
### -------------------------------------- ###

### --- Qbittorrent: Torrent Client --- ###
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    depends_on:
      gluetun:
        condition: service_healthy
    environment:
      - PUID=${PUID}
      - PGID=${GUID}
      - TZ=${TZ}
      - WEBUI_PORT=8080
      
      # This torrenting port is enabled by default. Gluetun handles this here, so it is disabled. 
      # - TORRENTING_PORT=8694 # Make sure to port forward this port in your router so you can seed more effectively
    volumes:
      - ${MEDIA_DATA}:/data
      - ${APPDATA_PATH}/qbittorrent/config:/config

    restart: unless-stopped
    network_mode: "service:gluetun"

  qbit_manage:
    container_name: qbit_manage
    image: ghcr.io/stuffanthings/qbit_manage:latest
    volumes:
      - ${APPDATA_PATH}/qbit_manage/:/config:rw
      - ${MEDIA_DATA}:/data:rw
      - ${APPDATA_PATH}/qbittorrent/config:/qbittorrent/:ro
    ports:
      - "8084:8080"  # Web API port (when enabled) - Default 8080 changed to 8084 to avoid conflict with qbittorrent's web UI
    environment:
      # Web API Configuration
      - QBT_WEB_SERVER=true     # Set to true to enable web API and web UI
      - QBT_PORT=8080           # Web API port (default: 8080)

      # Scheduler Configuration
      - QBT_RUN=false
      - QBT_SCHEDULE=1440
      - QBT_CONFIG_DIR=/config
      - QBT_LOGFILE=qbit_manage.log

      # Command Flags
      - QBT_RECHECK=false
      - QBT_CAT_UPDATE=false
      - QBT_TAG_UPDATE=false
      - QBT_REM_UNREGISTERED=false
      - QBT_REM_ORPHANED=false
      - QBT_TAG_TRACKER_ERROR=false
      - QBT_TAG_NOHARDLINKS=false
      - QBT_SHARE_LIMITS=false
      - QBT_SKIP_CLEANUP=false
      - QBT_DRY_RUN=false
      - QBT_STARTUP_DELAY=0
      - QBT_SKIP_QB_VERSION_CHECK=false
      - QBT_DEBUG=false
      - QBT_TRACE=false

      # Logging Configuration
      - QBT_LOG_LEVEL=INFO
      - QBT_LOG_SIZE=10
      - QBT_LOG_COUNT=5
      - QBT_DIVIDER==
      - QBT_WIDTH=100
    restart: on-failure:2


### --- Glutun: VPN for Downloading --- ###
# https://www.reddit.com/r/gluetun/comments/1kpbfs2/the_definitive_howto_for_setting_up_protonvpn/
  gluetun:
    image: qmcgaw/gluetun:v3
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    ports:
      - 8080:8080/tcp # qbittorrent
    environment: 
      - TZ=${TZ}
      - UPDATER_PERIOD=24h
      - VPN_SERVICE_PROVIDER=protonvpn
      - VPN_TYPE=${VPN_TYPE}
      - BLOCK_MALICIOUS=off
      - OPENVPN_USER=${OPENVPN_USER}
      - OPENVPN_PASSWORD=${OPENVPN_PASSWORD}
      - OPENVPN_CIPHERS=AES-256-GCM
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - PORT_FORWARD_ONLY=on
      - VPN_PORT_FORWARDING=on
      - VPN_PORT_FORWARDING_UP_COMMAND=/bin/sh -c 'wget -O- --retry-connrefused --post-data "json={\"listen_port\":{{PORTS}}}" http://127.0.0.1:8080/api/v2/app/setPreferences 2>&1'
      - SERVER_COUNTRIES=${SERVER_COUNTRIES}
    volumes:
      - ${APPDATA_PATH}/gluetun/config:/gluetun
    restart: unless-stopped
```

### `.env`

```env
APPDATA_PATH=/path/to/appdata
MEDIA_DATA=/root/path/to/mediapool
TORRENT_PATH=/path/to/torrents

PUID=1000
GUID=1000

TZ=America/Chicago

PLEX_CLAIM=claim-xxxxxx # Optional - Get this from https://www.plex.tv/claim/ to link your plex server to your account automatically on first run.



### GLUETUN VPN SETTINGS ###
# Fill in either the OpenVPN or Wireguard sections. The choice of vpn is made with VPN_TYPE. Choose 'wireguard' or 'openvpn'. The settings for the other vpn type will be ignored. 
# Alter the TZ, MEDIA_DIR, and SERVER_COUNTRIES to your preference. Run 'docker run --rm -v eraseme:/gluetun qmcgaw/gluetun format-servers -protonvpn' to get a list of server countries


# Gluetun config
VPN_TYPE=wireguard #openvpn # Choose 'wireguard' or 'openvpn'. The settings for the other vpn type will be ignored.

SERVER_COUNTRIES=United States

# OpenVPN config
OPENVPN_USER=username+pmp
OPENVPN_PASSWORD=password

# Wireguard config (example key)
WIREGUARD_PRIVATE_KEY=yourprivatekey
```
