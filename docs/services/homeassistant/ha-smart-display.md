# Voice Assistant & Smart Display

## Overview

Self-hosted Echo Show replacement using a Samsung Galaxy Tab A8 as an always-on smart display with fully local voice assistant, Spotify control, and calling.

**Architecture:**

```
Tab A8 (Fully Kiosk Browser + HA Companion App)
  |
  | wake word detected on-device (microWakeWord)
  | audio sent to ops-01 after wake word
  v
ops-01 (10.10.30.40)
  ├── Home Assistant (host network, port 8123)
  ├── Wyoming Whisper (STT, port 10300)
  ├── Wyoming Piper (TTS, port 10200)
  └── Wyoming openWakeWord (port 10400, for future ESP32 satellites)
```

The HA Companion App (2026.3+) handles wake word detection on-device using microWakeWord. No audio leaves the tablet until after the wake word is detected. The openWakeWord container is for future ESP32 voice satellites only.

---

## Phase 1: Voice Pipeline Containers

### Directory Setup

```bash
sudo mkdir -p /opt/docker/wyoming/{whisper-data,piper-data,openwakeword-data,openwakeword-custom}
sudo chown -R goose:goose /opt/docker/wyoming
```

### Compose File

`/opt/docker/wyoming/compose.yaml`

```yaml
services:
  whisper:
    container_name: whisper
    image: rhasspy/wyoming-whisper
    restart: unless-stopped
    ports:
      - "10300:10300"
    volumes:
      - ./whisper-data:/data
    command:
      - "--model"
      - "small-int8"
      - "--language"
      - "en"
    dns:
      - 10.10.30.1

  piper:
    container_name: piper
    image: rhasspy/wyoming-piper
    restart: unless-stopped
    ports:
      - "10200:10200"
    volumes:
      - ./piper-data:/data
    command:
      - "--voice"
      - "en_US-lessac-medium"
    dns:
      - 10.10.30.1

  openwakeword:
    container_name: openwakeword
    image: rhasspy/wyoming-openwakeword
    restart: unless-stopped
    ports:
      - "10400:10400"
    volumes:
      - ./openwakeword-data:/data
      - ./openwakeword-custom:/custom
    command:
      - "--preload-model"
      - "ok_nabu"
      - "--custom-model-dir"
      - "/custom"
    dns:
      - 10.10.30.1
```

### Deploy

```bash
cd /opt/docker/wyoming
docker compose up -d
docker compose logs -f
```

Models download on first boot (1-3 minutes). Wait for `Ready` in each container's logs.

### Model Notes

