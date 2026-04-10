# Frigate NVR Setup Guide
## Dahua IPC-T54IR-AS-S3 + GTX 1080 + Home Assistant

**Stack:** jupiter (10.10.30.30) · Docker · Mosquitto on ops-01 (10.10.30.40)  
**Cameras:** 3x Dahua/EmpireTech IPC-T54IR-AS-S3 4MP turret, H.265, PoE  
**Image:** `ghcr.io/blakeblackshear/frigate:stable-tensorrt`

---

## 1. Physical Setup

### HDD for Recordings

```bash
lsblk                          # identify new disk
sudo fdisk /dev/sdX            # g (GPT), n (partition), accept defaults, w (write)
sudo mkfs.ext4 /dev/sdX1
sudo mkdir -p /mnt/frigate
sudo blkid /dev/sdX1           # copy UUID
echo "UUID=<uuid>  /mnt/frigate  ext4  defaults,nofail  0  2" | sudo tee -a /etc/fstab
sudo mount -a
df -h /mnt/frigate             # verify
```

`nofail` is required -- without it jupiter won't boot if the drive is absent.

### Camera Default IP

!!! warning
    Dahua cameras ship with a **static IP of 192.168.1.108** -- they do not DHCP by default. To access them initially (If on different VLAN):

1. Set laptop NIC to `192.168.1.50 / 255.255.255.0`
2. Browse to `http://192.168.1.108`
3. Set admin password (required on first login)
4. Go to **Setup > Network > TCP/IP**, assign static IoT VLAN IP
5. Confirm RTSP enabled: **Setup > Network > Port**, default port 554

Assign cameras sequential statics: `10.10.50.101`, `.102`, `.103`

### PoE Switch (SG2016P)

The SG2016P has 8 PoE+ ports (802.3at/af, 30W per port, 120W total budget). PoE ports are ports 1-8 -- confirm in Omada UI under **Devices > [switch] > PoE**. Each camera draws ~12-15W, well within budget for 3 cameras.

---

## 2. Camera Configuration

### Stream Settings (Camera Web UI)

**Main Stream** (record -- full quality):
- Resolution: 2688x1520
- Framerate: 20fps
- Bitrate: 4096-6144 kbps
- Codec: H.265
- I-frame interval: 40 (2x framerate)

**Sub Stream 2** (detect -- use subtype=2, not subtype=1):
- Resolution: 1280x720
- Framerate: 10fps
- Bitrate: 1024 kbps
- Codec: H.265
- I-frame interval: 20 (2x framerate)

!!! question "**Why Sub Stream 2:**"
     Sub Stream 1 is hard-capped at 704x480 by Dahua firmware -- this is by design for legacy NVR compatibility. Sub Stream 2 allows up to 1280x720 and is referenced with `subtype=2` in the RTSP URL.

!!! question "**Why not max resolution on sub stream:**" 
    Detection runs on every frame of the sub stream. The model runs at 640x640 internally regardless -- higher sub stream resolution adds GPU decode cost with no detection accuracy gain. The main stream captures full resolution for recordings.

### RTSP URL Format

```
Main:  rtsp://admin:password@10.10.50.10x:554/cam/realmonitor?channel=1&subtype=0
Sub2:  rtsp://admin:password@10.10.50.10x:554/cam/realmonitor?channel=1&subtype=2
```

> **Password note:** Special characters (`#`, `$`, `@`) in passwords break RTSP URL parsing. Use alphanumeric-only passwords on the cameras.

### Verify Streams Before Frigate

```bash
ffprobe -v quiet -print_format json -show_streams \
  'rtsp://admin:password@10.10.50.101:554/cam/realmonitor?channel=1&subtype=0'
```

Confirms codec, resolution, framerate. Run for all three cameras on both subtype=0 and subtype=2.

---

## 3. Host Prerequisites (jupiter)

### NVIDIA Driver + Container Toolkit

