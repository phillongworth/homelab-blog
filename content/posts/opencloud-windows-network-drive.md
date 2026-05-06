---
title: Mapping OpenCloud as a Windows Network Drive via WebDAV
date: 2026-05-06T00:00:00Z
draft: false
tags:
  - opencloud
  - tailscale
  - webdav
  - windows
  - self-hosted
  - file-sync
categories:
  - Guides
description: How to map a self-hosted OpenCloud instance as a persistent Windows network drive using WebDAV over Tailscale, including fixes for nginx startup ordering and Windows credential quirks.
showToc: true
---

This post covers mapping a self-hosted OpenCloud instance as a Windows network drive using WebDAV. It follows on from [Self-Hosting OpenCloud with Docker and Tailscale](/posts/opencloud-docker-tailscale/), which covers the server-side setup. The goal here is simple: make OpenCloud appear as a drive letter in File Explorer on Windows, persisting across reboots.

There were a few non-obvious gotchas along the way — nginx wasn't starting after reboots, Windows credential handling is quirky with WebDAV, and an old Apache installation was silently blocking everything. All documented below.

---

## What We Built

- OpenCloud accessible as drive `O:` in Windows File Explorer
- Connection routed securely over Tailscale — no open router ports
- Credentials stored in Windows Credential Manager using a Personal Access Token
- Drive reconnects automatically on reboot

---

## Prerequisites

- The OpenCloud server setup from [the previous post](/posts/opencloud-docker-tailscale/) — OpenCloud running in Docker behind an nginx reverse proxy, accessible at `https://optiplex7040-server.taila5b04f.ts.net`
- Tailscale installed on the Windows machine and signed into the same account as the server

---

## 1. Fix: nginx Wasn't Starting After Reboots

Before mapping the drive, OpenCloud wasn't reachable at all. The server was up and the Docker container was running, but HTTPS connections failed. The culprit: nginx was failing to start on boot because Tailscale hadn't finished assigning its IP yet when nginx tried to bind to it.

SSH into the server and check:

```bash
sudo systemctl status nginx
```

The error will look like:

```
nginx: [emerg] bind() to 100.64.x.x:443 failed (99: Cannot assign requested address)
```

The secondary problem: a default Apache installation was still running and had grabbed ports 80 and 443 first. Since we only use nginx here, Apache can go.

```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
```

Then fix the startup ordering so nginx waits for Tailscale:

```bash
sudo mkdir -p /etc/systemd/system/nginx.service.d
sudo tee /etc/systemd/system/nginx.service.d/tailscale.conf <<EOF
[Unit]
After=tailscaled.service
Wants=tailscaled.service
EOF
sudo systemctl daemon-reload
sudo systemctl start nginx
```

Verify nginx is running:

```bash
sudo systemctl status nginx
```

---

## 2. Enable the Windows WebClient Service

Windows WebDAV support depends on the `WebClient` service, which is disabled by default. Open an **elevated** PowerShell (right-click → Run as administrator):

```powershell
sc.exe config WebClient start= auto
sc.exe start WebClient
```

---

## 3. Generate a Personal Access Token in OpenCloud

OpenCloud's browser login uses OIDC/OAuth2, which is separate from WebDAV authentication. Rather than using your account password, generate a Personal Access Token specifically for WebDAV.

Log into OpenCloud at `https://optiplex7040-server.taila5b04f.ts.net`, then go to:

**Settings → Personal → Security → Personal Access Tokens**

Create a new token and copy it — you won't be able to see it again after leaving the page.

---

## 4. Store Credentials and Map the Drive

> **Important:** Run these commands in a **regular (non-elevated)** PowerShell. If you use an elevated session, the drive maps to the Administrator account and won't appear in File Explorer for your normal user.

Store the token in Windows Credential Manager first. Pasting the token directly at `net use`'s password prompt can fail silently if the token contains special characters — storing it via `cmdkey` avoids this.

Replace `YOUR_TOKEN` with the token you just generated:

```powershell
cmdkey /add:optiplex7040-server.taila5b04f.ts.net /user:admin /pass:YOUR_TOKEN
```

Then map the drive:

```powershell
net use O: "https://optiplex7040-server.taila5b04f.ts.net/remote.php/webdav" /persistent:yes
```

When prompted, enter `admin` as the username and your token as the password.

To rename the drive from the default `webdav` label, right-click it in File Explorer and select **Rename**.

---

## 5. Verify

Open File Explorer — drive `O:` should appear under **This PC**. You can browse, copy, and edit files directly. Performance will vary depending on your Tailscale connection quality.

---

## Ongoing Maintenance

### Token expires

Generate a new token in OpenCloud settings, then update the stored credential:

```powershell
cmdkey /add:optiplex7040-server.taila5b04f.ts.net /user:admin /pass:NEW_TOKEN
net use O: /delete
net use O: "https://optiplex7040-server.taila5b04f.ts.net/remote.php/webdav" /persistent:yes
```

### Reinstalling Windows

The full steps to restore the mapped drive on a fresh Windows install:

1. Install Tailscale and sign in
2. Enable WebClient (elevated PowerShell): `sc.exe config WebClient start= auto && sc.exe start WebClient`
3. Generate a new Personal Access Token in OpenCloud
4. Store with `cmdkey` and map with `net use` as above (regular PowerShell)

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Drive not visible in File Explorer | `net use` was run as Administrator | Remap in a regular (non-elevated) PowerShell |
| "Access is denied" (error 5) | Token has special characters mangled at prompt | Use `cmdkey` to pre-store the credential |
| "Access is denied" (error 5) | Tailscale not connected | Connect Tailscale first |
| "The network connection could not be found" | WebClient service not running | `sc.exe start WebClient` (elevated) |
| HTTPS connection fails from browser | nginx not started | `sudo systemctl start nginx` on server |
| nginx fails to start on server reboot | Starts before Tailscale IP is assigned | Add `tailscale.conf` override as above |