- **Whisper `small-int8`**: ~1 second processing on i5-8500T, ~500MB RAM. Drop to `tiny-int8` if too slow, bump to `medium-int8` if accuracy is poor.
- **Piper `en_US-lessac-medium`**: Best general English voice. Preview others at [rhasspy.github.io/piper-samples](https://rhasspy.github.io/piper-samples/).
- **openWakeWord**: Built-in words include `ok_nabu`, `hey_jarvis`, `hey_mycroft`. Only used by ESP32 satellites, not the tablet.

To change a model, edit the command in `compose.yaml` and run `docker compose up -d`. The new model downloads automatically.

---

## Phase 2: Connect to Home Assistant

### Add Wyoming Integrations

Add three separate Wyoming integrations, one for each container. Use ops-01's VLAN IP (not localhost).

1. **Settings > Devices & Services > + Add Integration > Wyoming Protocol**
2. Host: `10.10.30.40`, Port: `10300` (detected as Whisper STT)
3. Repeat: Host: `10.10.30.40`, Port: `10200` (detected as Piper TTS)
4. Repeat: Host: `10.10.30.40`, Port: `10400` (detected as openWakeWord)

!!! warning "Use the actual IP, not localhost"
    HA runs with `network_mode: host` but using `127.0.0.1` can fail depending on Docker's loopback handling. Always use the VLAN IP.

### Create Assist Pipeline

1. **Settings > Voice Assistants > + Add Assistant**
2. Name: `Vanth`
3. Speech-to-text: faster-whisper
4. Text-to-speech: piper
5. Wake word: hey_jarvis (or ok_nabu)
6. Conversation agent: Home Assistant (default)

### Expose Entities

Assist can only control entities explicitly exposed to it.

1. **Settings > Voice Assistants > Expose** tab
2. Toggle on entities you want voice-controllable
3. Set clear aliases for anything with cryptic names
4. Assign entities to **Areas** (Settings > Areas & Zones) for room-based commands like "turn off the lights in the kitchen"

---

## Phase 3: Tablet Setup

### Factory Reset & Debloat

Factory reset the Tab A8 first. After initial Google sign-in, debloat via ADB before installing anything else.

From any machine with ADB (e.g., kupier):

```bash
sudo apt install android-tools-adb
```

Enable Developer Options on tablet (Settings > About tablet > tap Build number 7 times), then enable USB debugging. Connect via USB.

```bash
adb devices   # approve prompt on tablet
```

Remove Samsung bloatware:

```bash
adb shell pm uninstall -k --user 0 com.samsung.android.app.spage
adb shell pm uninstall -k --user 0 com.samsung.android.bixby.agent
adb shell pm uninstall -k --user 0 com.samsung.android.bixby.service
adb shell pm uninstall -k --user 0 com.samsung.android.visionintelligence
adb shell pm uninstall -k --user 0 com.samsung.android.game.gamehome
adb shell pm uninstall -k --user 0 com.samsung.android.game.gametools
adb shell pm uninstall -k --user 0 com.samsung.android.app.tips
adb shell pm uninstall -k --user 0 com.samsung.android.mobileservice
adb shell pm uninstall -k --user 0 com.samsung.android.app.watchmanagerstub
adb shell pm uninstall -k --user 0 com.samsung.android.ardrawing
adb shell pm uninstall -k --user 0 com.samsung.android.aremoji
adb shell pm uninstall -k --user 0 com.samsung.android.arzone
adb shell pm uninstall -k --user 0 com.samsung.android.app.dressroom
adb shell pm uninstall -k --user 0 com.samsung.android.spay
adb shell pm uninstall -k --user 0 com.samsung.android.spayfw
adb shell pm uninstall -k --user 0 com.samsung.android.app.sbrowseredge
adb shell pm uninstall -k --user 0 com.samsung.android.samsungpass
adb shell pm uninstall -k --user 0 com.samsung.android.samsungpassautofill
adb shell pm uninstall -k --user 0 com.sec.android.app.samsungapps
adb shell pm uninstall -k --user 0 com.samsung.android.themestore
adb shell pm uninstall -k --user 0 com.samsung.android.app.reminder
adb shell pm uninstall -k --user 0 com.samsung.android.calendar
adb shell pm uninstall -k --user 0 com.samsung.android.email.provider
adb shell pm uninstall -k --user 0 com.samsung.android.app.notes
adb shell pm uninstall -k --user 0 com.microsoft.office.officehubrow
adb shell pm uninstall -k --user 0 com.microsoft.skydrive
adb shell pm uninstall -k --user 0 com.microsoft.appmanager
adb shell pm uninstall -k --user 0 com.linkedin.android
adb shell pm uninstall -k --user 0 com.facebook.katana
adb shell pm uninstall -k --user 0 com.facebook.appmanager
adb shell pm uninstall -k --user 0 com.facebook.system
adb shell pm uninstall -k --user 0 com.netflix.mediaclient
adb shell pm uninstall -k --user 0 com.samsung.android.forest
adb shell pm uninstall -k --user 0 com.samsung.android.app.social
```

To find anything else: `adb shell pm list packages -f | grep samsung`

To restore if needed: `adb shell cmd package install-existing <package.name>`

!!! danger "Do NOT remove"
    `com.samsung.android.incallui`, `com.samsung.android.telecom`, `com.samsung.android.providers.contacts`, `com.google.android.gms`, `com.google.android.gsf`, `com.android.systemui`

### Install HA Companion App

1. Install from Play Store
2. Open, enter HA URL: `http://ops-01.internal:8123` (or direct IP)
3. Grant all permissions (microphone, camera, location, notifications)
4. Configure wake word: **Settings > Companion app > Assist for Android**
5. Tap **Set as default**, set Home Assistant as default digital assistant
6. Enable **Wake word detection**, select "Hey Jarvis"
7. Select the Assist pipeline you created ("Vanth")

!!! warning "Samsung battery optimization will kill wake word detection"
    - Settings > Apps > Home Assistant > Battery > **Unrestricted**
    - Settings > Device care > Battery > Background usage limits > add HA to **Never sleeping**
    - Consider disabling Adaptive battery entirely since this tablet is always plugged in

### Install Fully Kiosk Browser

1. Install from Play Store (~$9 for Plus license to remove watermark)
2. Open, enter start URL: `http://ops-01.internal:8123/tablet/0`
3. Swipe from left edge to open settings menu
4. Configure:
    - **Web Content Settings > Autoplay Videos:** ON
    - **Web Content Settings > Enable Microphone Access:** OFF (critical -- prevents mic conflict with Companion App)
    - **Device Management > Keep Screen On:** ON
    - **Motion Detection > Enable:** ON (use camera, not microphone)
    - **Motion Detection > Turn Screen On on Motion:** ON
    - **Remote Administration > Enable:** ON, set a password
    - **Kiosk Mode > Enable Kiosk Mode:** ON
    - **Kiosk Mode > Kiosk Exit Gesture:** set a gesture you'll remember
    - **Kiosk Mode > App Whitelist:** add one per line:
      ```
      io.homeassistant.companion.android
      com.google.android.apps.googlevoice
      com.facebook.orca
      ```
    - **Kiosk Mode > Applications to Run on Start in Background:**
      ```
      io.homeassistant.companion.android
      ```

!!! warning "Microphone conflict"
    Fully Kiosk and the Companion App cannot use the mic simultaneously. Keep **Enable Microphone Access OFF** in Fully Kiosk and use camera-based motion detection, not microphone-based. This lets the Companion App hold the mic for wake word detection.

### Add Fully Kiosk Integration to HA

1. **Settings > Devices & Services > + Add Integration > Fully Kiosk Browser**
2. Enter the tablet's IP and Remote Admin password
3. Uncheck "Verify SSL"

---

## Phase 4: Dashboard

Create a dedicated tablet dashboard: **Settings > Dashboards > + Add Dashboard**, name "Tablet", URL `tablet`.

### Greeting Card

```yaml
type: markdown
content: >
  {% set hour = now().hour %}
  {% if hour < 12 %}
  <h1 style="font-size: 2.5em; font-weight: 300; margin: 0;">Good morning</h1>
  {% elif hour < 17 %}
  <h1 style="font-size: 2.5em; font-weight: 300; margin: 0;">Good afternoon</h1>
  {% else %}
  <h1 style="font-size: 2.5em; font-weight: 300; margin: 0;">Good evening</h1>
  {% endif %}
  <h3 style="font-size: 1.2em; font-weight: 300; opacity: 0.6; margin: 0;">{{ now().strftime('%A, %B %-d') }}</h3>
```

!!! note
    HA's markdown renderer strips inline styles from `<p>` tags but allows them on headings. Use `<h1>`, `<h3>`, etc.

### Weather

Add a Weather Forecast card. If no weather integration exists: **Settings > Integrations > + Add > Met.no** (no API key needed).

### Timer

Create a timer helper: **Settings > Devices & Services > Helpers > + Create Helper > Timer**. Name it, set a default duration.

Add to dashboard:

```yaml
type: tile
entity: timer.timer
```

Tap the card to start. Voice command "Hey Jarvis, set a timer for 5 minutes" works natively via Assist (separate from this entity).

### Media / Spotify

See Phase 5. Once configured, add:

```yaml
type: media-control
entity: media_player.spotify_YOUR_USERNAME
```

### Call Buttons

See Phase 6. Add Button cards for each contact.

---

## Phase 5: Spotify

### Official Integration

1. Go to [developer.spotify.com/dashboard](https://developer.spotify.com/dashboard), log in with Spotify Premium
2. Create App:
    - Redirect URI: `http://ops-01.internal:8123/auth/external/callback`
    - Select "Web API"
3. Copy Client ID and Client Secret
4. In HA: **Settings > Integrations > + Add > Spotify**
5. Enter credentials, authenticate

!!! warning "Redirect URI must exactly match your HA URL"
    IP vs hostname mismatch causes auth failures.

### Spotcast (for voice-triggered playback)

The official integration cannot start playback on idle devices. Spotcast works around this.

1. Install HACS if not already installed ([hacs.xyz](https://hacs.xyz/docs/use/download/download/))
2. HACS > Integrations > search "Spotcast" > install
3. Get Spotify cookies:
    - Open incognito window, go to [open.spotify.com](https://open.spotify.com), log in
    - DevTools (F12) > Application > Cookies > open.spotify.com
    - Copy `sp_dc` and `sp_key` values
    - **Close window WITHOUT logging out**
4. Add to `configuration.yaml`:

```yaml
spotcast:
  sp_dc: "your_sp_dc_value"
  sp_key: "your_sp_key_value"
```

5. Restart HA

!!! note
    Spotify cookies expire periodically. Re-extract when playback stops working.

### Tablet as Spotify Connect Device

Install the Spotify app on the Tab A8 and log in. The tablet appears as a Spotify Connect target for playback through its speakers.

---

## Phase 6: Calling

### Apps on Tablet

- **Google Voice**: PSTN calls (real phone numbers). Set up at [voice.google.com](https://voice.google.com). In app settings, set calls to "Prefer Wi-Fi and mobile data".
- **Facebook Messenger**: Video calls to contacts on Messenger.

Both apps are whitelisted in Fully Kiosk (done in Phase 3). Incoming calls overlay the kiosk, and Fully Kiosk returns to the dashboard after the call ends.

### HA Scripts

Create scripts in **Settings > Automations & Scenes > Scripts > + Add Script**.

**Call Mom (Google Voice):**

```yaml
alias: "Call Mom"
sequence:
  - action: fully_kiosk.load_url
    target:
      device_id: YOUR_TABLET_DEVICE_ID
    data:
      url: "tel:+15551234567"
```

**Call Dad (Google Voice):**

```yaml
alias: "Call Dad"
sequence:
  - action: fully_kiosk.load_url
    target:
      device_id: YOUR_TABLET_DEVICE_ID
    data:
      url: "tel:+15559876543"
```

**Call Grandpa (Messenger):**

```yaml
alias: "Call Grandpa"
sequence:
  - action: fully_kiosk.start_application
    target:
      device_id: YOUR_TABLET_DEVICE_ID
    data:
      application: "com.facebook.orca"
```

Find `YOUR_TABLET_DEVICE_ID` by creating the script via the visual editor and selecting the tablet as the target device.

### Dashboard Buttons

```yaml
type: button
name: "Call Mom"
icon: mdi:phone
icon_height: 80px
tap_action:
  action: perform-action
  perform_action: script.call_mom
show_state: false
```

```yaml
type: button
name: "Call Grandpa"
icon: mdi:video
icon_height: 80px
tap_action:
  action: perform-action
  perform_action: script.call_grandpa
show_state: false
```

Use `mdi:phone` for voice calls, `mdi:video` for Messenger video calls.

### Voice-Triggered Calling

Create `config/custom_sentences/en/call_contacts.yaml`:

```yaml
language: "en"
intents:
  CallGrandpa:
    data:
      - sentences:
          - "call grandpa"
          - "video call grandpa"
  CallMom:
    data:
      - sentences:
          - "call mom"
          - "call mommy"
  CallDad:
    data:
      - sentences:
          - "call dad"
          - "call daddy"
```

Add to `configuration.yaml`:

```yaml
intent_script:
  CallGrandpa:
    speech:
      text: "Calling grandpa now"
    action:
      - action: fully_kiosk.start_application
        target:
          device_id: YOUR_TABLET_DEVICE_ID
        data:
          application: "com.facebook.orca"
  CallMom:
    speech:
      text: "Calling mom"
    action:
      - action: fully_kiosk.load_url
        target:
          device_id: YOUR_TABLET_DEVICE_ID
        data:
          url: "tel:+15551234567"
  CallDad:
    speech:
      text: "Calling dad"
    action:
      - action: fully_kiosk.load_url
        target:
          device_id: YOUR_TABLET_DEVICE_ID
        data:
          url: "tel:+15559876543"
```

Restart HA. "Hey Jarvis, call mom" now works.

!!! danger "Do not rely on this tablet for 911"
    Google Voice E911 is unreliable. Keep a cell phone in the house -- even a deactivated phone with no service can dial 911.

---

## HACS Recommendations

- **card-mod**: Custom CSS styling on any card (remove backgrounds, etc.)
- **Mushroom Cards**: Clean, modern, touch-friendly cards
- **Mini Media Player**: Better Spotify/media card than default
- **Browser Mod**: HA control of the tablet's browser (pop-up camera feeds, navigate views)
- **Spotcast**: Voice-triggered Spotify playback (covered in Phase 5)

---

## Future: ESP32 Voice Satellites

Adding an M5Stack ATOM Echo (~$13) or ESP32-S3-BOX as a dedicated voice satellite:

- Satellite streams audio to HA
- openWakeWord on ops-01 handles wake word detection
- Whisper transcribes, Piper responds
- Audio plays through the satellite speaker

The Wyoming infrastructure deployed here supports this with no changes.

---

## Smart Switches

For integrating existing lights with HA/voice, replace dumb switches with Zigbee smart switches.

**Recommended: Inovelli Blue Series (VZM31-SN)**

- Zigbee 3.0, full Zigbee2MQTT support
- Works with and without neutral wire (important for older homes)
- Dimmer + on/off in one device
- Multi-tap scene control for HA automations
- LED notification bar
- 3-way compatible
- ~$40/switch

**Budget alternative: Sonoff ZBMINIL2** (~$10) -- no-neutral relay that fits behind existing switches. Keeps your current switch appearance.

!!! note "Check for neutral wire before buying"
    Turn off breaker, pull a switch out, look for a bundle of white wires nutted together in the back of the box not connected to the switch. If present, you have neutral. Houses built before ~1985 often lack neutral in switch boxes.

---

## Sources

| Component | Docs |
|-----------|------|
| HA Assist | <https://www.home-assistant.io/voice_control/> |
| Local Assist pipeline | <https://www.home-assistant.io/voice_control/voice_remote_local_assistant/> |
| Wyoming integration | <https://www.home-assistant.io/integrations/wyoming/> |
| Wyoming Whisper | <https://github.com/rhasspy/wyoming-faster-whisper> |
| Wyoming Piper | <https://github.com/rhasspy/wyoming-addons> |
| Wyoming openWakeWord | <https://github.com/rhasspy/wyoming-openwakeword> |
| Piper voice samples | <https://rhasspy.github.io/piper-samples/> |
| Assist on Android | <https://www.home-assistant.io/voice_control/android/> |
| Wake word docs | <https://www.home-assistant.io/voice_control/about_wake_word/> |
| Fully Kiosk integration | <https://www.home-assistant.io/integrations/fully_kiosk/> |
| Fully Kiosk app | <https://www.fully-kiosk.com/> |
| Spotify integration | <https://www.home-assistant.io/integrations/spotify/> |
| Spotcast | <https://github.com/fondberg/spotcast> |
| Custom sentences | <https://www.home-assistant.io/voice_control/custom_sentences/> |
| ATOM Echo satellite | <https://www.home-assistant.io/voice_control/thirteen-usd-voice-remote/> |
| HACS | <https://hacs.xyz/docs/use/download/download/> |
| Inovelli Blue | <https://inovelli.com/products/blue-series-smart-2-1-switch-on-off-dimmer/> |
