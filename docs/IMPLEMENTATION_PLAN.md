# 📹 DroidCast — Implementation Plan

## Goal
Membangun aplikasi gratis mirip DroidCam yang terdiri dari dua komponen:
1. **DroidCast Android App** (Flutter/Dart) — menangkap kamera, mikrofon, screen, dan internal audio HP, lalu men-stream melalui jaringan
2. **DroidCast PC Companion** (Python → `.exe`) — menerima stream, membuat virtual webcam & virtual microphone di PC tanpa membutuhkan OBS Studio

---

## Key Decisions

| Decision | Choice |
|----------|--------|
| App Name | **DroidCast** |
| Android Framework | **Flutter (Dart)** |
| Minimum Android SDK | **API 24 (Android 7.0)** — internal audio auto-off below Android 10 |
| Resolution Options | **720p / 1080p / 4K** (user-selectable) |
| PC Companion Language | **Python** |
| PC Distribution | **PyInstaller .exe** + **Inno Setup installer** |
| Virtual Camera | **Unity Capture** (standalone, no OBS required) |
| Virtual Microphone | **VB-Cable** (user installs separately, free) |
| Connection Modes | **WiFi** (auto-discovery via mDNS) + **USB** (ADB forward) |
| Screen Sharing | **MediaProjection API** + **AudioPlaybackCapture** (Android 10+) |

---

## System Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                        📱 DroidCast Android App                          │
│                                                                           │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐                  │
│  │ Camera Mode  │    │ Screen Mode │    │ Audio Source  │                  │
│  │ (CameraX)    │    │ (MediaProj) │    │ Mic / Internal│                  │
│  └──────┬───────┘    └──────┬──────┘    └──────┬───────┘                  │
│         │                   │                   │                          │
│         ▼                   ▼                   │                          │
│  ┌─────────────────────────────────┐            │                          │
│  │  YUV/RGBA → JPEG Encoder       │            │                          │
│  │  (Native Kotlin Platform Channel)│            │                          │
│  └──────────────┬──────────────────┘            │                          │
│                 │                                │                          │
│                 ▼                                ▼                          │
│  ┌──────────────────────────────────────────────────┐                     │
│  │         HTTP Server (shelf) :8080                 │                     │
│  │  GET /video  → MJPEG stream                       │                     │
│  │  GET /audio  → Raw PCM stream                     │                     │
│  │  GET /status → JSON device info                   │                     │
│  │  POST /control → zoom/brightness/flip/mode        │                     │
│  └──────────────────────┬───────────────────────────┘                     │
│                         │                                                  │
│  ┌──────────────────────┼───────────────┐                                 │
│  │ mDNS Broadcast       │               │                                 │
│  │ (bonsoir)            │               │                                 │
│  └──────────────────────┼───────────────┘                                 │
└─────────────────────────┼─────────────────────────────────────────────────┘
                          │
            WiFi ─────────┤──────── USB (adb forward)
                          │
┌─────────────────────────┼─────────────────────────────────────────────────┐
│                         ▼                                                  │
│  ┌──────────────────────────────────────────────────┐                     │
│  │   MJPEG Reader (requests + cv2.imdecode)         │                     │
│  │   Low-latency: manual SOI/EOI parsing            │                     │
│  │   (NOT cv2.VideoCapture — avoids 1-5s buffering) │                     │
│  └──────────────────────┬───────────────────────────┘                     │
│                         │                                                  │
│            ┌────────────┴────────────┐                                    │
│            ▼                         ▼                                    │
│  ┌─────────────────┐    ┌──────────────────┐                              │
│  │ pyvirtualcam     │    │ sounddevice       │                              │
│  │ → Unity Capture  │    │ → VB-Cable Input  │                              │
│  │ (virtual webcam) │    │ (virtual mic)     │                              │
│  └─────────────────┘    └──────────────────┘                              │
│                                                                           │
│  ┌─────────────────────────────────────────────┐                          │
│  │ CustomTkinter GUI + pystray System Tray     │                          │
│  │ Zeroconf Auto-Discovery + ADB USB Bridge    │                          │
│  └─────────────────────────────────────────────┘                          │
│                                                                           │
│                       💻 DroidCast PC Companion (.exe)                    │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Android App: Detailed Component Design

### 1. Camera Service (`camera_service.dart`)
- Uses `CameraController` from `camera` package
- `startImageStream()` captures YUV420 frames at sensor rate (30-60fps)
- Throttle to target fps (25-30) using frame-skip flag
- YUV bytes sent to native encoder via MethodChannel
- Resolution presets: `low` (480p) / `medium` (720p) / `high` (1080p) / `ultraHigh` (4K)

### 2. Native JPEG Encoder (`JpegEncoder.kt`)
- Uses Android's hardware-accelerated `YuvImage.compressToJpeg()`
- ~3-8ms per frame (vs ~50-100ms in pure Dart)
- Also handles RGBA→JPEG for screen capture frames
- Quality parameter: 50-95 (default 80)

