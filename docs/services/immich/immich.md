# Immich Photo Backup

## Overview

**Immich** is a self-hosted, high-performance photo and video backup solution that aims to directly replace Google Photos. Unlike traditional file managers, Immich relies heavily on machine learning to provide features like facial recognition, object detection (e.g., "search for 'cat'"), and map-based exploration.

The architecture is microservice-based, meaning it runs several distinct containers (Server, Machine Learning, Postgres, Redis) that work in concert to handle massive upload queues and background processing tasks.

---

## Installation

### Docker Compose

Immich is designed to be run via Docker Compose. While there is a monolithic "all-in-one" container available, it is not recommended for production or heavy use. This guide uses the official `ghcr.io` images.

#### Prerequisites

Because Immich uses modular configuration files for hardware acceleration, you **must** download two helper files into your directory before running the compose file.

```bash
wget [https://github.com/immich-app/immich/releases/latest/download/hwaccel.ml.yml](https://github.com/immich-app/immich/releases/latest/download/hwaccel.ml.yml)
wget [https://github.com/immich-app/immich/releases/latest/download/hwaccel.transcoding.yml](https://github.com/immich-app/immich/releases/latest/download/hwaccel.transcoding.yml)

```

#### Compose Configuration

Create a `.env` file in the same directory to handle your storage locations and database passwords.

!!! note "Hardware Note"
This configuration is optimized for an **Intel i5-8500T** using Intel QuickSync and OpenVINO. If you have a dedicated GPU, you will need to adjust the `service: quicksync` and `service: openvino` lines.

```yaml
#
# WARNING: To install Immich, follow our guide: [https://docs.immich.app/install/docker-compose](https://docs.immich.app/install/docker-compose)
#
# Make sure to use the docker-compose.yml of the current release:
#
# [https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml](https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml)
#
# The compose file on main may not be compatible with the latest release.

name: immich

services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    extends:
      file: hwaccel.transcoding.yml
      service: quicksync # set to one of [nvenc, quicksync, rkmpp, vaapi, vaapi-wsl] for accelerated transcoding
    volumes:
      # Do not edit the next line. If you want to change the media storage location on your system, edit the value of UPLOAD_LOCATION in the .env file
      - ${UPLOAD_LOCATION}:/data
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - .env
    ports:
      - '2283:2283'
    depends_on:
      - redis
      - database
    restart: always
    healthcheck:
      disable: false

  immich-machine-learning:
    container_name: immich_machine_learning
    # For hardware acceleration, add one of -[armnn, cuda, rocm, openvino, rknn] to the image tag.
    # Example tag: ${IMMICH_VERSION:-release}-cuda
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}-openvino
    extends: # uncomment this section for hardware acceleration - see [https://docs.immich.app/features/ml-hardware-acceleration](https://docs.immich.app/features/ml-hardware-acceleration)
      file: hwaccel.ml.yml
      service: openvino # set to one of [armnn, cuda, rocm, openvino, openvino-wsl, rknn] for accelerated inference - use the `-wsl` version for WSL2 where applicable
    volumes:
      - model-cache:/cache
    env_file:
      - .env
    restart: always
    healthcheck:
      disable: false

  redis:
    container_name: immich_redis
    image: docker.io/valkey/valkey:9@sha256:fb8d272e529ea567b9bf1302245796f21a2672b8368ca3fcb938ac334e613c8f
    healthcheck:
      test: redis-cli ping || exit 1
    restart: always

  database:
    container_name: immich_postgres
    image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:bcf63357191b76a916ae5eb93464d65c07511da41e3bf7a8416db519b40b1c23
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
      # Uncomment the DB_STORAGE_TYPE: 'HDD' var if your database isn't stored on SSDs
      # DB_STORAGE_TYPE: 'HDD'
    volumes:
      # Do not edit the next line. If you want to change the database storage location on your system, edit the value of DB_DATA_LOCATION in the .env file
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    shm_size: 128mb
    restart: always

volumes:
  model-cache:

```

---

## Initial Setup

After ensuring the helper YAML files and `.env` are present, run `docker compose up -d`.

Navigate to `http://your.server.ip:2283` to see the "Getting Started" wizard.

1. Click **Getting Started**.
2. **Create Admin Account:** Enter your email, name, and password.
3. **Log in:** You will be taken to the main timeline.

---

## Hardware Acceleration

### Video Transcoding (Intel QuickSync)

**Why use it:**
When you view a 4K video on a phone over a slow connection, Immich must transcode (shrink) that video in real-time. Doing this on the CPU will pin your processor to 100% usage and create stuttering. QuickSync offloads this to the iGPU.

**How to configure:**
The `hwaccel.transcoding.yml` in the compose file handles the backend permissions, but you must enable it in the UI.

1. Go to **Administration** (top right) -> **Settings** -> **Video Transcoding**.
2. Find **Hardware Acceleration**.
3. Select **Quick Sync** from the dropdown.
4. Click **Save**.

### Machine Learning (OpenVINO)

**Why use it:**
Immich scans every photo for faces and objects. On a raw CPU, scanning 1,000 photos can take hours. OpenVINO optimizes these mathematical operations for Intel CPUs, significantly increasing indexing speed.

**How to configure:**
There is no UI toggle for this. It is controlled entirely by the image tag (`-openvino`) and the `extends` block in the compose file.

**Verification:**
Run the following to check if it loaded correctly:

```bash
docker compose logs immich-machine-learning | grep "OpenVINO"

```

* **Success:** You will see a line stating `Loaded execution provider: OpenVINO`.
* **Failure:** If you only see `CPUExecutionProvider`, check that your image tag ends in `-openvino`.

---

## Database & Storage

### PostgreSQL with pgvector

Immich requires a specific version of Postgres equipped with `pgvector`. This extension allows the database to store "vector embeddings"—mathematical representations of images. This enables smart searches (e.g., searching for "red truck" finds trucks without manual tags).

!!! warning "Performance Tip"
Unlike Nextcloud, you rarely touch the DB config manually. However, ensure `DB_DATA_LOCATION` in your `.env` points to fast storage (SSD/NVMe). Vector searches on HDDs are noticeably slower.

### Storage Template

By default, Immich uploads files into a generic folder structure. If you ever want to access your files directly on the disk (outside of Immich), this is difficult to navigate.

**The Fix:**
Configure this **before** uploading your library.

1. Go to **Administration** -> **Settings** -> **Storage Template**.
2. **Enabled:** Toggle ON.
3. **Template:** Select or customize your structure.
* *Recommendation:* `{{y}}/{{y}}-{{MM}}-{{dd}}/{{filename}}`
* *Result:* `/2024/2024-12-25/IMG_1234.jpg`


4. Click **Save**.

---

## Post-Install Monitoring

### GPU Utilization

Since you are using hardware acceleration, verify the load is hitting the GPU and not the CPU.

**The Tool:** `intel_gpu_top`

1. **Install on Host:**
```bash
sudo apt install intel-gpu-tools

```


2. **Run Monitor:**
```bash
sudo intel_gpu_top

```


3. **Test:** Open Immich and scroll rapidly through the timeline (triggers thumbnail generation) or play a large video. You should see the `Video` or `Render` bars spike.

---

*Sources:*

* [Immich Hardware Acceleration Guide](https://immich.app/docs/features/hardware-transcoding)
* [Recommended Compose File](https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml)

