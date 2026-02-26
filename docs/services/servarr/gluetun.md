# Gluetun - ProtonVPN + Qbittorrent

[Gluetun :simple-github: ](https://github.com/qdm12/gluetun)

[Gluetun + ProtonVPN :simple-github: ](https://github.com/qdm12/gluetun-wiki/blob/main/setup/providers/protonvpn.md)

---

Gluetun is a VPN client in a thin Docker container for multiple VPN providers, written in Go, and using OpenVPN or Wireguard, DNS over TLS, with a few proxy servers built-in. 

## Gluetun Configuration for ProtonVPN + QBittorrent

---

[Reddit Guide :simple-reddit: ](https://www.reddit.com/r/gluetun/comments/1kpbfs2/the_definitive_howto_for_setting_up_protonvpn/)

For this I'm using a Wireguard setup instead of OpenVPN.
    * It is lighter, faster, and easier to configure. 

In ProtonVPN webpage: 
    * go into the Downloads section and create a new WireGuard configuration. Select Router, no filtering, and "NAT-PMP (Port Forwarding)". Deselect VPN accelerator. When you click Create, a popup of the config will display. Copy the PrivateKey.
  
Use this `compose.yaml` to setup the gluetun container for ProtonVPN and bind it to QB. The `.env` is below it. No need to configure the `compose.yaml`. It can all be done via the `.env`.

```yaml
services:
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
      - ${MEDIA_DIR}/gluetun/config:/gluetun
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    depends_on:
      gluetun:
        condition: service_healthy
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=${TZ}
      - WEBUI_PORT=8080
    volumes:
      - ${MEDIA_DIR}/qbittorrent/config:/config
      - ${MEDIA_DIR}/qbittorrent/downloads:/downloads
    restart: unless-stopped
    network_mode: "service:gluetun"
```
**.env**

```yaml
# Fill in either the OpenVPN or Wireguard sections. The choice of vpn is made with VPN_TYPE. Choose 'wireguard' or 'openvpn'. The settings for the other vpn type will be ignored. 
# Alter the TZ, MEDIA_DIR, and SERVER_COUNTRIES to your preference. Run 'docker run --rm -v eraseme:/gluetun qmcgaw/gluetun format-servers -protonvpn' to get a list of server countries

# Base config
TZ=Australia/Brisbane
MEDIA_DIR=/media

# Gluetun config
VPN_TYPE=wireguard #openvpn
SERVER_COUNTRIES=Albania,Algeria,Angola,Argentina,Australia,Austria,Azerbaijan

# OpenVPN config
OPENVPN_USER=username+pmp
OPENVPN_PASSWORD=password

# Wireguard config (example key)
WIREGUARD_PRIVATE_KEY=wOEI9rqqbDwnN8/Bpp22sVz48T71vJ4fYmFWujulwUU=
```

Run `docker compose up -d` to start it. 

!!!note
    This WILL fail to set the port on first run. Fix below.

Login to the Qbittorrent WebUI. 
    * Settings > WebUI tab
    * Set user/pass
    * Check 'Bypass authentication for clients on localhost'
    * Save. 
  
Restart the stack, check the container logs, and it should work. 

You can test a download from here https://webtorrent.io/free-torrents.