### 3. Screen Capture Service (`ScreenCaptureService.kt`)
- Foreground service with `foregroundServiceType="mediaProjection"`
- `MediaProjectionManager.createScreenCaptureIntent()` → system permission dialog
- `ImageReader.newInstance(w, h, RGBA_8888, 2)` for frame capture
- Handles GPU row stride padding (`rowStride ≠ width × pixelStride`)
- `createVirtualDisplay()` called ONCE (Android 14 restriction)
- Same `MediaProjection` token reused for both screen + internal audio

### 4. Internal Audio Capturer (`InternalAudioCapturer.kt`)
- `AudioPlaybackCaptureConfiguration.Builder(mediaProjection)`
- Captures: `USAGE_MEDIA`, `USAGE_GAME`, `USAGE_UNKNOWN`
- NOT captured (Android policy): calls, alarms, notifications
- Output: Raw PCM 16-bit, 44100Hz, Stereo
- **Android 10+ only** — gracefully disabled on older versions

### 5. HTTP Server (`http_server.dart`)
- `shelf` + `shelf_router` bound to `0.0.0.0:8080`
- `/video`: MJPEG multipart stream (`multipart/x-mixed-replace; boundary=frame`)
- `/audio`: Chunked PCM stream (`audio/l16; rate=44100; channels=1`)
- `/status`: JSON with device info, mode, resolution, fps, connected clients
- `/control`: POST endpoint for remote zoom/brightness/flip/mode changes
- Broadcast stream controller supports multiple concurrent clients

### 6. Discovery Service (`discovery_service.dart`)
- `bonsoir` package broadcasts `_droidcast._tcp` service
- TXT record includes: video path, audio path, device model, version
- Auto-stops when streaming stops

### 7. Permissions Required (`AndroidManifest.xml`)
```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.CHANGE_WIFI_MULTICAST_STATE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_CAMERA" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MICROPHONE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.WAKE_LOCK" />
```

---

## PC Companion: Detailed Component Design

