---
icon: simple/opnsense
title: OPNsense
---

OPNsense is the router and firewall for my network. It handles DHCP, DNS, VLANs, NAT, firewall rules, and Tailscale. Managed switches and APs are TP-Link Omada.

<div class="grid cards" markdown>

-   :lucide-layers:{ .lg .middle } __[DNS Stack](opnsense-dns-stack.md)__

    ---

    AdGuard Home, Unbound, and dnsmasq chained together for filtering, recursive resolution, and local hostname registration.

-   :lucide-network:{ .lg .middle } __[VLANs with Omada](vlan.md)__

    ---

    VLAN setup on OPNsense and TP-Link Omada — interfaces, DHCP, firewall rules, and switch port profiles.

-   :simple-adguard:{ .lg .middle } __[AdGuard Home](opnsenseadguard.md)__

    ---

    Network-wide ad and tracker filtering. Sits in front of Unbound as the client-facing DNS server.

-   :simple-tailscale:{ .lg .middle } __[Tailscale](opnsensetailscale.md)__

    ---

    Subnet routing and split DNS over Tailscale, with OPNsense as the exit node.

-   :lucide-box:{ .lg .middle } __[Macvlan](macvlan.md)__

    ---

    Giving containers their own IP and MAC address so they appear as distinct devices on the network.

</div>