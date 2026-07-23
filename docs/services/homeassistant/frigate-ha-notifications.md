# Frigate Notifications via Home Assistant

Push notifications to the HA Companion App when Frigate detects objects, with
tap-to-view clips and snapshots.

**Architecture:** Frigate → MQTT → Home Assistant → SgtBatten Blueprint → HA Companion App

## Prerequisites

| Component | Minimum Version | Notes |
|---|---|---|
| Frigate | 0.14.0+ | Running with MQTT enabled |
| Frigate HACS Integration | 5.7.0+ | Installed via HACS |
| Home Assistant | 2024.11.0+ | — |
| Mosquitto MQTT Broker | Any | Connected to both HA and Frigate |
| HA Companion App | Android or iOS | Installed and connected to HA |

## Step 1: Verify MQTT Connectivity

Frigate and Home Assistant must both be connected to the same MQTT broker. This
is the most common failure point.

### Frigate Config

Ensure `mqtt` is enabled in your Frigate config with the broker IP pointing to
your Mosquitto instance:

```yaml
mqtt:
  enabled: true
  host: 10.10.30.40    # IP of your Mosquitto broker
  port: 1883
  topic_prefix: frigate
  client_id: frigate
```

!!! warning "MQTT Authentication"

    If your Mosquitto broker requires authentication (`password_file` in
    `mosquitto.conf`), you **must** include `user` and `password` fields here.
    Frigate will silently fail to connect without them.

    If your broker has `allow_anonymous true`, no credentials are needed.

### Verify the Connection

Check Frigate's logs for MQTT status:

```bash
docker logs frigate 2>&1 | grep -i mqtt
```

