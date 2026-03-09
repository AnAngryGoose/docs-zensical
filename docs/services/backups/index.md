---
icon: lucide/archive
title: Backups
---

Borgmatic handles all backups across the homelab. All three hosts (prod-deb-01, jupiter, kupier) run Borgmatic on a daily cron schedule with Discord notifications. Borgmatic configuration lives in `utilities/borg.md`.

| Host | Repository | Type |
|---|---|---|
| prod-deb-01 | `/mnt/nas/prod-deb-01` | Samba mount on jupiter |
| jupiter | `/mnt/storage/backups/jupiter` | Local |
| jupiter | `prod-deb-01:/mnt/appdata/backups/jupiter` | SSH |
| kupier | `/mnt/nas/kupier` | Samba mount on jupiter |
| All three | Whatbox SSH | Offsite (pending) |

<div class="grid cards" markdown>

-   :lucide-archive:{ .lg .middle } __[Borgmatic](../../utilities/borg.md)__

    ---

    Borg-based encrypted backups with retention, scheduling, and Discord notifications via Apprise.

-   :lucide-history:{ .lg .middle } __[Backrest](backrest.md)__

    ---

    Restic-based backup UI. Superseded by Borgmatic — kept for reference.

-   :lucide-server:{ .lg .middle } __[Backrest + SSH](backrestssh.md)__

    ---

    Configuring Backrest with SSH remotes for off-host repository storage.

</div>