---
icon: simple/plex
title: Plex
---

# Plex

Plex is the media server / streaming front-end for the stack — it indexes the
libraries the *arr apps fill and streams them to clients (TV apps, phones, web,
Plex Amp). This page covers the container as it runs here; the **full stack
compose** lives on the [Media Server](index.md#docker-compose) page.

[Plex Support](https://support.plex.tv/) ·
[LinuxServer image docs](https://docs.linuxserver.io/images/docker-plex/) ·
[Get a claim token](https://www.plex.tv/claim/)

---

## Container

Runs from the LinuxServer.io image in **host networking** mode — Plex relies on a
number of ports plus local discovery (GDM/DLNA), and host mode is the
[recommended](https://support.plex.tv/articles/200430283-network/) setup.

```yaml
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    network_mode: host
    environment:
      - PUID=${PUID}
      - PGID=${GUID}
      - TZ=${TZ}
      - VERSION=docker
      - PLEX_CLAIM=${PLEX_CLAIM}   # from https://www.plex.tv/claim (first run only)
    devices:
      - /dev/dri:/dev/dri          # Intel QuickSync HW transcode (8th-gen+)
    volumes:
      - ${APPDATA_PATH}/plex/config:/config
      - ${APPDATA_PATH}/plex/transcode:/transcode
      - ${MEDIA_DATA}/media:/data/media
    restart: unless-stopped
```

!!! note

    `network_mode: host` means the `ports:` block is ignored — Plex binds `32400`
    (and its discovery ports) directly on the host. Keep the media mount **read-only
    to Plex's needs**: Plex only reads media, the *arr apps write it.

## First run & claiming the server

The `PLEX_CLAIM` token links a brand-new server to your account automatically on
first boot.

1. While logged in, open **[plex.tv/claim](https://www.plex.tv/claim/)** and copy
   the `claim-xxxx` token into `PLEX_CLAIM` in your `.env`.
2. `docker compose up -d plex` **within ~4 minutes** — the token expires that fast.
3. Browse to `http://<server-ip>:32400/web`, finish the setup wizard, and add
   libraries pointing at `/data/media/...`.

!!! tip

    The claim token is only needed for the very first launch. Once the server is
    associated with your account you can blank `PLEX_CLAIM` — subsequent restarts
    keep the association from the `/config` volume.

## Libraries & naming

Plex matches files to metadata by **naming convention**. Follow Plex's naming so
matches are clean:

- Movies: `Movie Name (Year)/Movie Name (Year).ext`
- TV: `Show Name (Year)/Season 01/Show Name - S01E01.ext`

Radarr and Sonarr already produce Plex-friendly names — see
[TRaSH Guides](https://trash-guides.info/) for the recommended naming schemes and
quality settings that keep Plex, Radarr, and Sonarr agreeing on file layout.

## Hardware transcoding (QuickSync)

Direct Play (client plays the file as-is) is always best. When a client needs a
different format/bitrate, Plex **transcodes** — and hardware transcoding offloads
that to the Intel iGPU instead of hammering the CPU.

!!! warning "Requires Plex Pass"

    Hardware-accelerated streaming is a **[Plex Pass](https://www.plex.tv/plex-pass/)**
    feature. Without it, transcoding is CPU-only.

- Passing `/dev/dri` into the container exposes the Intel iGPU for **QuickSync**
  (Intel 8th-gen or newer).
- Enable it: **Settings → Transcoder → "Use hardware acceleration when available"**.
- Verify: play a file that forces a transcode and check the Dashboard — the session
  should show **"(hw)"** next to *Transcode*.
- The `/transcode` mount is scratch space for in-progress transcodes; put it on fast
  storage (SSD/NVMe), or map it to `/dev/shm` (RAM) if you have the memory to spare.

## Remote access

- **Built-in:** Settings → Remote Access. Plex can set up a port mapping (UPnP) or
  you forward `32400/tcp` manually.
- **Reverse proxy / tunnel:** front it with a reverse proxy or a Cloudflare Tunnel
  instead of opening the port — see [Networking](../networking/index.md). Set the
  custom access URL under **Settings → Network → Custom server access URLs**.

## Sources

- [Plex Support — main help](https://support.plex.tv/)
- [Plex — Using Hardware-Accelerated Streaming](https://support.plex.tv/articles/115002178853-using-hardware-accelerated-streaming/)
- [LinuxServer.io — docker-plex](https://docs.linuxserver.io/images/docker-plex/)
- [TRaSH Guides](https://trash-guides.info/)