```bash
# Verify driver
nvidia-smi

# Install desktop driver variant (required for libnvcuvid.so.1)
# The server variant does NOT include video decode libraries
sudo apt install nvidia-driver-535

# Verify library present
ldconfig -p | grep nvcuvid

# Install Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit

# Configure Docker runtime
# WARNING: nvidia-ctk writes to /etc/docker/daemon.json
# Do NOT set default-runtime to nvidia -- use deploy.resources in compose instead
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Verify
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

> **daemon.json warning:** `nvidia-ctk` creates `/etc/docker/daemon.json`. If Docker breaks after running it, check the file -- remove it entirely if it was empty before (`sudo rm /etc/docker/daemon.json && sudo systemctl restart docker`). Use `deploy.resources` in compose instead of setting a default runtime.

> **Server vs desktop driver:** `nvidia-utils-535-server` does not include `libnvcuvid.so.1` which is required for NVDEC hardware video decoding in ffmpeg. Must install `nvidia-driver-535` (desktop variant).

---

## 4. ONNX Detection Model

Frigate 0.16+ does not ship pre-built models. You must provide one.

### Download and Build YOLOv9-m at 640px

```bash
mkdir -p /home/goose/docker/frigate/config/model_cache
cd /home/goose/docker/frigate/config/model_cache

# Download pre-trained weights
wget https://github.com/WongKinYiu/yolov9/releases/download/v0.1/yolov9-m-converted.pt

