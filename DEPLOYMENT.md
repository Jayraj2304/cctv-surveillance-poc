# Deployment Guide

This guide explains how to deploy and test this RTSP/WebRTC surveillance POC on a different system.

## Prerequisites

1. **Docker**: Ensure Docker Desktop or Docker Engine is installed and running on the target machine.
2. **Git**: Installed to clone the repository.
    * `docker-compose.yml`
    * `go2rtc.yaml`
    * `index.html`
    * `sample-30s.mp4` (or your own video file)

## Installation Steps

1. **Clone the Repository**:

    ```bash
    git clone https://github.com/Jayraj2304/cctv-surveillance-poc.git
    cd cctv-surveillance-poc
    ```

2. **Open Terminal**: Navigate to the folder in your terminal/command prompt.
3. **Find the IP Address**:
    * Run `ipconfig` (Windows) or `ifconfig` (Linux/Mac) to find the Local IP address of the new machine (e.g., `192.168.1.50`).
4. **Update Configuration**:
    * Open `go2rtc.yaml` in a text editor.
    * Update the `candidates` section with the **new** IP address.

    ```yaml
    webrtc:
      listen: ":8555"
      candidates:
        - 192.168.1.50:8555    <-- Change this to the NEW machine's IP
        - 127.0.0.1:8555
    ```

5. **Start the Server**:
    Run the following command to start the Docker container:

    ```bash
    docker-compose up -d
    ```

## How to Test

### Option A: Test on the Same Machine (Localhost)

1. Open `index.html` in your browser.
2. Leave the **Server** field as `localhost:1984`.
3. Click **Play**.

### Option B: Test from Another Device (e.g., Phone or Laptop)

1. Ensure both devices are on the same Wi-Fi/Network.
2. On your phone/laptop, you cannot easily open the `index.html` file directly since it is on the server's disk.
    * **Solution**: You can host the `index.html` using a simple web server (like `python -m http.server 8000`) OR just copy `index.html` to your laptop.
3. Open `index.html` on your viewing device.
4. In the **Server** field, enter the **IP address** of the machine running Docker (e.g., `192.168.1.50:1984`).
5. Click **Play**.

## Troubleshooting

* **Connection Failed**: Check if the firewall is blocking ports `8555`, `8554`, or `1984`.
* **Logs**: Check the logs on the server machine using `docker-compose logs go2rtc`.
