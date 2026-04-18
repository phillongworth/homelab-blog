---
title: "Deploying Actual Budget on Docker with Nginx HTTPS"
date: 2026-04-18T00:00:00Z
draft: false
tags: ["docker", "nginx", "actual-budget", "https", "self-hosted"]
categories: ["Guides"]
description: "Deploying Actual Budget via Docker Compose, solving the SharedArrayBuffer HTTPS requirement, removing Nextcloud, and setting up Nginx as a reverse proxy with a self-signed certificate."
showToc: true
---

This post documents deploying [Actual Budget](https://actualbudget.org/) — a local-first personal finance app — on a Linux homelab server using Docker Compose and Nginx. The main challenge turned out to be a browser security requirement that forces HTTPS even on a LAN, which led to reworking the reverse proxy setup and taking the opportunity to remove Nextcloud at the same time.

---

## What We Built

By the end of this session:

- Actual Budget running in Docker, managed by Compose with `restart: unless-stopped`
- Nginx reverse proxy serving it over HTTPS on port 443
- A fresh self-signed TLS certificate for the LAN IP
- Nextcloud fully removed (web files, database, and certs)
- HTTP on port 80 redirecting to HTTPS automatically

---

## Prerequisites

- Docker and Docker Compose installed (`docker compose version` to check)
- Nginx installed and running
- `sudo` access on the server

---

## 1. Create the Directory and Compose File

```bash
mkdir -p ~/actual-budget/data
```

`~/actual-budget/docker-compose.yml`:

```yaml
services:
  actual-budget:
    image: actualbudget/actual-server:latest
    container_name: actual-budget
    ports:
      - "127.0.0.1:5006:5006"
    volumes:
      - ./data:/data
    restart: unless-stopped
```

Binding to `127.0.0.1:5006` means the container is only reachable from the host itself — Nginx proxies inbound requests to it, so there is no reason to expose it on the LAN directly.

Set ownership of the data directory to the `node` user the image runs as (UID/GID 1000):

```bash
sudo chown -R 1000:1000 ~/actual-budget/data
```

Pull and start:

```bash
cd ~/actual-budget
docker compose pull
docker compose up -d
```

---

## 2. The SharedArrayBuffer Problem

Visiting `http://192.168.x.x:5006` in a browser produces:

> **Fatal Error** — Actual requires access to SharedArrayBuffer in order to function properly. If you're seeing this error, either your browser does not support SharedArrayBuffer, or your server is not sending the appropriate headers, or you are not using HTTPS.

Checking the response headers confirms the server is doing its part correctly:

```bash
curl -sI http://localhost:5006
```

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

The issue is a browser security restriction, not a misconfiguration. `SharedArrayBuffer` requires a [secure context](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts). On non-`localhost` origins that means **HTTPS** — the `COOP`/`COEP` headers alone are not enough. The fix is a TLS-terminating reverse proxy.

---

## 3. Setting Up Nginx with a Self-Signed Certificate

### Generate the certificate

```bash
sudo mkdir -p /etc/ssl/actual-budget
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout /etc/ssl/actual-budget/actual.key \
  -out /etc/ssl/actual-budget/actual.crt \
  -subj "/CN=192.168.x.x" \
  -addext "subjectAltName=IP:192.168.x.x"
```

The `subjectAltName` extension is required — modern browsers ignore the `CN` field for IP addresses without it.

### Create the Nginx site

`/etc/nginx/sites-available/actual-budget`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name 192.168.x.x;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name 192.168.x.x;

    ssl_certificate     /etc/ssl/actual-budget/actual.crt;
    ssl_certificate_key /etc/ssl/actual-budget/actual.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:5006;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Required for SharedArrayBuffer — pass through from upstream
        # and re-add so they survive proxy_hide_header
        proxy_hide_header Cross-Origin-Opener-Policy;
        proxy_hide_header Cross-Origin-Embedder-Policy;
        add_header Cross-Origin-Opener-Policy "same-origin" always;
        add_header Cross-Origin-Embedder-Policy "require-corp" always;

        # WebSocket support (Actual uses WS for sync)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Enable and reload:

```bash
sudo ln -s /etc/nginx/sites-available/actual-budget /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

> **Browser warning:** you will see an untrusted certificate warning the first time you visit — self-signed certs are not in the browser's trust store. Click through once. If you want to suppress the warning permanently, install Caddy's `tls internal` CA, or use [mkcert](https://github.com/FiloSottile/mkcert) to generate a locally trusted certificate.

---

## 4. Removing Nextcloud

Nextcloud was previously installed on this server as a manual PHP install, with its own Nginx block and self-signed cert occupying port 443. With nothing worth keeping (no user data), it was removed cleanly.

### Disable the Nginx block

```bash
sudo rm /etc/nginx/sites-enabled/nextcloud
sudo nginx -t && sudo systemctl reload nginx
```

### Drop the database

```bash
sudo mysql -e "DROP DATABASE IF EXISTS nextcloud;"
```

### Delete the web files and cert

```bash
sudo rm -rf /var/www/html/nextcloud
sudo rm -rf /etc/ssl/nextcloud
sudo rm /etc/nginx/sites-available/nextcloud
```

PHP-FPM (`php8.3-fpm`) was left running as it may be needed by other services.

---

## 5. Validate the Deployment

```bash
# Container running?
cd ~/actual-budget && docker compose ps

# Port bound on host?
ss -tlnp | grep 5006

# No startup errors?
docker compose logs --tail=30

# Nginx serving HTTPS?
curl -sk https://192.168.x.x | head -c 100
```

Expected: container `Up`, port `127.0.0.1:5006` listening, logs showing `Listening on :::5006...`, and a valid HTML response from curl.

---

## Useful References

- [Actual Budget docs](https://actualbudget.org/docs/)
- [MDN — Secure Contexts](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts)
- [MDN — SharedArrayBuffer security requirements](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer#security_requirements)
- [Nginx reverse proxy docs](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
