# Frigate NVR 

A complete and local NVR designed for Home Assistant with AI object detection. Uses OpenCV and Tensorflow to perform realtime object detection locally for IP cameras.

[https://docs.frigate.video/](Official Docs)

---

!!! note
    This is written with specific references to Loryta(Dahua) IPC-T54IR-AS cameras and a GTX 1080. Other configs will vary for different models.

## Installation 

### Docker 

This is the recommended install method per frigate docs

```yaml 
services:
  frigate:
    container_name: frigate
    # The stable-tensorrt image contains the Nvidia CUDA/TensorRT libraries.
    image: ghcr.io/blakeblackshear/frigate:stable-tensorrt
    restart: unless-stopped
    stop_grace_period: 30s
    # Frigate explicitly recommends larger shared memory for multiple high-res cameras.
    # 1gb is highly recommended for 3x 4MP cameras to prevent frame dropping.
    shm_size: "1gb" 
    env_file:
      - .env
    environment:
      # Optional but recommended to ensure internal timestamps match your host.
      - TZ=America/Chicago
    dns:
      - 10.10.30.1
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              # Frigate needs both 'gpu' (for TensorRT AI) and 'video' (for NVDEC FFmpeg decoding).
              # 'utility' is required for monitoring tools like nvidia-smi.
              capabilities: [gpu, video, utility] 
              
    volumes:
      - ./config:/config
      - /mnt/frigate:/media/frigate
      # Official recommendation: 1GB tmpfs mount to prevent heavy I/O wear on your disks from 24/7 video caching.
      - type: tmpfs 
        target: /tmp/cache
        tmpfs:
          size: 1000000000
          
    ports:
    - "8971:8971"     # Frigate Web UI
    - "8554:8554"     # RTSP feeds from go2rtc
    - "8555:8555/tcp" # WebRTC over TCP
    - "8555:8555/udp" # WebRTC over UDP
```
### Object detector models

> In order to run object detection local modals with the GTX 1080, you must build it from source. This is due to frigate versions before 0.14 used TensorRT natively with default bundled models. When Frigate moved to the ONNX detector approach (0.14+), they deliberately chose not to bundle models -- the docs explicitly say "There is no default model provided." The reasoning is flexibility: users can pick any supported model type/size rather than being locked to one.

> So the build-from-source requirement isn't really a Frigate regression -- it's that the ONNX detector is designed as "bring your own model," and the only way to get a YOLOv9 ONNX file is to export it from the official PyTorch weights yourself.

!!! tip
    The YOLO detector has been designed to support YOLOv3, YOLOv4, YOLOv7, and YOLOv9 models, but may support other YOLO model architectures as well. See the models section for more information on downloading YOLO models for use in Frigate.

YOLOv9 model can be exported as ONNX using the command below. You can copy and paste the whole thing to your terminal and execute, altering MODEL_SIZE=t and IMG_SIZE=320 in the first line to the model size you would like to convert (available model sizes are t, s, m, c, and e, common image sizes are 320 and 640).

```bash
docker build . --build-arg MODEL_SIZE=t --build-arg IMG_SIZE=320 --output . -f- <<'EOF'
FROM python:3.11 AS build
RUN apt-get update && apt-get install --no-install-recommends -y cmake libgl1 && rm -rf /var/lib/apt/lists/*
COPY --from=ghcr.io/astral-sh/uv:0.10.4 /uv /bin/
WORKDIR /yolov9
ADD https://github.com/WongKinYiu/yolov9.git .
RUN uv pip install --system -r requirements.txt
RUN uv pip install --system onnx==1.18.0 onnxruntime onnx-simplifier==0.4.* onnxscript
ARG MODEL_SIZE
ARG IMG_SIZE
ADD https://github.com/WongKinYiu/yolov9/releases/download/v0.1/yolov9-${MODEL_SIZE}-converted.pt yolov9-${MODEL_SIZE}.pt
RUN sed -i "s/ckpt = torch.load(attempt_download(w), map_location='cpu')/ckpt = torch.load(attempt_download(w), map_location='cpu', weights_only=False)/g" models/experimental.py
RUN python3 export.py --weights ./yolov9-${MODEL_SIZE}.pt --imgsz ${IMG_SIZE} --simplify --include onnx
FROM scratch
ARG MODEL_SIZE
ARG IMG_SIZE
COPY --from=build /yolov9/yolov9-${MODEL_SIZE}.onnx /yolov9-${MODEL_SIZE}-${IMG_SIZE}.onnx
EOF
```

## Camera Setup

!!! important
    By default, Loryta(dahua) cameras will use the default IP of `192.168.1.108` - this can be changed in the WebUI following the inital mappings.

1) Access the WebUI and complete the inital setup. Setting an alphanumeric password for the admin password will make it easier for mapping the stream URI later in the frigate `config.yml` 

2) Disable sub stream 1 & enable sub stream 2. 15FPS

3) 

## Link the cameras to Frigate

```yml
mqtt:
  enabled: true
  host: 10.10.30.40
  port: 1883

database:
  path: /media/frigate/frigate.db

auth:
# set to true to reset pass - pass is in logs at first recreate 
  reset_admin_password: false

# ---------------------------------------------------------
# GO2RTC: Official recommendation for restreaming to HA
# ---------------------------------------------------------
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

# ---------------------------------------------------------
# DETECTORS: Officially validated ONNX setup for TensorRT image
# https://docs.frigate.video/configuration/object_detectors#onnx-supported-models
# ---------------------------------------------------------
detectors:
  onnx:
    type: onnx

# Model must be defined for onnx detectors - model built from source via frigate yolov9 docs (m variant)
model:
  path: /config/model_cache/yolov9-m-320.onnx
  model_type: yolo-generic
  width: 320
  height: 320
  input_tensor: nchw
  input_dtype: float
  labelmap_path: /labelmap/coco-80.txt

# ---------------------------------------------------------
# FFMPEG: Hardware Acceleration using the GTX 1080
# ---------------------------------------------------------
ffmpeg:
  # This uses the GTX 1080's NVDEC hardware decoder for your Dahua H.265 streams.
  hwaccel_args: preset-nvidia

# ---------------------------------------------------------
# CAMERAS
# ---------------------------------------------------------
cameras:
  cam_01:
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/cam_01
          # Official Frigate preset for pulling streams from go2rtc
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
  continuous:
    days: 7
  motion:
    days: 7

snapshots:
  enabled: true
  retain:
    default: 7

version: 0.17-0
semantic_search:
  enabled: true
  model_size: large
face_recognition:
  enabled: false
  model_size: small
lpr:
  enabled: false
classification:
  bird:
    enabled: false
```