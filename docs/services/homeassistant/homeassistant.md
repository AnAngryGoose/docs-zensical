# Home Assistant & Zigbee Stack

[Home Assistant Github :simple-github: ](https://github.com/home-assistant)

[Z2MQTT Github :simple-github:](https://github.com/Koenkk/zigbee2mqtt)

[Eclipse Mosquitto Github :simple-github:](https://github.com/eclipse-mosquitto/mosquitto)

---
## Overview

This integrates **Home Assistant**, **Zigbee2MQTT (Z2M)**, and an **MQTT Broker (Mosquitto)** to create a local smarthome. 

I am specifically configuring this for the **Sonoff ZBDongle-MG24** (Model "E" or "MG24"), which runs on the Silicon Labs EFR32MG24 chip. This requires the specific `ember` driver in Zigbee2MQTT, unlike the older Texas Instruments-based dongles.

The stack runs via Docker Compose, with Home Assistant running in `host` networking mode for optimal device discovery.

I'm a big fan of the zigbee system as each device acts as a repeater for the other devices. Also, working of seperate protocol, your devices will continue working even without WiFi.

---

## Installation

### 1. Hardware Preparation

**Identify the Dongle**
The Sonoff MG24 uses a different driver protocol than older models. Before configuring software, ensure the dongle is recognized.

1. Plug the dongle into a USB 2.0 port **(or use a USB 2.0 extension cable to avoid USB 3.0 interference).**
2. Run the following to identify your specific device ID:
```bash
ls -l /dev/serial/by-id/

```


3. Copy the output string. It usually looks like `usb-ITEAD_SONOFF_Zigbee_...` or similar. You will need this for your Compose file.

### 2. Directory Structure

Create a persistent directory structure to store your configurations and data.

```bash
mkdir -p ~/homelab/smarthome/{homeassistant,zigbee2mqtt/data,mosquitto/config,mosquitto/data,mosquitto/log}

```

### 3. Docker Compose

Create your `compose.yaml` file in `~/homelab/smarthome/`.

!!! note "Port Mapping"
This configuration maps the Zigbee2MQTT frontend to port **8082** to avoid conflicts, and maps the USB device. Ensure the `devices` section matches the ID you found in Step 1.

```yaml
### --- Smart Home Stack: MQTT Broker, Zigbee2MQTT, Home Assistant --- ###

services:
  # MQTT Broker
  mosquitto:
    container_name: mosquitto
    image: eclipse-mosquitto
    restart: unless-stopped
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log

  # Zigbee2MQTT
  zigbee2mqtt:
    container_name: zigbee2mqtt
    image: koenkk/zigbee2mqtt
    restart: unless-stopped
    depends_on:
      - mosquitto
    volumes:
      - ./zigbee2mqtt/data:/app/data
      - /run/udev:/run/udev:ro
    environment:
      - TZ=America/Chicago # Update to your timezone
    devices:
      # Map the USB dongle directly. UPDATE THIS to match your 'ls -l' output
      - /dev/serial/by-id/usb-youthings_Zigbee_Adapter-if00:/dev/ttyACM0
    ports:
      - "8082:8080" # Web Interface

  # Home Assistant 
  homeassistant:
    container_name: homeassistant
    image: lscr.io/linuxserver/homeassistant:latest
    network_mode: host # Critical for Home Assistant discovery
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Chicago
    volumes:
      - ./homeassistant:/config
    restart: unless-stopped
    depends_on:
      - mosquitto
      - zigbee2mqtt

```

---

## Configuration

You must create the configuration files for Mosquitto and Zigbee2MQTT before starting the containers, or they will likely crash on boot.

### 1. Mosquitto Broker

Create the file: `~/homelab/smarthome/mosquitto/config/mosquitto.conf`

```text
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log

# Listen on all interfaces
listener 1883

# Allow anonymous for internal local network setup
allow_anonymous true

```

### 2. Zigbee2MQTT

Create the file: `~/homelab/smarthome/zigbee2mqtt/data/configuration.yaml`

!!! warning "Driver Selection"
The `adapter: ember` setting is critical for the Sonoff MG24. If you omit this, Z2M will try to use the default driver and fail to start.

```yaml
# Home Assistant Integration
homeassistant: true

# MQTT Settings
mqtt:
  base_topic: zigbee2mqtt
  server: 'mqtt://mosquitto:1883'

# Serial Settings for Sonoff MG24 (Ember Driver)
serial:
  # This path inside the container is mapped from your host in the compose file
  port: /dev/ttyACM0 
  adapter: ember

# Frontend Settings (GUI)
frontend:
  port: 8080

# Zigbee Network
permit_join: false

```

---

## Setup & Device Pairing

Once the files are created, launch the stack:

```bash
docker compose up -d

```

### Pairing Devices (Example: ThirdReality ZL1 Bulb)

1. **Access Z2M:** Open `http://<server-ip>:8082`.
2. **Permit Join:** Click **Permit Join (All)** in the top bar to allow new devices.
3. **Reset Bulb:**
* Screw in the ThirdReality ZL1 bulb.
* **Power Cycle 5 Times:** Turn the switch OFF and ON 5 times quickly (approx. 1 second per toggle).
* *Sequence:* ON -> OFF -> ON -> OFF -> ON -> OFF -> ON -> OFF -> ON.
* *Confirmation:* The bulb will flash a sequence of colors (Warm White -> Cool White -> Red -> Green -> Blue) and stay solid Warm White.


4. **Rename:** Once it appears in the Z2M dashboard, click the blue "Edit" icon. Give it a friendly name (e.g., "Office_Lamp") and ensure "Update Home Assistant entity ID" is checked.

### Connecting to Home Assistant

1. **Access HA:** Open `http://<server-ip>:8123`.
2. **Integrations:** Go to **Settings > Devices & Services**.
3. **Discovery:** HA should auto-discover "MQTT". If not, click **Add Integration > MQTT**.
4. **Broker Settings:**
* **Broker:** `localhost` (Since HA is on the host network).
* **Port:** `1883`.
* **Auth:** Leave blank (I enabled `allow_anonymous`).


5. **Verify:** Your "Office_Lamp" should now appear as a device in Home Assistant.

---

## Maintenance

### Updates

To update the stack, pull the latest images and recreate the containers. Your data is safe in the persistent volumes.

```bash
docker compose pull
docker compose up -d

```

### Troubleshooting

If Zigbee2MQTT fails to start, check the logs immediately:

```bash
docker compose logs zigbee2mqtt

```

* **Error: "Error: Error: No such file or directory..."**: Check your USB mapping in `compose.yaml`. The ID `usb-youthings...` or `usb-ITEAD...` must match exactly what `ls -l /dev/serial/by-id/` shows.
* **Error: "Adapter failed to start"**: Ensure `adapter: ember` is in your `configuration.yaml`.

---
