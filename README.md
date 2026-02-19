# Gopher — Shared Infrastructure (Traefik Reverse Proxy)

This repository contains the **Traefik** reverse proxy configuration used to serve multiple web applications on a single host IP with automatic SSL via Let's Encrypt.

## Architecture

```
Internet → Port 80/443 → [Traefik Container] → proxy-net → [App Containers]
```

- **Traefik** owns host ports 80 and 443.
- **App containers** (e.g., `star-wars-explorer`) do NOT bind to any host ports. They register with Traefik via Docker labels and connect through the shared `proxy-net` Docker network.
- **SSL certificates** are automatically provisioned and renewed by Traefik using the TLS-ALPN-01 challenge.

## Prerequisites

- Docker installed on the host.
- A `docker-compose` binary (standalone v2 or Docker Compose plugin).

## Deployment

### 1. Create the shared Docker network (one-time)

```bash
docker network create proxy-net
```

### 2. Start Traefik

```bash
cd gopher
docker compose up -d    # or ./docker-compose up -d
```

Traefik will start listening on ports 80 (HTTP → HTTPS redirect) and 443 (SSL).

## Adding a New Application

Each application lives in its own repo/directory with its own `docker-compose.yml`. To register with Traefik, the app needs:

1. **Traefik labels** with a **unique router name**.
2. Connection to the `proxy-net` network.

### Template `docker-compose.yml` for a new app

```yaml
version: '3'

services:
  web:
    image: nginx:alpine   # or your app image
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.APPNAME.rule=Host(`APPNAME.dreamgearweb.com`)"
      - "traefik.http.routers.APPNAME.entrypoints=websecure"
      - "traefik.http.routers.APPNAME.tls.certresolver=myresolver"
    networks:
      - proxy-net
    restart: always

networks:
  proxy-net:
    external: true
```

> **Important**: Replace `APPNAME` (in all 4 label lines) with a unique identifier for the app (e.g., `starwars`, `blog`, `api`).

## Renaming a Domain

Edit the `Host()` rule in the app's `docker-compose.yml` and re-deploy:

```yaml
- "traefik.http.routers.starwars.rule=Host(`newname.dreamgearweb.com`)"
```

Traefik will automatically request a new certificate for the new domain.

## Troubleshooting

| Issue | Fix |
|---|---|
| 504 Gateway Timeout | Ensure the app container is on the `proxy-net` network |
| Certificate not issued | Check `docker logs traefik` for ACME errors |
| Port 80/443 in use | Stop any other web server (e.g., `systemctl stop nginx`) |

## Current Sites

| Domain | App Repo | Router Name |
|---|---|---|
| `one.dreamgearweb.com` | `dreamgear/star-wars-explorer` | `starwars` |
