---
title: Syncing Obsidian Vaults Across Devices with Syncthing
date: 2026-04-07T20:49:09+01:00
draft: false
tags:
  - syncthing
  - obsidian
description: ""
showToc: true
---
I've been using [Obsidian](https://obsidian.md) for a while now and one of the first problems I wanted to solve was keeping my vaults in sync across my machines without relying on a cloud service. The solution I landed on is [Syncthing](https://syncthing.net) — open source, peer-to-peer, and self-hosted.

  

This post covers how I set it up with a Linux server acting as an always-on central node, syncing to a Windows machine.

  

---

  

## The Architecture

  

Rather than syncing directly between devices (which only works when both are online at the same time), I use my home server as a central hub. Every device syncs with the server, so files are always available even if one device is offline.

  

```

Windows Machine  ←→  Linux Server (always on)

```

  

All vaults live under a single master directory on both machines:

  

```

# Linux

~/ObsidianVaults/

├── MasterVault/

└── homelab-blog/

  

# Windows

D:\ObsidianVaults\

├── MasterVault\

└── homelab-blog\

```

  

---

  

## Server Setup

  

### 1. Install Syncthing

  

On Ubuntu/Debian, Syncthing is in the default repos:

  

```bash

sudo apt update && sudo apt install syncthing -y

```

  

### 2. Create the vault directory

  

```bash

mkdir -p ~/ObsidianVaults

```

  

### 3. Configure for remote access

  

Start Syncthing once to generate the config, then stop it:

  

```bash

syncthing --no-browser &

sleep 8 && kill %1

```

  

The config lives at `~/.local/state/syncthing/config.xml`. By default the web UI only listens on localhost. Change it to listen on all interfaces so you can reach it from other machines on your network:

  

```bash

sed -i 's|127.0.0.1:8384|0.0.0.0:8384|' ~/.local/state/syncthing/config.xml

```

  

### 4. Open firewall ports

  

```bash

sudo ufw allow 8384/tcp comment 'Syncthing GUI'

sudo ufw allow 22000/tcp comment 'Syncthing Sync'

sudo ufw allow 22000/udp comment 'Syncthing Sync'

sudo ufw reload

```

  

### 5. Enable as a systemd service

  

Syncthing ships with a system service template on Ubuntu:

  

```bash

sudo systemctl enable --now syncthing@$USER.service

```

  

This starts Syncthing at boot and runs it as your user.

  

### 6. Access the web UI

  

Open `http://<your-server-ip>:8384` from any machine on your network.

  

> **Important:** Set a GUI password immediately via **Actions → Settings → GUI**.

  

---

  

## Adding Vaults

  

Each vault is a separate Syncthing folder. To add a new one:

  

1. Create the directory on the server:

   ```bash

   mkdir ~/ObsidianVaults/NewVault

   ```

2. In the web UI, click **Add Folder** and set the path to `~/ObsidianVaults/NewVault`

3. Under the **Sharing** tab, select the devices to sync with

4. On each device, accept the folder prompt and set the local path

  

---

  

## Windows Setup

  

Syncthing for Windows can be downloaded from [syncthing.net](https://syncthing.net/downloads/). Once installed, open `http://localhost:8384` to access the local GUI.

  

To make Syncthing start automatically at login without a terminal window, run this in an admin PowerShell (adjust the path to match your install location):

  

```powershell

$action = New-ScheduledTaskAction -Execute "C:\Users\<username>\AppData\Local\Programs\Syncthing\syncthing.exe" -Argument "--no-browser"

$trigger = New-ScheduledTaskTrigger -AtLogOn -User $env:USERNAME

$settings = New-ScheduledTaskSettingsSet -ExecutionTimeLimit 0 -RestartCount 3 -RestartInterval (New-TimeSpan -Minutes 1)

Register-ScheduledTask -TaskName "Syncthing" -Action $action -Trigger $trigger -Settings $settings -RunLevel Highest -Force

```

  

---

  

## Ignoring Device-Specific Files

  

Obsidian's `workspace.json` tracks things like open panes and cursor position — it changes constantly and is device-specific, causing unnecessary sync conflicts. Add a `.stignore` file to each vault to exclude it:

  

```

.obsidian/workspace.json

.obsidian/workspace-mobile.json

```

  

The `.stignore` file itself syncs between devices, so you only need to create it once.

  

---

  

## Troubleshooting

  

**Device shows as disconnected**

- Check Syncthing is running on the other machine

- Edit the device entry and set a static address: `tcp://<server-ip>:22000` — this bypasses discovery and connects directly over LAN

  

**Folder stuck in error state**

- Remove the folder from the affected device's Syncthing config (no files are deleted)

- Refresh the GUI — the server will re-offer the folder

- Re-add it with the correct local path

  

**Check service status on the server**

```bash

sudo systemctl status syncthing@<username>.service

journalctl -u syncthing@<username>.service -f

```

  

---

  

## Why Not Obsidian Sync?

  

Obsidian Sync is a great product and I have nothing against it — I just prefer keeping my notes on hardware I control. Syncthing is free, works entirely on your local network (no cloud relay needed for LAN devices), and handles the occasional conflict gracefully.

  

If you're already running a home server, it's a natural fit.
