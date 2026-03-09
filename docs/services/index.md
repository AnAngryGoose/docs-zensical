---
icon: lucide/container
title: Services
---

Self-hosted services I run/ran. Networking related services (reverse proxy, tunnels, DNS) are covered under [Networking](../networking/index.md).

<div class="grid cards" markdown>

-   :lucide-archive:{ .lg .middle } __[Backups](backups/index.md)__

    ---

    Borgmatic on all three hosts, with local and remote Borg repositories.

-   :simple-git:{ .lg .middle } __[git](git/git.md)__

    ---

    Git workflow for Docker compose files and config management. Secret detection and disaster recovery.

-   :simple-homeassistant:{ .lg .middle } __[Home Assistant](homeassisstant/homeassisstant.md)__

    ---

    Home automation hub. Runs on prod-deb-01 with Zigbee2MQTT and Mosquitto.

-   :simple-immich:{ .lg .middle } __[Immich](immich/immich.md)__

    ---

    Self-hosted photo and video library. Runs on jupiter, accessible on local network and over Tailscale.

-   :simple-ollama:{ .lg .middle } __[Ollama & AI](ollama/ollama.md)__

    ---

    Local LLM inference on kupier. Accessed via Open WebUI.

-   :simple-tp-link:{ .lg .middle } __[Omada](omada/omada.md)__

    ---

    TP-Link Omada controller for managing switches and access points.

-   :simple-pihole:{ .lg .middle } __[Pi-hole](pihole/pihole.md)__

    ---

    DNS-based ad blocker. Superseded by AdGuard Home — kept for reference.

-   :simple-tailscale:{ .lg .middle } __[Tailscale](tailscale/tailscale.md)__

    ---

    Tailscale client setup and configuration. OPNsense-level Tailscale is covered under [Networking](../networking/opnsense/opnsensetailscale.md).

-   :lucide-wrench:{ .lg .middle } __[Utilities](utilities/convertx.md)__

    ---

    ConvertX for file conversion, Obsidian LiveSync for cross-device vault sync.

</div>