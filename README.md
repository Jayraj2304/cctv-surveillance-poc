# CCTV Surveillance PoC

A Proof of Concept for low-latency RTSP streaming using [go2rtc](https://github.com/AlexxIT/go2rtc).
This project demonstrates how to stream video (simulated via file) to a web browser using WebRTC.

## Features

- **Dockerized Setup**: Runs `go2rtc` with FFmpeg in a container.
- **WebRTC Player**: Custom `index.html` player with:
  - Low-latency streaming.
  - Dynamic RTSP link input.
  - Diagnostic logging.
  - configurable Server URL for remote testing.
- **RTSP Simulation**: Streams a sample video file as a loop.

## Quick Start

1. **Clone the Repo**:

    ```bash
    git clone <your-repo-url>
    cd cctv-surveillance-poc
    ```

2. **Run with Docker**:

    ```bash
    docker-compose up -d
    ```

3. **Open Player**:
    open `index.html` in your browser.

## Deployment

See [DEPLOYMENT.md](DEPLOYMENT.md) for instructions on running this on a different machine.
