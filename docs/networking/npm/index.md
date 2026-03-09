---
icon: simple/nginxproxymanager
title: NGINX Proxy Manager
---

NPM handles internal reverse proxying only — mapping `service.domain.internal` hostnames to running containers. External traffic goes through Cloudflare Tunnels, not NPM.

<div class="grid cards" markdown>

-   :simple-nginxproxymanager:{ .lg .middle } __[Overview](npm.md)__

    ---

    Installation, initial setup, and configuring proxy hosts with SSL.

-   :simple-pihole:{ .lg .middle } __[NPM + Pi-hole](npm-pihole.md)__

    ---

    Running NPM alongside Pi-hole without port conflicts.

</div>