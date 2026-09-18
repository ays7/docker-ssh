# ays77/ssh-tunnels-server

[![Docker Pulls](https://img.shields.io/docker/pulls/ays77/ssh-tunnels-server)](https://hub.docker.com/r/ays77/ssh-tunnels-server)
[![Image Size](https://img.shields.io/badge/size-~18MB-brightgreen)](https://hub.docker.com/r/ays77/ssh-tunnels-server)
[![GitHub](https://img.shields.io/badge/GitHub-ays7%2Fssh--tunnels--server-blue?logo=github)](https://github.com/ays7/ssh-tunnels-server)

Hardened, lightweight (~18MB) Alpine Linux-based SSH tunnel server designed exclusively for secure port-forwarding bastions into clusters and private networks.

---

## Features

- **Zero-Shell Security**: Interactive shells and command execution are completely blocked (`nologin-tunnel`).
- **Safe Tunnel Holding**: Connect with or without `-N` — holding sessions open cleanly until disconnected.
- **Strict Port Forwarding**: Local (`-L`) and Dynamic SOCKS (`-D`) forwarding only; reverse forwarding (`-R`) is disabled.
- **Destination Whitelisting**: Restrict access to specific targets using `SSH_PERMIT_OPEN`.
- **High Performance**: Optimized for maximum throughput and low latency (IPv4-first, no CS1 scavenger packet tagging, hardware AES-GCM acceleration).
- **Multi-Arch**: Supports `linux/amd64` and `linux/arm64`.

---

## Quick Start

### 1. Run with Docker CLI

```bash
docker run -d \
  --name ssh-tunnel \
  --restart unless-stopped \
  -p 2222:2222 \
  -e ALLOWED_IPS="AllowUsers tunnel@*" \
  -e AUTHORIZED_KEYS="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5... user@client" \
  ays77/ssh-tunnels-server:latest
```

### 2. Run with Docker Compose

```yaml
services:
  ssh-tunnel:
    image: ays77/ssh-tunnels-server:latest
    container_name: ssh-tunnel
    restart: unless-stopped
    ports:
      - "2222:2222"
    environment:
      - ALLOWED_IPS=AllowUsers tunnel@*
      - AUTHORIZED_KEYS=ssh-ed25519 AAAAC3NzaC1lZDI1NTE5... user@client
      - SSH_PERMIT_OPEN=database:3306 redis:6379   # Optional: limit tunnel targets
    volumes:
      - ./ssh_host_keys:/etc/ssh/ssh_host_keys     # Optional: persist host keys
```

---

## Connecting & Creating Tunnels

### Local Port Forwarding (`-L`)
Forward local port `3306` through the tunnel to internal service `database:3306`:
```bash
ssh -p 2222 -L 3306:database:3306 tunnel@<server-ip>
```
*(Optionally append `-N` to skip holding session output)*

### Dynamic SOCKS5 Proxy (`-D`)
Route browser or tool traffic through your private network:
```bash
ssh -p 2222 -D 1080 tunnel@<server-ip>
```

---

## Environment Variables

| Variable | Default | Description |
| :--- | :--- | :--- |
| `ALLOWED_IPS` | **Required** | OpenSSH `AllowUsers` directive (e.g. `AllowUsers tunnel@*` or `AllowUsers tunnel@192.168.1.*`) |
| `AUTHORIZED_KEYS` | **Required\*** | Public SSH key string (e.g. `ssh-ed25519 ...`) |
| `SSH_USER` | `tunnel` | Non-root SSH username |
| `SSH_PORT` | `2222` | Listening SSH port inside container |
| `SSH_PERMIT_OPEN` | *(all)* | Space/comma-delimited allowed destinations (e.g. `db:5432 internal:80`) |
| `SSH_BANNER` | `"Connected..."` | Pre-auth banner text displayed upon connection (`none` to disable) |
| `PUID` / `PGID` | `9999` / `9999` | User ID and Group ID for the SSH tunnel user |
| `DEBUG` | `false` | Set to `true` to enable verbose SSHD debugging logs |

*\* Alternatively, mount your keys to `/authorized_keys` or `/home/tunnel/.ssh/authorized_keys`.*

---

## Source & Documentation

For complete documentation, security benchmarks, and advanced configuration, visit the [GitHub Repository](https://github.com/ays7/ssh-tunnels-server).
