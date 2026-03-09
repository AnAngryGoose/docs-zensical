---
icon: simple/docker
title: Docker
---

Running and managing containerized services with Docker Compose.Bascically all the services I run are ran as compose stacks.

<div class="grid cards" markdown>

-   :lucide-download:{ .lg .middle } __[Installation](docker.md)__

    ---

    Installing Docker Engine and the Compose plugin on Debian.

-   :lucide-settings:{ .lg .middle } __[Post-Installation](dockerpostinstall.md)__

    ---

    Non-root access, socket permissions, and initial configuration.

-   :lucide-file-code:{ .lg .middle } __[Compose Reference](compose-reference.md)__

    ---

    Anatomy of a compose.yaml — services, environment, volumes, networks, healthchecks, and restart policies.

-   :lucide-network:{ .lg .middle } __[Networks](networks.md)__

    ---

    Bridge, host, and macvlan networks. External networks for cross-stack communication.

-   :lucide-hard-drive:{ .lg .middle } __[Volumes](volumes.md)__

    ---

    Named volumes vs bind mounts, the `/opt/docker` directory convention, and external volumes.

-   :lucide-globe:{ .lg .middle } __[DNS in Containers](dns.md)__

    ---

    The `dns:` key in compose and resolving `.internal` hostnames from containers.

-   :lucide-shield:{ .lg .middle } __[Security](security.md)__

    ---

    Non-root users, `.env` file permissions, read-only mounts, and socket exposure.

-   :lucide-list:{ .lg .middle } __[Cheatsheet](cheatsheet.md)__

    ---

    Command reference, update workflows, volume backups, and environment variable patterns.

</div>