# Build ONNX export
docker build --no-cache --output . -f- . <<'EOF'
FROM python:3.11 AS build
RUN apt-get update && apt-get install --no-install-recommends -y cmake libgl1 && rm -rf /var/lib/apt/lists/*
COPY --from=ghcr.io/astral-sh/uv:0.10.4 /uv /bin/
WORKDIR /yolov9
ADD https://github.com/WongKinYiu/yolov9.git .
RUN uv pip install --system -r requirements.txt
RUN uv pip install --system onnx==1.18.0 onnxruntime onnx-simplifier==0.4.35
COPY yolov9-m-converted.pt yolov9-m.pt
RUN sed -i "s/ckpt = torch.load(attempt_download(w), map_location='cpu')/ckpt = torch.load(attempt_download(w), map_location='cpu', weights_only=False)/g" models/experimental.py
RUN python3 export.py --weights ./yolov9-m.pt --imgsz 640 --simplify --include onnx
FROM scratch
COPY --from=build /yolov9/yolov9-m.onnx /yolov9-m-640.onnx
EOF
```

Output: `yolov9-m-640.onnx`

> **Why 640px:** The sub stream is 1280x720. Frigate scales it to the model's input size for inference. Building at 320px throws away resolution you paid for -- 640px gives meaningfully better detection accuracy with negligible GPU overhead on a 1080.

> **onnx-simplifier pinned to 0.4.35:** Unpinned (`>=0.4.1`) resolves to 0.5.0 which pulls `onnxsim==0.6.2` which fails to build from source. 0.4.35 is the last version before this dependency was introduced.

> **Model variants:** `t` (tiny, 4.4MB) → `s` (small, 14MB) → `m` (medium, 38MB) → `c` (large, 49MB) → `e` (112MB). For 3 cameras on a GTX 1080, `-m` at 640px is the right balance -- inference ~15-25ms, meaningfully better accuracy than `-s` especially for IR/night footage.

---

## 5. Docker Compose

`/home/goose/docker/frigate/compose.yaml`:

```yaml
services:
  frigate:
    container_name: frigate
    image: ghcr.io/blakeblackshear/frigate:stable-tensorrt
    restart: unless-stopped
    stop_grace_period: 30s
    shm_size: "256mb"
    env_file:
      - .env
    dns:
      - 10.10.30.1
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    volumes:
      - ./config:/config
      - /mnt/frigate:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "8971:8971"
      - "8554:8554"
      - "8555:8555/tcp"
      - "8555:8555/udp"
```

`/home/goose/docker/frigate/.env`:

```bash
FRIGATE_RTSP_PASSWORD=yourpassword
```

```bash
chmod 600 .env
```

> **tmpfs /tmp/cache:** Frigate buffers recording segments in memory before writing to disk. Reduces small random writes to the HDD. 1GB is sufficient for 3 cameras.

> **GPU passthrough:** NVIDIA uses `deploy.resources.reservations.devices` in compose, not `devices:` mappings. The device mappings (`/dev/dri/renderD128` etc.) are for Intel/AMD only.

> **dns:** Required for containers to resolve `.internal` hostnames via AdGuard/Unbound chain.

---

## 6. Frigate Config

`/home/goose/docker/frigate/config/config.yml`:

```yaml
mqtt:
  enabled: true
  host: 10.10.30.40
  port: 1883

database:
  path: /media/frigate/frigate.db

go2rtc:
  streams:
    cam_01:
      - rtsp://admin:{FRIGATE_RTSP_PASSWORD}@10.10.50.101:554/cam/realmonitor?channel=1&subtype=0
    cam_01_sub:
      - rtsp://admin:{FRIGATE_RTSP_PASSWORD}@10.10.50.101:554/cam/realmonitor?channel=1&subtype=2
    cam_02:
      - rtsp://admin:{FRIGATE_RTSP_PASSWORD}@10.10.50.102:554/cam/realmonitor?channel=1&subtype=0
    cam_02_sub:
      - rtsp://admin:{FRIGATE_RTSP_PASSWORD}@10.10.50.102:554/cam/realmonitor?channel=1&subtype=2
    cam_03:
      - rtsp://admin:{FRIGATE_RTSP_PASSWORD}@10.10.50.103:554/cam/realmonitor?channel=1&subtype=0
    cam_03_sub:
      - rtsp://admin:{FRIGATE_RTSP_PASSWORD}@10.10.50.103:554/cam/realmonitor?channel=1&subtype=2

detectors:
  onnx:
    type: onnx
    model:
      path: /config/model_cache/yolov9-m-640.onnx
      model_type: yolo-generic
      width: 640
      height: 640
      input_tensor: nchw
      input_dtype: float
      labelmap_path: /labelmap/coco-80.txt

ffmpeg:
  hwaccel_args: preset-nvidia-h265

cameras:
  cam_01:
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/cam_01
          input_args: preset-rtsp-restream
          roles:
            - record
        - path: rtsp://127.0.0.1:8554/cam_01_sub
          input_args: preset-rtsp-restream
          roles:
            - detect
    detect:
      enabled: true
      width: 1280
      height: 720

  cam_02:
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/cam_02
          input_args: preset-rtsp-restream
          roles:
            - record
        - path: rtsp://127.0.0.1:8554/cam_02_sub
          input_args: preset-rtsp-restream
          roles:
            - detect
    detect:
      enabled: true
      width: 1280
      height: 720

  cam_03:
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/cam_03
          input_args: preset-rtsp-restream
          roles:
            - record
        - path: rtsp://127.0.0.1:8554/cam_03_sub
          input_args: preset-rtsp-restream
          roles:
            - detect
    detect:
      enabled: true
      width: 1280
      height: 720

record:
  enabled: true
  retain:
    days: 7
    mode: all

snapshots:
  enabled: true
  retain:
    default: 7
```

### Start Frigate

```bash
cd /home/goose/docker/frigate
docker compose up -d && docker logs -f frigate
```

First boot takes several minutes -- TensorRT builds an optimized engine for the GTX 1080 from the ONNX model. This is a one-time process per model.

### Get Initial Password

```bash
docker logs frigate | grep -i password
```

Password is only printed on first ever startup. If missed, reset:

```bash
docker exec -it frigate sqlite3 /media/frigate/frigate.db \
  "SELECT username, password FROM users;"
```

UI accessible at `https://10.10.30.30:8971`

---

## 7. Firewall Rules

### IoT Interface (OPNsense)

Order matters -- first match wins.

| # | Action | Source | Destination | Port | Description |
|---|---|---|---|---|---|
| 1 | Pass | IOT net | 10.10.50.1 | 53 | Force DNS to AdGuard |
| 2 | Block | IOT net | PrivateNetworks alias | any | Block IoT to all internal |
| 3 | Pass | IOT net | any | any | IoT internet access |

> Rule 1 from the original setup (Allow IoT cameras → Frigate on jupiter, port any) was identified as unnecessary and overly permissive. OPNsense is a stateful firewall -- return traffic from cameras to Frigate is handled automatically via state table. Only Frigate initiating outbound RTSP needs an explicit rule, which lives on the LAB interface.

### LAB Interface (OPNsense)

| # | Action | Source | Destination | Port | Description |
|---|---|---|---|---|---|
| 1 | Pass | 10.10.30.30 | IOT net | 554 | Frigate → cameras RTSP |
| 2 | Pass | 10.10.30.40/32 | MANAGEMENT net | any | Omada controller → MGMT |
| 3 | Pass | LAB net | 10.10.20.10 | any | Allow LAB to kupier |
| 4 | Block | LAB net | TRUSTED net | any | Block LAB to TRUSTED |
| 5 | Block | LAB net | MANAGEMENT net | any | Block LAB to MANAGEMENT |
| 6 | Pass | LAB net | 10.10.30.1 | 53 | Force DNS to AdGuard |
| 7 | Pass | LAB net | any | any | LAB internet access |

> Force DNS rule must be above the internet access pass rule -- first match wins and a catch-all pass above it would bypass DNS enforcement entirely.

---

## 8. Home Assistant Integration

### Install Frigate Integration (HACS)

The official Frigate integration is HACS-only -- it is not in the built-in HA integration store.

1. In HA: **HACS > Integrations > Search "Frigate" > Install**
2. Restart Home Assistant
3. **Settings > Devices & Services > Add Integration > Frigate**
4. Enter: `http://10.10.30.30:8971`

> Do not use `https://` -- HA integration connects internally and the self-signed cert will cause connection failures.

### What You Get

After integration:
- Camera entities for each cam (`camera.cam_01` etc.) with live view
- Binary sensors for each detected object class per camera
- `sensor.frigate_cam_01_person_count` etc.
- Event entities for alerts and detections
- Full clip and snapshot access from HA

### MQTT

Frigate communicates detections to HA via MQTT (Mosquitto on ops-01). The HA MQTT integration must already be configured and pointing to `10.10.30.40:1883`. Frigate publishes to `frigate/#` topic prefix by default.

Confirm Mosquitto is healthy before troubleshooting HA integration:

```bash
docker exec mosquitto mosquitto_sub -t '#' -v
```

---

## 9. Issues? 

| Issue | Root Cause | Fix |
|---|---|---|
| Camera not in ARP/DHCP | Dahua ships with static 192.168.1.108, not DHCP | Access at 192.168.1.108, set static IoT IP manually |
| RTSP URL 404 / empty ffprobe | Special chars in password break URL parsing | Use alphanumeric-only camera passwords |
| Sub stream capped at 704x480 | Sub Stream 1 is firmware-limited by design | Use Sub Stream 2 (`subtype=2`) |
| libnvcuvid.so.1 missing | Server driver variant excludes video decode libs | Install `nvidia-driver-535` (desktop), not server |
| Docker broke after nvidia-ctk | nvidia-ctk writes daemon.json, conflicts with `-H fd://` | Remove daemon.json, use `deploy.resources` in compose |
| ONNX loading None | Frigate 0.16+ ships no models | Build YOLOv9 ONNX from source or download pre-exported |
| onnx-simplifier build fails | Unpinned resolves to 0.5.0 which pulls broken onnxsim 0.6.2 | Pin to `onnx-simplifier==0.4.35` |
| Firefox crashes on stream | Snap-packaged Firefox sandbox blocks WebSocket | Install Firefox via PPA deb, remove snap version |
| Force DNS rule ineffective | Rule was below internet pass rule | Move Force DNS above internet access pass |
