# CCTV Surveillance Platform - Architecture Document

## Executive Summary

This document describes the architecture and core concepts behind the CCTV Surveillance Platform. It is intended to serve as a reference for developers building a production-ready, high-density surveillance dashboard capable of streaming hundreds of cameras with minimal latency.

---

## 1. Core Idea

### Problem Statement

Traditional CCTV systems rely on:

- **VMS Software**: Proprietary, expensive, and platform-locked.
- **RTSP Players**: VLC, ffplay - require installation, high CPU usage, not web-native.
- **Browser Plugins**: Flash (dead), ActiveX (dead) - security nightmares.

### Our Solution

A **browser-native, low-latency surveillance dashboard** that:

1. Ingests RTSP streams from IP cameras.
2. Converts them to WebRTC in real-time.
3. Displays them in the browser with sub-second latency.
4. Scales to 100+ cameras on a single page.

---

## 2. Technology Stack

| Layer          | Technology                | Purpose                                      |
|----------------|---------------------------|----------------------------------------------|
| **Frontend**   | HTML/CSS/JS (Vanilla)     | Lightweight, no framework overhead           |
| **Streaming**  | WebRTC                    | Ultra-low latency video delivery to browser  |
| **Gateway**    | go2rtc                    | Protocol conversion (RTSP → WebRTC)          |
| **Encoding**   | FFmpeg (bundled in go2rtc)| Hardware/software transcoding when needed    |
| **Container**  | Docker                    | Portable deployment, includes all dependencies|

---

