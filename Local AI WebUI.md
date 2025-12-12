# Ollama + Open WebUI Local Network Setup  
*(Windows & macOS with Docker Desktop)*

## Overview
This document describes how to set up **Ollama** and **Open WebUI** using **Docker Desktop** so that:

- Ollama runs locally on one machine
- Open WebUI runs in Docker
- Multiple users on the **local network (LAN)** can access the Web UI
- User authentication and model access are handled by Open WebUI
- The setup can be safely enabled when in use and disabled when not needed

This guide is intended for **local/home/lab environments**, not public internet exposure.

---

## Architecture (Mental Model)

```
User Devices (phone, laptop, tablet)
        ↓
Local Network (LAN)
        ↓
Windows / macOS Host
        ↓
Docker Desktop
        ↓
Open WebUI (container)
        ↓
Ollama (local model server)
```

---

## Prerequisites

### Windows
- Windows 10 or 11
- Docker Desktop for Windows
- Ollama for Windows
- PowerShell

### macOS
- macOS (Apple Silicon or Intel)
- Docker Desktop for Mac
- Ollama for macOS
- Terminal

---

## Step 1: Install Ollama

### Windows
1. Download from: https://ollama.com
2. Install normally
3. Verify:
   ```powershell
   ollama --version
   ```
4. Ollama listens on:
   ```
   http://127.0.0.1:11434
   ```

### macOS
```bash
brew install ollama
ollama serve
```

Verify:
```bash
curl http://127.0.0.1:11434
```

---

## Step 2: Install Docker Desktop

### Windows
- Download Docker Desktop for Windows
- Enable WSL integration if prompted
- Start Docker Desktop

### macOS
- Download Docker Desktop for Mac
- Start Docker Desktop

Verify:
```bash
docker version
```

---

## Step 3: Ensure Docker Context is Correct (Windows)

In **PowerShell**:

```powershell
docker context ls
```

Make sure this is selected:

```
* desktop-linux
```

If not:
```powershell
docker context use desktop-linux
```

---

## Step 4: Download Models in Ollama

Download models once (shared by all users):

```bash
ollama pull llama3
ollama pull mistral
```

Verify:
```bash
ollama list
```

---

## Step 5: Run Open WebUI (Docker Desktop)

### Windows (PowerShell)

```powershell
docker run -d `
  -p 8080:8080 `
  -v open-webui:/app/backend/data `
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 `
  --name open-webui `
  --restart always `
  ghcr.io/open-webui/open-webui:main
```

### macOS (Terminal)

```bash
docker run -d \
  -p 8080:8080 \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

Verify:
```bash
docker ps
```

You should see:
```
0.0.0.0:8080->8080/tcp
```

---

## Step 6: Access the Web UI

### Same machine
```
http://localhost:8080
```

### Other devices on LAN
```
http://<HOST_IP>:8080
```

Example:
```
http://192.168.x.50:8080
```

---

## Step 7: Windows Network Profile (Important)

To allow LAN access:

1. Settings → Network & Internet
2. Click active connection
3. Set **Network profile** to:
   ```
   Private
   ```

Public networks block inbound LAN traffic by default.

---

## Step 8: Windows Firewall Rule (If Needed)

Run PowerShell as Administrator:

```powershell
netsh advfirewall firewall add rule name="Open WebUI 8080" dir=in action=allow protocol=TCP localport=8080
```

---

## Step 9: User Accounts & Authentication

- First account created = **Admin**
- Admin Panel → Settings:
  - Enable Signups
  - Require Admin Approval (recommended)
- Admin Panel → Users:
  - Approve users
  - Promote admins if needed

User data is persisted via:
```
-v open-webui:/app/backend/data
```

---

## Step 10: Model Access Control

Admin Panel → Models:
- Enable desired models
- Control visibility per role
- Set a default model
- Disable user model pulling (recommended)

All users share the same Ollama model pool.

---

## Operational Usage (On / Off)

### Start Everything
1. Start Docker Desktop
2. Ensure Ollama is running
3. Start container (if stopped):
   ```bash
   docker start open-webui
   ```
4. Ensure network profile is **Private**

---

### Stop Everything (Recommended When Not in Use)

```bash
docker stop open-webui
```

Optional:
- Quit Docker Desktop
- Stop Ollama
- Switch network profile back to **Public** (Windows)

---

## Security Notes

- This setup is **LAN-only**
- Traffic is HTTP, not HTTPS
- Do NOT expose port 8080 to the internet
- For external access, use a reverse proxy + HTTPS

---

## Cleanup / Removal

Stop and remove container:
```bash
docker stop open-webui
docker rm open-webui
```

Remove persistent data:
```bash
docker volume rm open-webui
```


