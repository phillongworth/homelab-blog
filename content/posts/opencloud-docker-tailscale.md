---
title: Self-Hosting OpenCloud with Docker and Tailscale
date: 2026-04-21T00:00:00Z
draft: false
tags:
  - docker
  - tailscale
  - nginx
  - opencloud
  - self-hosted
  - file-sync
categories:
  - Guides
description: Setting up OpenCloud as a self-hosted Dropbox alternative using Docker Compose, with Tailscale providing secure remote access and no open router ports.
showToc: true
---

This post documents setting up [OpenCloud](https://opencloud.eu/) — an open-source file sync and sharing platform — on a home Linux server using Docker Compose. The interesting challenge was getting secure remote access without opening any ports on the router, which led to using Tailscale as a zero-config WireGuard VPN instead of the usual port-forwarding approach.

---

## What We Built

- OpenCloud running in Docker, persisting data to `/home/phil/opencloud/data`
- Nginx reverse proxy handling TLS termination, bound only to the Tailscale network interface
- A real Let's Encrypt certificate issued for the Tailscale MagicDNS hostname via Cloudflare DNS challenge
- Secure access from any device on the Tailscale network — no open router ports

---

## Prerequisites

- Docker and Docker Compose installed
- Nginx installed and running
- Tailscale installed on the server with MagicDNS enabled
- A Cloudflare account managing your domain (for DNS-based cert issuance)

---

## 1. Why Tailscale Instead of Port Forwarding

The standard approach for home server remote access is forwarding ports 80 and 443 through the router. That works, but your home IP becomes directly reachable from the internet and will be scanned constantly.

Tailscale creates a private WireGuard-encrypted network between all your devices. The daemon makes an outbound connection to Tailscale's relay — no inbound ports needed at all. Your home IP stays hidden, and access is gated by Tailscale authentication.

By binding Nginx to the Tailscale IP only, OpenCloud becomes unreachable from the public internet entirely.

---

## 2. Create the Directory Structure

```bash
sudo mkdir -p /opt/opencloud
sudo mkdir -p /home/phil/opencloud/config
sudo mkdir -p /home/phil/opencloud/data
sudo chown -R 1000:1000 /home/phil/opencloud
```

OpenCloud runs internally as UID 1000, so the data and config directories must be owned by that user or the container will fail to write files.

---

## 3. Initialise OpenCloud Before First Start

This step catches many people out. The container will crash on startup if `opencloud.yaml` doesn't exist with valid JWT secrets. Run `init` first with the exact URL you'll use to access the instance:

```bash
docker run --rm \
  -e OC_URL=https://YOUR-TAILSCALE-HOSTNAME.ts.net \
  -e INSECURE=true \
  -v /home/phil/opencloud/config:/etc/opencloud \
  -v /home/phil/opencloud/data:/var/lib/opencloud \
  --user 1000:1000 \
  opencloudeu/opencloud-rolling:latest \
  init --insecure --force-overwrite
```

This prints a generated admin password — save it. The `OC_URL` gets baked into the OIDC registration files at init time. If you initialise with the wrong URL and later change it in `.env`, login will still fail because the callback URLs in `data/idp/` still point to the old hostname. The fix is to wipe both `config/` and `data/` and re-run `init` with the correct URL.

---

## 4. Docker Compose File

`/opt/opencloud/docker-compose.yml`:

```yaml
services:
  opencloud:
    image: opencloudeu/opencloud-rolling:latest
    container_name: opencloud
    restart: unless-stopped
    ports:
      - "127.0.0.1:9200:9200"
    env_file:
      - .env
    volumes:
      - /home/phil/opencloud/config:/etc/opencloud
      - /home/phil/opencloud/data:/var/lib/opencloud
    extra_hosts:
      - "YOUR-TAILSCALE-HOSTNAME.ts.net:YOUR-TAILSCALE-IP"
    user: "1000:1000"
```

The `extra_hosts` entry is the non-obvious one. OpenCloud's internal proxy makes outbound HTTPS calls to its own `OC_URL` to verify OIDC tokens after login. Docker's internal DNS resolver (`127.0.0.11`) has no knowledge of Tailscale's MagicDNS, so without this entry every login attempt results in "Not logged in" even though the login form succeeds. The `extra_hosts` entry injects a static `/etc/hosts` record inside the container pointing the hostname at the host's Tailscale IP.

`/opt/opencloud/.env`:

```env
OC_URL=https://YOUR-TAILSCALE-HOSTNAME.ts.net
PROXY_HTTP_ADDR=0.0.0.0:9200
INSECURE=true
```

`INSECURE=true` tells OpenCloud to trust the upstream reverse proxy for TLS rather than managing its own certificates.

---

## 5. Issue a TLS Certificate for the Tailscale Hostname

Tailscale can issue real Let's Encrypt certificates for MagicDNS hostnames. First enable it in the admin console:

**tailscale.com/admin/dns → HTTPS Certificates → Enable**

Because the server is behind a home router with no open ports, the standard HTTP-01 ACME challenge won't work. Use the Cloudflare DNS-01 challenge instead — it proves domain ownership by creating a TXT record via the API, with no inbound connectivity required.

Install the Certbot Cloudflare plugin:

```bash
sudo apt install python3-certbot-dns-cloudflare -y
```

Create a Cloudflare API token at **My Profile → API Tokens → Create Token → Edit zone DNS**, scoped to your domain only. Then:

```bash
sudo mkdir -p /etc/letsencrypt
printf 'dns_cloudflare_api_token = YOUR_TOKEN\n' | sudo tee /etc/letsencrypt/cloudflare.ini
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
```

Issue the Tailscale cert using the Tailscale daemon:

```bash
sudo tailscale cert \
  --cert-file /tmp/hostname.crt \
  --key-file /tmp/hostname.key \
  YOUR-TAILSCALE-HOSTNAME.ts.net

sudo mkdir -p /etc/ssl/tailscale
sudo mv /tmp/hostname.crt /tmp/hostname.key /etc/ssl/tailscale/
sudo chmod 600 /etc/ssl/tailscale/*.key
sudo chmod 644 /etc/ssl/tailscale/*.crt
```

> If Tailscale is installed as a snap, the daemon writes certs to `/var/snap/tailscale/common/certs/` — copy from there if the `--cert-file` path gives a permission error.

---

## 6. Nginx Configuration

OpenCloud needs specific Nginx settings for large file uploads (TUS resumable upload protocol) and real-time sync (WebSockets).

`/etc/nginx/sites-available/opencloud`:

```nginx
server {
    listen 100.64.x.x:80;
    server_name YOUR-TAILSCALE-HOSTNAME.ts.net;
    return 301 https://$host$request_uri;
}

server {
    listen 100.64.x.x:443 ssl;
    server_name YOUR-TAILSCALE-HOSTNAME.ts.net;

    ssl_certificate     /etc/ssl/tailscale/hostname.crt;
    ssl_certificate_key /etc/ssl/tailscale/hostname.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # Required: unlimited upload size for large files
    client_max_body_size 0;

    location / {
        # OpenCloud serves HTTPS internally despite INSECURE=true
        proxy_pass https://127.0.0.1:9200;
        proxy_ssl_verify off;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Required: WebSocket support for real-time sync
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Required: disable buffering for TUS resumable uploads
        proxy_buffering         off;
        proxy_request_buffering off;

        # Long timeouts for large transfers
        proxy_connect_timeout   3600s;
        proxy_send_timeout      3600s;
        proxy_read_timeout      3600s;
    }
}
```

Binding to the Tailscale IP (`100.64.x.x`) means the vhost is only reachable from within your Tailscale network.

```bash
sudo ln -s /etc/nginx/sites-available/opencloud /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx
```

> Use `restart` not `reload` on first deploy. Nginx can cache the previous SSL context across reloads and present the wrong certificate until it's fully restarted.

---

## 7. Start OpenCloud

```bash
cd /opt/opencloud
docker compose up -d
docker compose logs -f
```

---

## 8. Verify and Access

```bash
# Container running?
docker compose ps

# OpenCloud reachable directly?
curl -sk https://127.0.0.1:9200/config.json | python3 -m json.tool | grep server

# Correct hostname in config?
# Should return: "server": "https://YOUR-TAILSCALE-HOSTNAME.ts.net/"
```

**Browser:** `https://YOUR-TAILSCALE-HOSTNAME.ts.net`

**Android:** Install the ownCloud app (OpenCloud is protocol-compatible), add server `https://YOUR-TAILSCALE-HOSTNAME.ts.net`

**Windows desktop sync:** Install the ownCloud desktop client, same server URL

**WebDAV:** `https://YOUR-TAILSCALE-HOSTNAME.ts.net/dav/files/admin`

All devices must be connected to your Tailscale network.

---

## Gotchas Summary

| Symptom | Cause | Fix |
|---|---|---|
| Container crashes immediately | Missing `opencloud.yaml` | Run `init` before first start |
| "Missing or invalid config" in browser | `OC_URL` wrong at init time | Wipe `config/` and `data/`, re-run `init` |
| "Not logged in" after every login | Docker can't resolve Tailscale hostname | Add `extra_hosts` in compose file |
| `proxy_pass` returns 400 | OpenCloud serves HTTPS, not HTTP | Use `proxy_pass https://` with `proxy_ssl_verify off` |
| Wrong cert served after config change | Nginx SSL context cached | Use `systemctl restart nginx`, not `reload` |

---

## TLS Certificate Renewal

Tailscale certs are valid for 90 days. The daemon auto-renews them into `/var/snap/tailscale/common/certs/`. Add a weekly cron job to copy the renewed cert into place:

`/etc/cron.weekly/renew-tailscale-cert`:

```bash
#!/bin/bash
cp /var/snap/tailscale/common/certs/*.crt /etc/ssl/tailscale/
cp /var/snap/tailscale/common/certs/*.key /etc/ssl/tailscale/
chmod 600 /etc/ssl/tailscale/*.key
systemctl reload nginx
```

```bash
sudo chmod +x /etc/cron.weekly/renew-tailscale-cert
```

---

## Useful References

- [OpenCloud documentation](https://docs.opencloud.eu/)
- [Tailscale HTTPS certificates](https://tailscale.com/kb/1153/enabling-https)
- [Certbot Cloudflare DNS plugin](https://certbot-dns-cloudflare.readthedocs.io/)
- [TUS resumable upload protocol](https://tus.io/)
- [ownCloud desktop client](https://owncloud.com/desktop-app/) (compatible with OpenCloud)
