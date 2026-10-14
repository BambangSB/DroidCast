# 📹 DroidCast

> **Turn your Android phone into a wireless webcam & screen mirror for PC — for free.**

DroidCast is a free, open-source alternative to DroidCam that lets you use your Android phone's camera or screen as a virtual webcam on your Windows PC. It supports both WiFi and USB connections, with features including audio streaming, zoom/brightness controls, screen sharing, and internal audio capture.

---

## ✨ Features

### 📷 Camera Mode
- Stream front or back camera as a PC webcam
- Adjustable resolution: **720p / 1080p / 4K**
- Real-time zoom and brightness controls
- Flip/switch camera on the fly
- Microphone audio streaming

### 🖥️ Screen Share Mode
- Mirror your Android screen to PC as a webcam
- **Internal audio capture** (game sounds, music, video — Android 10+)
- Perfect for mobile game streaming, app demos, presentations

### 🔗 Connectivity
- **WiFi**: Auto-discovery via mDNS (zero-configuration)
- **USB**: Via ADB port forwarding (low latency, no WiFi needed)
- **QR Code**: Quick connect by scanning QR code
- **Auto-reconnect**: Automatically reconnects on disconnect

### 💻 PC Companion
- **Standalone** — no OBS Studio required
- Bundled **Unity Capture** virtual camera driver (MIT license)
- **VB-Cable** support for virtual microphone
- System tray icon with minimize-to-tray
- Modern dark/light UI (CustomTkinter)

---

## 🏗️ Architecture

```
┌──────────────────────────────┐          ┌──────────────────────────────┐
│     📱 DroidCast Android     │          │     💻 DroidCast PC          │
│                              │          │                              │
│  Camera/Screen → JPEG Encode │──WiFi──▶│  MJPEG Reader → Virtual Cam  │
│  Mic/Internal → PCM Audio    │──USB───▶│  Audio Receiver → VB-Cable   │
│  HTTP Server (:8080)         │          │  Auto-Discovery (Zeroconf)   │
│  mDNS Broadcast              │          │  System Tray + Settings GUI  │
└──────────────────────────────┘          └──────────────────────────────┘
```

---

## 📱 Android App (Flutter/Dart)

### Requirements
- Android 7.0+ (API 24) minimum
- Android 10+ (API 29) for internal audio capture
- Camera and microphone permissions
- Screen capture permission (for screen share mode)

### Tech Stack
| Component | Technology |
|-----------|-----------|
| Framework | Flutter (Dart) |
| Camera | CameraX via `camera` package |
| Video Encoding | Native Kotlin `YuvImage.compressToJpeg()` |
| Audio | `record` package (mic) + `AudioPlaybackCapture` (internal) |
| HTTP Server | `shelf` + `shelf_router` |
| Screen Capture | `MediaProjection` API (native Kotlin) |
| Service Discovery | `bonsoir` (mDNS/Bonjour) |

### HTTP Endpoints
| Method | Path | Description |
|--------|------|-------------|
| GET | `/video` | MJPEG video stream (camera or screen) |
| GET | `/audio` | Raw PCM audio stream (mic or internal) |
| GET | `/status` | Device info, mode, resolution, fps |
| POST | `/control` | Remote control (zoom, brightness, flip, mode) |
| GET | `/` | Web preview page |

---

## 💻 PC Companion (Python → .exe)

### Requirements
- Windows 10/11
- **Unity Capture** driver (bundled in installer, auto-registered)
- **VB-Cable** (free download) for virtual microphone

### Tech Stack
| Component | Technology |
|-----------|-----------|
| Virtual Camera | `pyvirtualcam` + Unity Capture |
| Video Decoding | `opencv-python` + `requests` |
| Audio Routing | `sounddevice` → VB-Cable |
| Auto-Discovery | `zeroconf` (mDNS) |
| USB Bridge | `adbutils` (ADB port forwarding) |
| GUI | `customtkinter` |
| System Tray | `pystray` + `Pillow` |
| Distribution | PyInstaller + Inno Setup |

### Installation
1. Download `DroidCast-Setup.exe` from [Releases](https://github.com/BambangSB/DroidCast/releases)
2. Run installer (auto-registers virtual camera driver)
3. (Optional) Install [VB-Cable](https://vb-audio.com/Cable/) for audio routing
4. Launch DroidCast PC Companion

---

## 🚀 Quick Start

### WiFi Connection
1. Open **DroidCast** on your Android phone
2. Select **Camera** or **Screen** mode
3. Tap **Start Streaming**
4. Open **DroidCast PC Companion** on your computer
5. Your phone will be auto-discovered — click **Start Virtual Camera**
6. In Zoom/Teams/Meet, select **"Unity Video Capture"** as your camera

### USB Connection
1. Connect your phone via USB cable
2. Enable **USB Debugging** in Developer Options
3. In DroidCast PC, select **USB** mode
4. Click **Start Virtual Camera**

---

## 📁 Project Structure

```
DroidCast/
├── android/                          # Flutter Android App
│   ├── lib/
│   │   ├── main.dart
│   │   ├── screens/
│   │   ├── services/
│   │   ├── models/
│   │   ├── widgets/
│   │   └── utils/
│   ├── android/app/src/main/
│   │   ├── AndroidManifest.xml
│   │   └── kotlin/com/droidcast/app/
│   │       ├── MainActivity.kt
│   │       ├── JpegEncoder.kt
│   │       ├── ScreenCaptureService.kt
│   │       └── InternalAudioCapturer.kt
│   └── pubspec.yaml
│
├── pc-companion/                     # Python PC Companion
│   ├── main.py
│   ├── core/
│   ├── discovery/
│   ├── ui/
│   ├── drivers/
│   ├── utils/
│   ├── requirements.txt
│   └── build/
│
├── docs/                             # Documentation
│   └── IMPLEMENTATION_PLAN.md
│
└── README.md
```

---

## 📋 Implementation Roadmap

- [x] **Phase 0**: Project planning & architecture design
- [ ] **Phase 1**: MVP — Camera streaming via WiFi
- [ ] **Phase 2**: Audio streaming + camera controls
- [ ] **Phase 3**: Screen sharing + internal audio capture
- [ ] **Phase 4**: Auto-connect (mDNS) + USB support
- [ ] **Phase 5**: Polish, installer, distribution

---

## 📄 License

This project is open-source and free to use.

---

## 🙏 Acknowledgments

- [Unity Capture](https://github.com/schellingb/UnityCapture) — Standalone virtual camera driver (MIT License)
- [pyvirtualcam](https://github.com/letmaik/pyvirtualcam) — Python virtual camera library
- [VB-Cable](https://vb-audio.com/Cable/) — Virtual audio device
- [Flutter](https://flutter.dev/) — Cross-platform UI framework