### 1. Low-Latency MJPEG Reader (`video_receiver.py`)
- Uses `requests.get(stream=True)` NOT `cv2.VideoCapture(url)`
- Manual JPEG SOI (`\xff\xd8`) / EOI (`\xff\xd9`) frame boundary parsing
- Atomic "latest frame" buffer (always overwrites, never queues → zero lag)
- Background daemon thread, thread-safe frame access
- Sub-50ms latency (vs 1-5 seconds with OpenCV's internal FFmpeg buffering)

### 2. Virtual Camera Output (`virtual_camera.py`)
- `pyvirtualcam.Camera(backend="unitycapture", fmt=PixelFormat.BGR)`
- Direct BGR frames from OpenCV → no color conversion overhead
- Resize to target resolution if source differs
- `cam.send()` + `cam.sleep_until_next_frame()` for smooth output

### 3. Audio Receiver & Router (`audio_receiver.py`, `audio_router.py`)
- HTTP streaming reader for PCM audio from `/audio` endpoint
- `sounddevice.OutputStream(device=cable_input_idx)` to play into VB-Cable
- Auto-detect "CABLE Input" device index
- Fallback to default speaker if VB-Cable not installed
- Apps select "CABLE Output" as microphone in Zoom/Teams/Discord

### 4. ADB USB Bridge (`adb_bridge.py`)
- `adbutils` for device detection and port forwarding
- `adbutils` bundles its own `adb.exe` — no Android SDK required
- `device.forward("tcp:8080", "tcp:8080")` for video
- `device.forward("tcp:8081", "tcp:8081")` for audio (if separate port)
- Stream accessible at `http://127.0.0.1:8080/video`

### 5. Auto-Discovery (`mdns_scanner.py`)
- `zeroconf.ServiceBrowser` listens for `_droidcast._tcp.local.`
- Extracts IP, port, device name, capabilities from TXT record
- Callback notifies GUI to show discovered devices

### 6. Driver Manager (`driver_manager.py`)
- Checks Windows registry for Unity Capture DirectShow registration
- First-run: copies DLL from bundle to `%LOCALAPPDATA%\DroidCast\`
- Triggers UAC elevation for `regsvr32 /s` registration
- Checks for VB-Cable installation (scans audio devices for "CABLE Input")
- Shows setup guidance in GUI if drivers missing

### 7. Settings GUI (`settings_window.py`)
- CustomTkinter with dark/light mode auto-detection
- Connection mode selector (WiFi / USB)
- Discovered devices list with auto-refresh
- Manual IP entry fallback
- Resolution dropdown (720p / 1080p / 4K)
- Audio toggle, auto-reconnect toggle
- Driver status indicators (✅/❌)
- Real-time fps and connection status

### 8. System Tray (`tray_icon.py`)
- `pystray.Icon` with dynamic status colors (🟢/🔴/🟡)
- Menu: Show Settings, Disconnect, Exit
- Window minimizes to tray on close (not exit)
- Double-click tray icon to restore window

### 9. Distribution
- **PyInstaller**: `--onedir` mode, bundled with Unity Capture DLL
- **Inno Setup**: Creates Windows installer (.exe)
  - Copies files to `C:\Program Files\DroidCast\`
  - Registers Unity Capture DLL via `regsvr32 /s`
  - Creates desktop & start menu shortcuts
  - Optional auto-start with Windows
  - Clean uninstall (unregisters DLL)

---

## Dependencies

### Flutter (pubspec.yaml)
```yaml
dependencies:
  camera: ^0.11.0
  record: ^5.1.2
  shelf: ^1.4.2
  shelf_router: ^1.1.4
  bonsoir: ^5.1.4
  provider: ^6.1.2
  qr_flutter: ^4.1.0
  permission_handler: ^11.4.0
  device_info_plus: ^11.2.0
  network_info_plus: ^6.1.0
  wakelock_plus: ^1.2.8
```

### Python (requirements.txt)
```
pyvirtualcam>=0.11.0
opencv-python>=4.10.0
requests>=2.32.0
sounddevice>=0.5.0
numpy>=2.0.0
pystray>=0.19.5
Pillow>=11.0.0
zeroconf>=0.135.0
adbutils>=2.8.0
customtkinter>=5.2.2
```

---

## Implementation Phases

### Phase 1: MVP — Camera Streaming via WiFi (~4-5 hours)
1. Flutter project setup with dependencies
2. Camera capture service (CameraX + native YUV→JPEG)
3. HTTP server with MJPEG streaming endpoint
4. PC MJPEG reader (low-latency, requests-based)
5. PC virtual camera output (pyvirtualcam + Unity Capture)
6. Basic UI: camera preview, IP display, start button

### Phase 2: Audio Streaming + Controls (~3 hours)
7. Microphone capture & PCM streaming (Flutter `record`)
8. Audio receiver on PC (sounddevice → VB-Cable)
9. Camera controls: zoom slider, brightness, flip camera
10. Remote control API (POST /control)

### Phase 3: Screen Sharing + Internal Audio (~4 hours)
11. MediaProjection foreground service (native Kotlin)
12. Screen frame capture (ImageReader + row padding handling)
13. Internal audio capture (AudioPlaybackCapture, Android 10+)
14. Mode toggle UI (Camera ↔ Screen switch)
15. Unified HTTP stream for both modes

### Phase 4: Auto-Connect + USB (~2-3 hours)
16. mDNS broadcast from Flutter (bonsoir)
17. Zeroconf discovery on PC
18. USB/ADB bridge (adbutils)
19. Auto-reconnect logic
20. QR code quick connect

### Phase 5: Polish + Distribution (~3 hours)
21. System tray icon with status colors
22. Driver manager (detect/install Unity Capture & VB-Cable)
23. Foreground service for background streaming
24. Settings persistence (JSON config)
25. PyInstaller build → .exe
26. Inno Setup installer with driver registration
27. Error handling & edge cases

**Total estimated: ~16-18 hours**

---

## Verification Plan

| # | Test Case | Mode | Expected Result |
|---|-----------|------|-----------------|
| 1 | Open app, tap Start (Camera) | Camera/WiFi | Preview shows, IP & port displayed |
| 2 | Open `http://<ip>:8080/` in browser | Camera/WiFi | Live camera stream in browser |
| 3 | PC companion connect WiFi | Camera/WiFi | Virtual webcam active |
| 4 | Select "Unity Video Capture" in Zoom | Camera/WiFi | Phone camera visible in Zoom |
| 5 | Enable audio + CABLE Output in Zoom | Camera/WiFi | Phone mic audio in meeting |
| 6 | Switch to Screen mode | Screen/WiFi | Stream switches to phone screen |
| 7 | Play YouTube on phone (Screen mode) | Screen/WiFi | Video + audio streaming to PC |
| 8 | Connect via USB (ADB forward) | Camera/USB | Stream via localhost |
| 9 | Screen share via USB | Screen/USB | Screen + internal audio via USB |
| 10 | Disconnect WiFi during stream | Any | Auto-reconnect when WiFi returns |
| 11 | Minimize PC companion | Any | Tray icon visible, streaming continues |
| 12 | Install DroidCast-Setup.exe fresh | — | Unity Capture registered, app works |
| 13 | Uninstall via Control Panel | — | Driver unregistered, clean removal |
| 14 | Change resolution (720p→1080p→4K) | Camera/WiFi | Stream resolution updates |
| 15 | Internal audio on Android 9 | Screen | Graceful disable, mic fallback |