## 3. System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              BROWSER                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     WebRTC Player (index.html)                    │   │
│  │  - Connects via WebSocket for signaling                          │   │
│  │  - Receives video/audio via WebRTC (UDP/TCP)                      │   │
│  │  - Renders in <video> element                                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ WebSocket (ws://host:1984/api/ws)
                                    │ + WebRTC Media (UDP/TCP :8555)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           GO2RTC SERVER                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  API Module │  │ RTSP Client │  │ WebRTC SFU  │  │ FFmpeg Exec │    │
│  │  :1984      │  │  (Ingest)   │  │   :8555     │  │ (Transcode) │    │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
│         │                │                │                │            │
│         └────────────────┴────────────────┴────────────────┘            │
│                              Stream Manager                              │
│                    (Routes streams between producers/consumers)          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ RTSP (rtsp://camera-ip/stream)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           IP CAMERAS                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Camera 1   │  │  Camera 2   │  │  Camera 3   │  │  Camera N   │    │
│  │  RTSP/H.264 │  │  RTSP/H.265 │  │  ONVIF      │  │  Tapo/Wyze  │    │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Data Flow

### Step-by-Step: How a User Views a Camera

1. **User Action**: Opens `index.html`, enters an RTSP URL, clicks "Play".

2. **Stream Registration** (API Call):

   ```
   PUT http://localhost:1984/api/streams?name=cam1&src=rtsp://camera/stream
   ```

   - go2rtc stores this stream configuration in memory.

3. **WebSocket Signaling**:

   ```
   Browser → WS → go2rtc: { type: "webrtc/offer", value: <SDP> }
   go2rtc → WS → Browser: { type: "webrtc/answer", value: <SDP> }
   ```

   - Browser and go2rtc exchange Session Description Protocol (SDP) messages.
   - They negotiate codecs, ICE candidates, and transport.

4. **ICE Candidate Exchange**:
   - Both sides share their network addresses (candidates).
   - STUN servers help discover public IP if behind NAT.

5. **Media Flow**:

   ```
   Camera → RTSP → go2rtc → WebRTC → Browser
   ```

   - go2rtc pulls the RTSP stream from the camera.
   - Repackages RTP packets into WebRTC format.
   - Streams directly to browser (ideally via UDP for lowest latency).

6. **Rendering**:
   - Browser receives RTP-over-WebRTC.
   - Decodes video using hardware (GPU) if available.
   - Displays in `<video>` element.

---

## 5. Key Components Explained

### 5.1 go2rtc

**What it is**: An open-source media server written in Go.

**Why we use it**:

- Zero-copy streaming (no transcoding if codec matches).
- Supports 30+ camera protocols (RTSP, ONVIF, Tapo, Wyze, etc.).
- Built-in WebRTC SFU (Selective Forwarding Unit).
- Tiny footprint (~20MB binary, ~50MB Docker image).

**Key Modules**:

| Module      | Port  | Purpose                                    |
|-------------|-------|--------------------------------------------|
| API         | 1984  | HTTP REST API for stream management        |
| RTSP Server | 8554  | Re-publishes streams as RTSP               |
| WebRTC      | 8555  | WebRTC signaling and media                 |
| HLS         | 1984  | Fallback streaming via HTTP Live Streaming |
| MJPEG       | 1984  | Fallback for legacy browsers               |

### 5.2 WebRTC

**What it is**: A browser-native protocol for real-time communication.

**Advantages**:

- Sub-second latency (50-500ms typical).
- Native browser support (no plugins).
- Works behind NAT with STUN/TURN.

**Limitations**:

- Codec support varies by browser:
  - H.264: Universal support.
  - H.265: Safari only (Chrome requires flag).
  - VP8/VP9: Chrome/Firefox.

### 5.3 RTSP

**What it is**: Real Time Streaming Protocol - the standard for IP cameras.

**Typical URL format**:

```
rtsp://username:password@192.168.1.100:554/stream1
```

**Variants**:

- RTSP over TCP (default, reliable).
- RTSP over UDP (lower latency, may lose packets).
- RTSPS (encrypted, port 322 typically).

---

## 6. Scaling Considerations

### 6.1 Single Server Limits

- **CPU**: ~10-20 cameras per core (passthrough, no transcoding).
- **Bandwidth**: Each 1080p stream ≈ 4-8 Mbps.
- **Memory**: ~50MB per active stream.

### 6.2 Horizontal Scaling

```
                    ┌─────────────┐
                    │  Load Balancer │
                    └───────┬───────┘
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
     ┌───────────┐   ┌───────────┐   ┌───────────┐
     │ go2rtc #1 │   │ go2rtc #2 │   │ go2rtc #3 │
     │ Cams 1-50 │   │ Cams 51-100│   │ Cams 101+ │
     └───────────┘   └───────────┘   └───────────┘
```

- Assign camera groups to different go2rtc instances.
- Use a load balancer or reverse proxy (nginx, Traefik).

### 6.3 GPU Acceleration

For transcoding (H.265 → H.264):

- NVIDIA: NVENC.
- Intel: QuickSync.
- AMD: VCE.

Configure in `go2rtc.yaml`:

```yaml
ffmpeg:
  h264: "-c:v h264_nvenc -preset llhq"
```

---

## 7. Future Development Roadmap

### Phase 1: MVP (Current)

- [x] Single stream playback.
- [x] Dynamic RTSP input.
- [x] Docker deployment.

### Phase 2: Dashboard

- [ ] Multi-camera grid layout (2x2, 3x3, 4x4).
- [ ] Camera management UI (add/edit/delete).
- [ ] Persistent stream configuration (database).

### Phase 3: Features

- [ ] PTZ (Pan-Tilt-Zoom) control via ONVIF.
- [ ] Motion detection alerts.
- [ ] Recording (MP4 clips on demand).
- [ ] User authentication (login system).

### Phase 4: Scale

- [ ] Multi-server deployment.
- [ ] Cloud recording (S3, Azure Blob).
- [ ] Mobile app (React Native / Flutter).

---

## 8. API Reference

### 8.1 Streams

**List all streams**:

```
GET http://localhost:1984/api/streams
```

**Add a stream**:

```
PUT http://localhost:1984/api/streams?name=cam1&src=rtsp://...
```

**Delete a stream**:

```
DELETE http://localhost:1984/api/streams?name=cam1
```

### 8.2 WebRTC Signaling

**Endpoint**: `ws://localhost:1984/api/ws?src=<stream_name>`

**Messages**:

| Type               | Direction     | Payload                       |
|--------------------|---------------|-------------------------------|
| webrtc/offer       | Client → Server | SDP offer string              |
| webrtc/answer      | Server → Client | SDP answer string             |
| webrtc/candidate   | Bidirectional   | ICE candidate string          |

---

## 9. Security Considerations

### 9.1 Current State (PoC)

- **No authentication**: Anyone with network access can view streams.
- **No encryption**: WebSocket and API calls are HTTP (not HTTPS).

### 9.2 Production Recommendations

1. **HTTPS Everywhere**: Put behind nginx with SSL certificates.
2. **API Authentication**: Add JWT or session-based auth.
3. **Network Isolation**: Keep cameras on a VLAN, separate from user network.
4. **Firewall Rules**: Only expose required ports (1984, 8555).

---

## 10. Glossary

| Term       | Definition                                                  |
|------------|-------------------------------------------------------------|
| **RTSP**   | Real Time Streaming Protocol - camera video delivery.       |
| **WebRTC** | Web Real-Time Communication - browser-native streaming.     |
| **SFU**    | Selective Forwarding Unit - routes media without mixing.    |
| **SDP**    | Session Description Protocol - describes media capabilities.|
| **ICE**    | Interactive Connectivity Establishment - NAT traversal.     |
| **STUN**   | Session Traversal Utilities for NAT - discovers public IP.  |
| **TURN**   | Traversal Using Relays around NAT - relay for blocked UDP.  |
| **HLS**    | HTTP Live Streaming - fallback protocol, higher latency.    |
| **MJPEG**  | Motion JPEG - fallback, one JPEG frame at a time.           |

---

## 11. File Structure

```
cctv-surveillance-poc/
├── docker-compose.yml    # Container orchestration
├── go2rtc.yaml           # go2rtc configuration
├── index.html            # Single-page web player
├── sample-30s.mp4        # Test video file
├── go2rtc/               # go2rtc source code (for custom tweaks)
├── README.md             # Quick start guide
├── DEPLOYMENT.md         # Multi-system deployment guide
├── ARCHITECTURE.md       # This document
└── WORK_DONE.md          # Development log
```

---

## 12. References

- [go2rtc GitHub](https://github.com/AlexxIT/go2rtc)
- [WebRTC API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API)
- [RTSP Specification (RFC 7826)](https://datatracker.ietf.org/doc/html/rfc7826)
- [Pion WebRTC (Go library)](https://github.com/pion/webrtc)

---

*Document Version: 1.0*  
*Last Updated: 2026-02-03*
