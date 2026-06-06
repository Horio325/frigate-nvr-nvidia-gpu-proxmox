# Frigate NVR — Using Proxmox LXC (Debian 12) + Docker + NVIDIA GPU

[![Documentation](https://img.shields.io/badge/docs-v1.0-00d4ff?style=flat-square&logo=gitbook&logoColor=white)](https://horio325.github.io/frigate-nvr-nvidia-gpu-proxmox/)
[![Frigate](https://img.shields.io/badge/Frigate-0.17.1-00e676?style=flat-square&logo=docker&logoColor=white)](https://frigate.video)
[![Proxmox](https://img.shields.io/badge/Proxmox-8.3.0-ff9100?style=flat-square&logo=proxmox&logoColor=white)](https://proxmox.com/en/downloads/proxmox-virtual-environment)
[![NVIDIA](https://img.shields.io/badge/NVIDIA-580.142-76b900?style=flat-square&logo=nvidia&logoColor=white)](https://www.nvidia.com/en-us/drivers/details/265443/)
[![License](https://img.shields.io/badge/license-CC0-b388ff?style=flat-square)](LICENSE)

> A summary guide on how I set up Frigate NVR on Proxmox with Docker, NVIDIA GPU acceleration, multi-camera support, Zabbix monitoring (optional), and automated backups.

---

## 📖 Documentation

The full documentation is available as an interactive dark-mode web page:

**👉 [View Full Documentation](https://horio325.github.io/frigate-nvr-nvidia-gpu-proxmox/)**

---

## 🧰 What's Covered

| # | Section | Description |
|---|---------|-------------|
| 01 | System Overview | Hardware stack, software versions, camera specs |
| 02 | Directory Structure | Where every file lives inside the LXC |
| 03 | Storage Configuration | Proxmox mount points, docker-compose, volume mapping |
| 04 | Frigate Configuration | Full config.yml with multi-camera and audio handling |
| 05 | Data Persistence | First-time setup, database mapping, safe restart |
| 06 | NVIDIA GPU Setup | Driver install, LXC passthrough, container toolkit |
| 07 | GPU Object Detector | ONNX model build, YOLOv9-t configuration |
| 08 | Driver Maintenance | Kernel update recovery, DKMS, auto-rebuild |
| 09 | GPU Verification | Confirm CUDA is being used, not CPU |
| 10 | Shared Memory | SHM sizing for multi-camera setups |
| 11 | Adding Cameras | Steps, audio handling, common errors |
| 12 | Troubleshooting | All errors encountered with causes and fixes |
| 13 | Storage Estimation | Bitrate vs retention calculator |
| 14 | Zabbix Monitoring | CT-100 to CT-102 agent integration *(optional)* |
| 15 | Timezone | Asia/Manila configuration for Philippines |
| 16 | Backup Procedures | Automated daily backup script, restore guide |

---

## ⚙️ Setup at a Glance

```
Proxmox VE 8.3.0
└── LXC CT-100 (frigate-cctv)
    ├── Docker
    │   └── Frigate 0.17.1 (stable-tensorrt image)
    ├── NVIDIA GTX 1050 Ti (driver 580.142)
    ├── 1TB loop1 recording storage at /media/records
    ├── 2× H.265 cameras
    │   ├── entrance    → 192.168.0.249 (no audio)
    │   └── perimeter_1 → 192.168.0.250 (PCMU/8000 audio)
    └── Zabbix Agent → monitored by CT-102 (optional)
```

---

## 📁 Repository Structure

```
frigate-cctv-proxmox/
├── README.md
├── index.html              # full dark-mode documentation (GitHub Pages)
├── LICENSE
├── .gitignore
├── config/
│   └── config.yml          # Frigate configuration template
└── docker/
    └── docker-compose.yml  # Docker Compose configuration
```

---

## 🔑 Key Configuration Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Container type | LXC (not VM) | Lower overhead, direct hardware access |
| Docker image | `stable-tensorrt` | Required for ONNX GPU detection on amd64 |
| Detector | ONNX + `device: cuda` | TensorRT deprecated on amd64 |
| AI Model | YOLOv9-t (tiny) | Best fit for GTX 1050 Ti memory |
| torch version | 2.5.1 (pinned) | 2.6+ breaks model loading |
| GPU Driver | 580.142 | Max version for GTX 1050 Ti (Pascal) |
| Audio (perimeter_1) | `#audio=aac` | PCMU incompatible with MP4 container |
| go2rtc listen | `":8554"` | Prevents ffmpeg startup race condition |

---

## ⚠️ Known Issues

- **`frigate.db` must be a file** — `touch ~/frigate.db` before first start
- **`rtsp: listen: ":8554"`** — the colon before port is required
- **Driver 590+ drops GTX 1050 Ti** — do not upgrade beyond 580.142
- **After Proxmox kernel update** — run `dkms autoinstall` to rebuild NVIDIA driver
- **ONNX defaults to CPU** — must set `device: cuda` explicitly in detectors config
- **PCMU audio → recording failure** — add `#video=copy#audio=aac` to go2rtc stream

---

## 🔗 References

- [Frigate Documentation](https://docs.frigate.video)
- [Proxmox Documentation](https://pve.proxmox.com/wiki)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit)
- [go2rtc](https://github.com/AlexxIT/go2rtc)
- [YOLOv9](https://github.com/WongKinYiu/yolov9)

---

*Documentation Version 1.0 — June 2026*
