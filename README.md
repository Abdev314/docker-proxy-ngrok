# Docker Image Pull Through HTTP Proxy for Isolated Linux Servers & Lightweight Proxy Gateway Setup

[![Debian](https://img.shields.io/badge/Debian-13-A81D33?logo=debian&logoColor=white)](https://www.debian.org/doc/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![Tinyproxy](https://img.shields.io/badge/Proxy-Tinyproxy-green)](https://tinyproxy.github.io/)
[![ngrok](https://img.shields.io/badge/ngrok-Tunnel-1F1E37?logo=ngrok&logoColor=white)](https://ngrok.com/docs)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/docs/15/)
[![pgAdmin](https://img.shields.io/badge/pgAdmin-4-orange)](https://www.pgadmin.org/docs/)

> A practical infrastructure solution for restricted network environments — enabling Docker image pulling and public app exposure via a LAN proxy gateway.

---

## Overview

This project documents how we solved network access issues affecting Docker image pulling and external service connectivity in a restricted network environment.

As a developer based in China, access to external services such as Docker Hub and ngrok infrastructure is often unstable or partially blocked depending on ISP routing and network conditions.

To solve this without installing VPN software on production servers, we built a **local proxy gateway** using a Debian laptop (personal machine) that already had VPN access. This proxy routed Docker traffic — and optionally ngrok traffic — from an isolated Linux server.

The production environment was a **Debian 13 server running in TTY** (headless mode, no GUI) for stability and performance.

---

## System Architecture

```
                    Internet
                        |
                      VPN
                        |
        +--------------------------------+
        | Debian Laptop (Proxy Gateway)  |
        | Personal Debian KDE machine    |
        | tinyproxy (port 8888)          |
        | IP: 192.168.10.11              |
        +--------------------------------+
                        |
                Local Network (LAN)
                        |
        +--------------------------------+
        | Linux Server (Docker Host)     |
        | Debian 13 (TTY, no GUI)        |
        | IP: 192.168.10.8               |
        +--------------------------------+
                        |
                     Docker
```

---

## Part 1 — Proxy Gateway Setup (Debian Laptop)

### 1. System Context

The proxy gateway was implemented on a personal Debian laptop running KDE.

- No OS modification or reinstall was performed
- The system remained a standard desktop environment
- It was used as a network proxy node due to its VPN connectivity

### 2. Install Tinyproxy

```bash
sudo apt install tinyproxy
```

### 3. Configure Tinyproxy

```bash
sudo nano /etc/tinyproxy/tinyproxy.conf
```

Key settings:

- Listen on all interfaces
- Allow LAN subnet: `192.168.10.0/24`
- Port: `8888`

### 4. Start and Enable the Service

```bash
sudo systemctl restart tinyproxy
sudo systemctl enable tinyproxy
```

Verify the port is open:

```bash
ss -lnt | grep 8888
```

Expected output:

```
0.0.0.0:8888 LISTEN
[::]:8888    LISTEN
```

✅ Proxy successfully exposed to LAN

### 5. Verify Network Interface

```bash
ip a
```

Confirm the laptop is reachable at `192.168.10.11/24`.

---

## Part 2 — Linux Server Configuration (Debian 13 TTY)

### 1. Server Context

The production server was:

- Debian 13
- Running in TTY (no GUI)
- Lightweight and production-focused

### 2. Initial Problem — Docker Pull Failure

```bash
docker compose up -d
```

Errors:

```
failed to resolve reference "docker.io/library/postgres:15"
i/o timeout
```

```bash
ping registry-1.docker.io
# ❌ 100% packet loss
```

### 3. Root Cause

- Docker Hub access blocked or unstable due to ISP routing
- IPv6 resolution causing timeouts
- No reliable direct path to external registries

### 4. Set Shell Proxy (for Testing)

```bash
export http_proxy=http://192.168.10.11:8888
export https_proxy=http://192.168.10.11:8888
```

Verify traffic is routed through the proxy:

```bash
curl https://ifconfig.me
```

✅ Confirmed traffic routes through the VPN-enabled laptop

### 5. Configure Docker System Proxy

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo nano /etc/systemd/system/docker.service.d/http-proxy.conf
```

```ini
[Service]
Environment="HTTP_PROXY=http://192.168.10.11:8888"
Environment="HTTPS_PROXY=http://192.168.10.11:8888"
```

Apply the changes:

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 6. Result

```bash
docker compose up -d
```

✅ `postgres:15` pulled successfully  
✅ `pgadmin4` pulled successfully  
✅ Containers running — confirmed with `docker ps`

---

## Part 3 — Docker Compose Stack

```yaml
services:
  postgres:
    image: postgres:15
    container_name: postgres15
    restart: unless-stopped
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password123
      POSTGRES_DB: db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  pgadmin:
    image: dpage/pgadmin4
    container_name: pgadmin4
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: youremail@gmail.com
      PGADMIN_DEFAULT_PASSWORD: password
    ports:
      - "5050:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin

volumes:
  postgres_data:
  pgadmin_data:
```

---

## Part 4 — Ngrok Deployment (Django Exposure via Tinyproxy Bridge)

For the ngrok setup, we installed ngrok on the **Debian server** (`192.168.10.8`) and used the existing Tinyproxy gateway on the laptop as a proxy bridge — giving ngrok a route to the internet through the laptop's VPN connection.

The goal was to expose a **Django application** running locally on the **Debian laptop** (`192.168.10.11:8000`) to the internet.

### 1. Architecture

```
         Internet
             |
         ngrok cloud
             |
      [HTTPS_PROXY tunnel]
             |
  Debian Server (192.168.10.8)       
     ngrok → Tinyproxy (:8888)
             |
           LAN
             |
  Debian Laptop (192.168.10.11)     
     Django App (:8000)
     Tinyproxy (:8888) + VPN
```

Traffic flow: `Internet → ngrok cloud → server → LAN → Django on laptop`

### 2. Configure Proxy for ngrok (on the Server)

ngrok itself has no direct internet access from the server. It reaches the ngrok cloud **through Tinyproxy** running on the laptop:

```bash
export HTTPS_PROXY=http://192.168.10.11:8888
```

Or set it permanently in the shell profile (`~/.bashrc` or `~/.profile`) so it persists across sessions.

### 3. Configure Ngrok Authentication (on the Server)

```bash
ngrok config add-authtoken <your-token>
ngrok config check
```

### 4. Expose the Django App

From the server, point ngrok at the Django application running on the laptop's LAN address:

```bash
ngrok http 192.168.10.11:8000
```

This creates a public HTTPS tunnel that forwards external traffic across the LAN to Django on the laptop.

### 5. Run as a Systemd Service (on the Server)

Create a service file for persistent operation:

```bash
sudo nano /etc/systemd/system/ngrok.service
```

```ini
[Unit]
Description=ngrok Tunnel
After=network.target

[Service]
User=gnu
Environment="HTTPS_PROXY=http://192.168.10.11:8888"
ExecStart=/usr/local/bin/ngrok http 192.168.10.11:8000
Restart=always

[Install]
WantedBy=multi-user.target
```

> **Note:** The `HTTPS_PROXY` environment variable is set directly in the service definition so ngrok can reach the internet even when launched by systemd (which does not inherit your shell environment).

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable ngrok
sudo systemctl start ngrok
```

### 6. Verify the Tunnel

```bash
curl -s http://127.0.0.1:4040/api/tunnels
```

A valid public HTTPS URL in the response confirms:

✅ ngrok running on the server with proxy access via Tinyproxy  
✅ Django reachable across the LAN from the server  
✅ Public HTTPS tunnel successfully created  
✅ External access to the Django application is working  

---

## Final Result

| Layer | Status |
|---|---|
| **Docker** | ✅ PostgreSQL deployed · ✅ pgAdmin deployed · ✅ Docker Hub access restored via LAN proxy |
| **Network** | ✅ Stable proxy gateway via personal laptop · ✅ VPN traffic routed through laptop only |
| **Application** | ✅ Django exposed via ngrok · ✅ Public HTTPS tunnel working |

---

## Key Lessons Learned

- **Docker Hub failures are usually network routing issues**, not Docker bugs — always check connectivity first
- **A LAN proxy gateway is more reliable** than installing VPN software directly on production servers
- **Debian TTY servers are ideal for production** — lightweight, stable, no GUI overhead
- **Tinyproxy is simple and effective** for HTTP proxy needs on a LAN
- **ngrok works best without proxy interference** — run it independently when possible
---

## Conclusion

This project demonstrates a practical infrastructure solution for restricted network environments. By using:

- A **personal Debian KDE laptop** as a VPN-enabled proxy gateway (Tinyproxy + Django app)
- A **Debian 13 TTY server** for production workloads (Docker, ngrok)

We successfully enabled:

- Reliable Docker image pulling through a LAN proxy
- Stable container deployment (PostgreSQL, pgAdmin)
- Public exposure of local services via ngrok

This approach avoids modifying production server network configurations while maintaining a scalable and maintainable architecture.
