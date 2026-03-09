---
icon: lucide/network
title: Networking
---

Covers the full network setup for what I use — firewall, VLANs, DNS, reverse proxy, and remote access. Built on OPNsense with TP-Link Omada switching and a layered DNS stack.

<div class="grid cards" markdown>

-   :simple-opnsense:{ .lg .middle } __[OPNsense](opnsense/index.md)__

    ---

    DNS stack, VLANs, AdGuard Home, Tailscale, and macvlan configuration.

-   :simple-nginxproxymanager:{ .lg .middle } __[NGINX Proxy Manager](npm/index.md)__

    ---

    Internal reverse proxy for vanity URLs. Not in the external traffic path.

-   :simple-cloudflare:{ .lg .middle } __[Cloudflare](cloudflare/index.md)__

    ---

    Cloudflare Tunnels for external access without port forwarding.

</div>

---

*(existing reference content below — DNS stack diagram, VLAN table, port table, etc.)*