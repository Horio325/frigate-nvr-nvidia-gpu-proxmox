# Frigate NVR — Using Proxmox LXC (Debian 12) + Docker + NVIDIA GPU

A summary guide on how I set up Frigate NVR on Proxmox with Docker, NVIDIA GPU acceleration, multi-camera support, Zabbix monitoring (optional), and automated backups.

Full Frigate Documentation goes here: https://docs.frigate.video/

---

## 📋 Table of Contents

- [System Overview](#system-overview)
- [Directory Structure](#directory-structure)
- [Storage Configuration](#storage-configuration)
- [Frigate Configuration](#frigate-configuration)
- [Data Persistence](#data-persistence)
- [NVIDIA GPU Setup](#nvidia-gpu-setup)
- [GPU Object Detector (ONNX)](#gpu-object-detector-onnx)
- [NVIDIA Driver Maintenance](#nvidia-driver-maintenance)
- [GPU Detection Verification](#gpu-detection-verification)
- [Shared Memory Configuration](#shared-memory-configuration)
- [Adding Additional Cameras](#adding-additional-cameras)
- [Troubleshooting](#troubleshooting)
- [Storage Estimation](#storage-estimation)
- [Zabbix Monitoring Integration](#zabbix-monitoring-integration)
- [Timezone Configuration](#timezone-configuration)
- [Backup Procedures](#backup-procedures)

---

## System Overview

Frigate is an open-source Network Video Recorder (NVR) that uses AI to detect people, vehicles, and other objects in your camera feeds. It runs inside a lightweight Linux container (LXC) on Proxmox, managed by Docker.

### Hardware & Software Stack

I'm using an old desktop PC with an NVIDIA GTX 1050 Ti as a Frigate NVR test system. The setup is used to evaluate GPU-accelerated object detection, camera stream performance, recording stability, and storage usage before deploying to dedicated hardware.

| Component | Details |
|-----------|---------|
| Processor | Intel i5-7400 3.5GHz |
| RAM | 8GB |
| GPU | NVIDIA GTX 1050 Ti |
| Hypervisor | Proxmox VE 8.3.0 |
| Container | LXC Container 100 (frigate-cctv) on node 'horios' |
| NVR Software | Frigate 0.17.1 |
| Deployment | Docker inside LXC |
| GPU Driver | 580.142 (Linux) |
| Primary Storage | 20GB root disk |
| Recording Storage | 1TB (1000GB) loop1 at `/media/records` |
| Camera 1 - Hikvision (entrance) | [Camera IP address] — H.265/HEVC, 1920x1080, no audio - DS-2CD1123G0-I |
| Camera 2 - Hikvision (perimeter_1) | [Camera IP address] — H.265/HEVC, 1920x1080, PCMU/8000 audio - DS-2CD2023G2-IU |
| OS (Proxmox) | Debian Bookworm |

---

## Directory Structure

```
/root/
├── docker-compose.yml       # Docker configuration
├── frigate.db               # Persistent Frigate database (alerts/detections)
├── config/
│   └── config.yml           # Frigate configuration
└── media/
    └── records/             # Loop1 1TB recording storage
        ├── recordings/      # Continuous recording segments
        ├── clips/           # Event clips and thumbnails
        └── exports/         # Exported footage
```

>⚠️ `frigate.db` must exist as a **file** before starting the container. Run `touch ~/frigate.db` to create it.

---

## Storage Configuration

### Proxmox Mount Points

- `loop0` (20GB) — Root filesystem at `/`
- `loop1` (1TB / 1000GB) — Recording storage at `/media/records`

```
cctv-records:100/vm-100-disk-1.raw,mp=/media/records,backup=1,size=1000G
```

### docker/docker-compose.yml

```yaml
services:
  frigate:
    container_name: frigate-cctv
    restart: unless-stopped
    stop_grace_period: 30s
    image: ghcr.io/blakeblackshear/frigate:stable-tensorrt 
    runtime: nvidia
    shm_size: "1024mb"
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=all
      - TZ=Asia/Manila
    devices:
      - /dev/nvidia0:/dev/nvidia0
      - /dev/nvidiactl:/dev/nvidiactl
      - /dev/nvidia-uvm:/dev/nvidia-uvm
    volumes:
      - ./config:/config
      - /media/records:/media/frigate
      - ./frigate.db:/config/frigate.db
      - /etc/localtime:/etc/localtime:ro      #  ^f^p sync with host time
      - /etc/timezone:/etc/timezone:ro        #  ^f^p sync with host timezone
      - type: tmpfs # 1GB In-memory filesystem for recording segment storage
        target: /tmp/cache
        tmpfs:
          size: 3000000000
    ports:
      - "8971:5000"
      - "8554:8554" # RTSP feeds

```

### Volume Mapping Summary

| Host Path | Container Path | Purpose | Persistent? |
|-----------|---------------|---------|-------------|
| `./config` | `/config` | Frigate config.yml | ✅ Yes |
| `/media/records` | `/media/frigate` | 1TB recordings & clips | ✅ Yes |
| `./frigate.db` | `/config/frigate.db` | Database (alerts/detections) | ✅ Yes |
| `/etc/localtime` | `/etc/localtime:ro` | Sync timezone with host | ✅ Yes |
| `/etc/timezone` | `/etc/timezone:ro` | Sync timezone name with host | ✅ Yes |
| `tmpfs` | `/tmp/cache` | Segment write cache (2GB) | ❌ RAM only |

### SHM Size Reference

| Resolution | Cameras | Recommended SHM |
|------------|---------|-----------------|
| 1080p | 1 | 128MB |
| 1080p | 2 (current) | 1024MB ✅ |
| 1080p | 4+ | 2GB |
| 4K | 1-2 | 1GB |
| 4K | 3+ | 2GB |

---

## Frigate Configuration

### config/config.yml

```yaml
mqtt:
  enabled: false

detectors:
  onnx:
    type: onnx         # NVIDIA GPU auto-detected in tensorrt image
    device: cuda

model:
  model_type: yolo-generic
  width: 320
  height: 320
  input_tensor: nchw
  input_dtype: float
  path: /config/model_cache/yolov9-t-320.onnx
  labelmap_path: /labelmap/coco-80.txt


ffmpeg:
  hwaccel_args: preset-nvidia-h265

record:
  enabled: true
  continuous:
    days: 30
  detections:
    pre_capture: 30        # seconds before detection to include
    post_capture: 30       # seconds after detection to include
    retain:
      days: 30
      mode: motion
  alerts:
    pre_capture: 10
    post_capture: 10
    retain:
      days: 30
      mode: motion

cameras:
  entrance:
    enabled: true
    ffmpeg:
      input_args: preset-rtsp-restream
      output_args:
        record: preset-record-generic-audio-aac
      inputs:
        - path: rtsp://127.0.0.1:8554/entrance_1
          roles:
            - record
        - path: rtsp://127.0.0.1:8554/entrance_2
          roles:
            - detect
    snapshots:
      enabled: true
    detect:
      enabled: true
      width: 1920
      height: 1080
      fps: 5
    record:
      enabled: true
      continuous:
        days: 14     # per-camera override
    live:
      streams:
        Stream 1: entrance_1
        Stream 2: entrance_2

  perimeter_1:
    enabled: true
    ffmpeg:
      input_args: preset-rtsp-restream
      hwaccel_args: preset-nvidia-h265
      inputs:
        - path: rtsp://127.0.0.1:8554/perimeter_1_main
          roles:
            - record
        - path: rtsp://127.0.0.1:8554/perimeter_1_sub
          roles:
            - detect
      output_args:
        record: preset-record-generic-audio-aac
    detect:
      enabled: true
      width: 1920      # adjust to your camera resolution
      height: 1080
      fps: 5           # ← start LOW (5fps) for detect stream
    record:
      enabled: true
      continuous:
        days: 14
    live:
      streams:
        Stream 1: perimeter_1_main
        Stream 2: perimeter_1_sub
    objects:
      mask: 0.46,0,0.994,1,0.994,0
version: 0.17-0
go2rtc:
  streams:
    entrance_1:
      - rtsp://admin:PASSWORD@[CAMERA-1 IP ADDRESS]/media/video1
    entrance_2:
      - rtsp://admin:PASSWORD@[CAMERA-1 IP ADDRESS]/media/video2
    perimeter_1_main:
      - "rtsp://admin:PASSWORD@[CAMERA-2 IP ADDRESS]/media/video1#video=copy#audio=aac"  # ← transcode audio to AAC
    perimeter_1_sub:
      - "rtsp://admin:PASSWORD@[CAMERA-2 IP ADDRESS]/media/video2#video=copy#audio=disable"  # ← disable audio on detect
  rtsp:
    listen: ":8554"
```

### Key Configuration Notes

| Setting | Value | Reason |
|---------|-------|--------|
| `detectors.onnx.device` | `cuda` | Forces GPU detection — without this ONNX uses CPU |
| `hwaccel_args` | `preset-nvidia-h265` | Both cameras use H.265/HEVC codec |
| `input_args` | `preset-rtsp-restream` | Routes through go2rtc to fix codec errors |
| `output_args record` | `preset-record-generic-audio-aac` | Handles audio transcoding for MP4 |
| `detect.enabled` | `true` | Must be explicitly enabled at boot |
| `entrance detect fps` | `20` | Stable for entrance camera (no audio) |
| `perimeter_1 detect fps` | `5` | Start low for new cameras; increase if stable |
| `go2rtc rtsp listen` | `:8554` | Binds RTSP immediately on startup — prevents race condition |
| `perimeter_1_main audio` | `#audio=aac` | Transcodes PCMU/8000 to AAC for MP4 |
| `perimeter_1_sub audio` | `#audio=disable` | Detect stream does not need audio |

### Multi-Camera Audio Handling

My Dome camera (entrance) has no built in audio so I don't have to put `#video=copy#audio=aac` & `#video=copy#audio=disable` in go2rtc stream.

| Camera | Audio Codec | go2rtc Flag | Reason |
|--------|------------|-------------|--------|
| entrance (main) | No audio | No flag needed | No audio stream |
| entrance (sub) | No audio | No flag needed | Detect, no audio |
| perimeter_1 (main) | PCMU/8000 | `#video=copy#audio=aac` | Required for MP4 muxing |
| perimeter_1 (sub) | PCMU/8000 | `#video=copy#audio=disable` | Detect stream, no audio |

> ⚠️ If PCMU audio is not handled: `No new valid recording segments were created in the last 120s.`

> ⚠️ `rtsp: listen` requires a colon before the port: `":8554"` not `"8554"`

---

## Data Persistence

### First-Time Setup Checklist

```bash
mkdir -p ~/config
nano ~/config/config.yml
touch ~/frigate.db
docker cp frigate-cctv:/config/frigate.db ~/frigate.db   # if container exists
docker compose up -d
docker logs frigate-cctv 2>&1 | grep -i "config\|camera\|database"
```

### Safe Restart

```bash
docker compose down && docker compose up -d
```

> ⚠️ `frigate.db` must be a file not a directory. Fix: `rm -rf ~/frigate.db && touch ~/frigate.db`

---

## NVIDIA GPU Setup

### 1. Install Driver on Proxmox Host

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/cuda-keyring_1.1-1_all.deb
dpkg -i cuda-keyring_1.1-1_all.deb
apt update && apt install -y cuda-drivers
echo "blacklist nouveau" > /etc/modprobe.d/blacklist-nouveau.conf
echo "options nouveau modeset=0" >> /etc/modprobe.d/blacklist-nouveau.conf
update-initramfs -u -k all && reboot
```

### 2. LXC GPU Passthrough (`/etc/pve/lxc/100.conf`)

```
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 236:* rwm
lxc.cgroup2.devices.allow: c 239:* rwm
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
```

```bash
pct stop 100 && pct start 100
```

### 3. Install Driver Inside LXC

```bash
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/580.142/NVIDIA-Linux-x86_64-580.142.run
chmod +x NVIDIA-Linux-x86_64-580.142.run
./NVIDIA-Linux-x86_64-580.142.run --no-kernel-module
```

### 4. NVIDIA Container Toolkit

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
apt update && apt install -y nvidia-container-toolkit
nvidia-ctk runtime configure --runtime=docker --set-as-default
systemctl restart docker
```

---

## GPU Object Detector (ONNX)

```bash
docker build . --build-arg MODEL_SIZE=t --build-arg IMG_SIZE=320 \
  --output /root/config/model_cache -f- <<'EOF'
FROM python:3.11 AS build
RUN apt-get update && apt-get install --no-install-recommends -y cmake libgl1 \
  && rm -rf /var/lib/apt/lists/*
COPY --from=ghcr.io/astral-sh/uv:0.10.4 /uv /bin/
WORKDIR /yolov9
RUN git clone https://github.com/WongKinYiu/yolov9.git .
RUN uv pip install --system -r requirements.txt
RUN uv pip install --system onnx==1.18.0 onnxruntime onnx-simplifier==0.4.* onnxscript
RUN uv pip install --system torch==2.5.1 torchvision \
  --index-url https://download.pytorch.org/whl/cpu
ARG MODEL_SIZE
ARG IMG_SIZE
ADD https://github.com/WongKinYiu/yolov9/releases/download/v0.1/yolov9-${MODEL_SIZE}-converted.pt \
  yolov9-${MODEL_SIZE}.pt
RUN sed -i "s/map_location='cpu'/map_location='cpu', weights_only=False/g" models/experimental.py
RUN python3 export.py --weights ./yolov9-${MODEL_SIZE}.pt --imgsz ${IMG_SIZE} --simplify --include onnx
FROM scratch
COPY --from=build /yolov9/yolov9-${MODEL_SIZE}.onnx /yolov9-${MODEL_SIZE}-${IMG_SIZE}.onnx
EOF
```

| Size | Flag | Speed | For |
|------|------|-------|-----|
| Tiny | `MODEL_SIZE=t` | Fastest | GTX 1050 Ti ✅ |
| Small | `MODEL_SIZE=s` | Fast | GTX 1060+ |
| Medium | `MODEL_SIZE=m` | Medium | RTX cards |

---

## NVIDIA Driver Maintenance

### Fix After Kernel Update

```bash
apt install -y dkms pve-headers-$(uname -r)
dkms autoinstall -k $(uname -r)
dkms install nvidia/580.142 -k $(uname -r) --force   # if status shows 'built'
dkms status                                            # verify 'installed'
modprobe nvidia && modprobe nvidia-uvm && modprobe nvidia_modeset
ls -la /dev/nvidia*
nvidia-smi
```

### Auto-load on Boot

```bash
echo -e "nvidia\nnvidia-uvm\nnvidia_modeset" > /etc/modules-load.d/nvidia.conf
systemctl restart systemd-modules-load
```

### Auto-rebuild on Kernel Update

```bash
echo 'DPkg::Post-Invoke {"dkms autoinstall 2>/dev/null || true";};' \
  > /etc/apt/apt.conf.d/99nvidia-dkms
```

| Status | Meaning | Action |
|--------|---------|--------|
| `installed` | Ready to load | None |
| `built` | Not installed | `dkms install nvidia/580.142 -k $(uname -r) --force` |
| `not found` | Not built | `dkms autoinstall -k $(uname -r)` |

> ⚠️ GTX 1050 Ti max driver is **580.142** — NVIDIA dropped Pascal (GTX 10 series) in driver 590+.

---

## GPU Detection Verification

```bash
docker exec -it frigate-cctv python3 -c "
import onnxruntime as ort
print('Providers:', ort.get_available_providers())
print('Device:', ort.get_device())
"
# Expected: ['TensorrtExecutionProvider', 'CUDAExecutionProvider', 'CPUExecutionProvider']
```

If GPU-Util shows 0%, add `device: cuda` to detectors:
```yaml
detectors:
  onnx:
    type: onnx
    device: cuda
```

---

## Shared Memory Configuration

```yaml
shm_size: "1024mb"     # 1GB for 2x 1080p cameras
```

---

## Adding Additional Cameras

1. Check the new camera's RTSP paths via its admin page
2. Add streams to `go2rtc` in config.yml
3. Handle audio: `#audio=aac` for record, `#audio=disable` for detect
4. Add camera block under `cameras`
5. Start with `fps: 5` for detect
6. Increase `shm_size` and `tmpfs` as needed
7. Restart and verify: `wget -q -O- http://localhost:1984/api/streams`

---

## Troubleshooting

This is a summary of the problems I faced while configuring the Frigate NVR.

| Error | Cause | Fix |
|-------|-------|-----|
| Settings reset on restart | config.yml not in `./config/` | `mkdir -p ~/config` |
| Alerts gone on restart | frigate.db not mapped as file | `touch ~/frigate.db` |
| Mount error: not a directory | frigate.db is a directory | `rm -rf ~/frigate.db && touch ~/frigate.db` |
| detect not running | `detect.enabled` not set | Add `detect: enabled: true` |
| ffmpeg process not running | go2rtc race condition | Add `rtsp: listen: ":8554"` |
| Connection refused port 8554 | Wrong listen syntax | Use `":8554"` not `"8554"` |
| 404 on rtsp://127.0.0.1:8554 | Stream name mismatch | Match go2rtc names exactly |
| No new valid recording segments | PCMU audio incompatible | Add `#video=copy#audio=aac` |
| Too many unprocessed segments | Detect fps too high | Lower `fps` to `5` |
| `CUDA_ERROR_UNKNOWN` | GPU not passed through | Add `lxc.mount.entry` lines |
| No `/dev/nvidia*` after reboot | Modules not loaded | `modprobe nvidia` or configure auto-load |
| DKMS `built` not `installed` | Not installed in kernel | `dkms install --force` |
| GPU at 0% / no processes | ONNX using CPU | Add `device: cuda` to detectors |
| SHM too small | Not enough shared memory | Increase `shm_size` to `1024mb` |
| Wrong timezone | No TZ set | Add `TZ=Asia/Manila` to environment |
| TensorRT not supported | Wrong detector | Use `type: onnx` with `stable-tensorrt` |


---

## Storage Estimation

| Bitrate | Per Day | 7 Days | 30 Days |
|---------|---------|--------|---------|
| 2 Mbps | ~21 GB | ~150 GB | ~630 GB |
| 4 Mbps | ~43 GB | ~300 GB | ~1.25 TB |
| 6 Mbps | ~65 GB | ~450 GB | ~1.9 TB |

> With 2 cameras on 1TB and 30-day retention, ~3 Mbps per camera is sustainable.

---

## Zabbix Monitoring Integration
This section is not necessary because I experimented with Zabbix to monitor the network, CPU, and memory usage of Frigate NVR.

| Container | CT ID | Role | IP |
|-----------|-------|------|----|
| frigate-cctv | CT-100 | Zabbix Agent | 192.168.0.242 |
| zabbix | CT-102 | Zabbix Server | 192.168.0.XXX |

```bash
apt install -y zabbix-agent
```

`/etc/zabbix/zabbix_agentd.conf`:
```
Server=192.168.0.XXX,127.0.0.1
ServerActive=192.168.0.XXX
Hostname=frigate-cctv
```

```bash
systemctl enable zabbix-agent && systemctl start zabbix-agent
```

---

## Timezone Configuration

```yaml
environment:
  - TZ=Asia/Manila
volumes:
  - /etc/localtime:/etc/localtime:ro
  - /etc/timezone:/etc/timezone:ro
```

```bash
timedatectl set-timezone Asia/Manila
docker exec -it frigate-cctv date   # verify UTC+8
```

---

## Backup Procedures

```bash
#!/bin/bash
BACKUP_DIR=~/backups/$(date +%Y%m%d)
mkdir -p $BACKUP_DIR
cp ~/config/config.yml $BACKUP_DIR/
cp ~/docker-compose.yml $BACKUP_DIR/
docker compose stop
cp ~/frigate.db $BACKUP_DIR/frigate_$(date +%Y%m%d).db
docker compose start
find ~/backups -type d -mtime +7 -exec rm -rf {} +
echo "Backup completed: $BACKUP_DIR"
```

Schedule: `0 2 * * * /root/backup_frigate.sh >> /var/log/frigate_backup.log 2>&1`