Healthy output shows a successful connection. Repeated `MQTT disconnected`
errors indicate a problem — see [Troubleshooting](#troubleshooting).

On the HA side, verify at **Settings → Devices & Services → MQTT** that the
integration shows **Connected**. You can also subscribe to `frigate/#` under
**Developer Tools → MQTT → Listen to a topic** and confirm `frigate/stats`
messages arrive.

## Step 2: Disable Frigate TLS

Frigate enables TLS with a self-signed certificate by default on port 8971.
Home Assistant cannot verify self-signed certs and will return **502 Bad
Gateway** errors when proxying clip and snapshot requests.

Add this to your Frigate config:

```yaml
tls:
  enabled: false
```

Restart Frigate after adding this.

!!! note "Reference"

    [Frigate TLS Documentation](https://docs.frigate.video/configuration/tls) —
    *"You will likely need to set your reverse proxy to allow self signed
    certificates or you can disable TLS in Frigate's config."*

## Step 3: Enable the Notification Proxy

The Frigate HACS integration includes an unauthenticated API proxy that serves
thumbnails, snapshots, and clips to your phone's push notifications. Without
this, notification images will not load.

1. Enable **Advanced Mode** in HA: click your user profile icon (bottom-left
   sidebar) → toggle **Advanced Mode** on.
2. Go to **Settings → Devices & Services → Frigate → Configure** (gear icon).
3. Check **"Enable the unauthenticated notification event proxy"**.
4. Save.

The proxy exposes these endpoints under your HA URL:

```
/api/frigate/notifications/<event-id>/thumbnail.jpg
/api/frigate/notifications/<event-id>/snapshot.jpg
/api/frigate/notifications/<event-id>/clip.mp4
```

## Step 4: Enable Recording and Snapshots

Both must be enabled in your Frigate config for clips and snapshots to be
available in notifications:

```yaml
record:
  enabled: true

snapshots:
  enabled: true
```

These can be set globally or per-camera.

## Step 5: Import the SgtBatten Blueprint

In Home Assistant:

1. Go to **Settings → Automations & Scenes → Blueprints → Import Blueprint**.
2. Paste the URL:

    ```
    https://github.com/SgtBatten/HA_blueprints/blob/main/Frigate_Camera_Notifications/Stable.yaml
    ```

3. Click **Preview Blueprint → Import Blueprint**.

!!! info "Source"

    [SgtBatten/HA_blueprints on GitHub](https://github.com/SgtBatten/HA_blueprints) —
    this is the blueprint recommended in the
    [official Frigate notification docs](https://docs.frigate.video/guides/ha_notifications/).

## Step 6: Create the Automation

Click **Create Automation** on the imported blueprint and configure:

```yaml
alias: Frigate - Camera Notifications
use_blueprint:
  path: SgtBatten/Stable.yaml
  input:
    camera:
      - camera.cam_03          # Your Frigate camera entity
    notify_device: <device-id> # Selected from dropdown
    base_url: http://10.10.30.40:8123
    title: "{{label}} Detected"
    labels:
      - person
      - car
      - dog
      - cat
```

### Configuration Notes

**`base_url`** — Use your HA's internal URL for local-only notifications. For
notifications that work away from home, use a Cloudflare Tunnel URL (without
Cloudflare Access — see [Remote Access](#remote-access-considerations)).
Include the scheme (`http://` or `https://`), no trailing slash.

**`labels`** — Only include objects that are in your Frigate `objects.track`
list. If an object isn't tracked by Frigate, it will never generate an event
and the filter will never match.

!!! danger "Do NOT set `mqtt_topic`"

    The `mqtt_topic` field is for the MQTT **topic prefix**, not the full topic
    path. The blueprint appends `/events` internally. Setting it to
    `frigate/events` would result in the blueprint listening on
    `frigate/events/events` — which does not exist.

    For a single Frigate instance with the default `topic_prefix: frigate`,
    **leave this field blank**.

### Multiple Devices

To notify more than one phone, create a notification group in
`configuration.yaml`:

```yaml
notify:
  - name: family_phones
    platform: group
    services:
      - service: mobile_app_your_phone
      - service: mobile_app_wifes_phone
```

Then set the **Notification Group** field in the blueprint to `family_phones`
(this overrides the single device selection).

## Step 7: Test

Walk in front of the camera. Within a few seconds you should receive a push
notification with a thumbnail of the detected object. The notification will
update with better images as the event progresses.

Check the automation trace if something fails: **Settings → Automations →
your automation → three dots → Traces**.

## Optional: Frigate Card for Clip Viewing

By default, tapping a notification opens the raw clip URL. For a cleaner
experience within the HA app, install the **Frigate Lovelace Card**.

### Install via HACS

Search for **Frigate Card** in HACS (by dermotduffy). Install and restart HA.

Source: [dermotduffy/frigate-hass-card](https://github.com/dermotduffy/frigate-hass-card)

### Create a Camera Dashboard

Add a view to your dashboard (e.g., path: `/lovelace/cameras`) with the Frigate
Card:

```yaml
type: custom:frigate-card
cameras:
  - camera_entity: camera.cam_03
    live_provider: go2rtc
menu:
  buttons:
    clips:
      enabled: true
    snapshots:
      enabled: true
    timeline:
      enabled: true
```

### Set the Blueprint Tap Action

In your automation, set **Tap Action URL** to:

```
/lovelace/cameras
```

Tapping the notification now opens the HA app directly to the camera dashboard.

## Remote Access Considerations

Push notification **delivery** always works regardless of network — notifications
route through Google/Apple push services. The issue is loading **images and
clips** when away from home, since those are served from your HA instance.

**Local only (default):** Use your internal HA URL as `base_url`. Notifications
arrive everywhere, but images/clips only load on home WiFi.

**Cloudflare Tunnel (no Access):** Use your tunnel URL as `base_url`. Images
and clips load from anywhere. Do **not** use Cloudflare Access — the HA
Companion App does not support authenticating reverse proxies and will fail
with "Invalid login session" errors.

**Cloudflare Tunnel with Access:** Not compatible with the HA Companion App.
The app's background processes (sensors, websocket, notification image fetching)
cannot handle the Access authentication layer.

## Troubleshooting

### MQTT Disconnects

Repeated `MQTT disconnected` errors in Frigate logs — check for:

- **Client ID collision:** If two MQTT clients use the same `client_id`, the
  broker kicks one off each time the other connects. Check Mosquitto logs for
  "already connected" messages.
- **Cross-VLAN firewall rules:** If Frigate and Mosquitto are on different
  subnets, ensure your firewall allows persistent TCP connections on port 1883
  (not just initial connection but sustained keepalive).
- **Network connectivity:** Test from the Frigate container:

    ```bash
    docker exec frigate bash -c \
      'echo > /dev/tcp/<broker-ip>/1883 && echo "OK" || echo "FAIL"'
    ```

### 502 Bad Gateway on Clip/Snapshot

- **TLS not disabled:** Frigate defaults to a self-signed cert on 8971. Add
  `tls: enabled: false` to your Frigate config.
- **Notification proxy not enabled:** Enable Advanced Mode in HA, then check
  the proxy setting under the Frigate integration configuration.

### Notifications Arrive but No Image

- Verify `base_url` is reachable from your phone.
- Confirm the notification proxy is enabled (Step 3).
- Check that `record` and `snapshots` are both enabled in Frigate.

### Automation Never Triggers

- Confirm MQTT is connected on both sides (Frigate logs + HA MQTT integration).
- Verify the camera entity name in the blueprint matches the Frigate camera
  name. The integration auto-creates `camera.cam_03` from Frigate's `cam_03`.
- Ensure detection is enabled for the camera (`detect: enabled: true`).

## References

- [Frigate HA Notifications Guide](https://docs.frigate.video/guides/ha_notifications/)
- [Frigate HA Integration Docs](https://docs.frigate.video/integrations/home-assistant/)
- [Frigate TLS Configuration](https://docs.frigate.video/configuration/tls)
- [SgtBatten Blueprint — GitHub](https://github.com/SgtBatten/HA_blueprints)
- [SgtBatten Blueprint — HA Community](https://community.home-assistant.io/t/frigate-mobile-app-notifications-2-0/559732)
- [Frigate Lovelace Card — GitHub](https://github.com/dermotduffy/frigate-hass-card)