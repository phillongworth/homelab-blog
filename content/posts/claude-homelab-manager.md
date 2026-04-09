---
title: "Building an Interactive Homelab Manager with Claude Code"
date: 2026-04-09T00:00:00Z
draft: false
tags: ["claude", "rclone", "docker", "obsidian", "automation", "python", "cron"]
categories: ["Guides"]
description: "Using Claude Code as an interactive homelab manager — SSHing into the server, indexing services and cloud drives, and building a self-updating Obsidian note."
showToc: true
---

This post documents a session using Claude Code as an interactive homelab manager. The goal was to get a full picture of everything running on the server — Docker containers, systemd services, config files, and cloud storage — and have it kept automatically up to date in an Obsidian note.

---

## What We Built

By the end of this session we had:

- A live **Homelab Manager** Obsidian note covering folders, containers, services, configs, and cloud drives
- A **Python script** that regenerates the note weekly with fresh data
- A **cron job** running every Sunday midnight
- **rclone configured** for six cloud services: Google Drive, Box, Dropbox, Mega, Proton Drive, and Filen
- The note **syncing automatically** to Windows via Syncthing

---

## Prerequisites

- Claude Code installed on the Windows machine
- Linux server accessible via SSH key auth
- `/home/phil` mapped to `Z:` on Windows via Samba
- Obsidian vault at `/home/phil/ObsidianVaults/MasterVault/` synced via Syncthing
- rclone installed on the Linux server (`rclone version` to check)

---

## 1. Indexing the Server

The first task was to get Claude Code to SSH into the server and pull a complete picture of what was running. The prompt was:

```
SSH into <user>@<server-ip>.
1. List all folders in /home/phil as clickable links using file:///Z:/ paths
2. Find any .conf or .yaml files in /home/phil/
3. Check for running Docker containers and systemd services
```

Claude ran these commands in parallel:

```bash
# Folders
ls -1d /home/phil/*/

# Config files
find /home/phil -maxdepth 3 \( -name '*.conf' -o -name '*.yaml' -o -name '*.yml' \) \
  2>/dev/null | grep -v '/\.' | sort

# Docker containers
docker ps --format 'NAME={{.Names}}\tIMAGE={{.Image}}\tSTATUS={{.Status}}\tPORTS={{.Ports}}'

# Systemd services
systemctl list-units --type=service --state=running --no-pager --no-legend | awk '{print $1}'
```

The output was formatted into tables with `file:///Z:/` clickable links for every folder and config file — so clicking any row in the table opens the file directly in Windows Explorer or VS Code.

---

## 2. Creating the Obsidian Note

Rather than a one-off output, the data was written into a permanent Obsidian note at:

```
/home/phil/ObsidianVaults/MasterVault/Linux Server/Homelab Manager.md
```

The `Linux Server/` folder already existed in the vault, so this slotted in naturally alongside existing notes like `SERVER_MANUAL.md` and `Cockpit.md`.

The note was written to the server via SSH using base64 to avoid heredoc escaping issues:

```bash
base64 /tmp/note.md | ssh <user>@<server-ip> \
  "base64 -d > '/home/phil/ObsidianVaults/MasterVault/Linux Server/Homelab Manager.md'"
```

Because the vault is synced by Syncthing, the note appeared in Obsidian on Windows within seconds — no manual copy needed.

---

## 3. Auto-Refresh with a Python Script and Cron

A static note goes stale the moment you add a new container. The solution was a Python script that regenerates the entire note from live data.

The script lives at `/home/phil/scripts/update_homelab_note.py` and does the following on each run:

1. Scans `/home/phil/` for all top-level folders
2. Runs `docker ps` and infers compose file paths from folder names
3. Runs `find` for all `.conf`/`.yaml`/`.yml` files up to 3 levels deep
4. Runs `systemctl list-units` for all running services
5. Runs `rclone about` and `rclone tree --level 2` for every configured remote
6. Rebuilds the full note from scratch
7. Appends to the Change Log table with the run date

The script handles errors gracefully — a timeout or broken remote gets a ⚠️ flag in the table instead of crashing the whole run.

Wire it up to cron:

```bash
crontab -e
```

```
0 0 * * 0  python3 /home/phil/scripts/update_homelab_note.py >> /home/phil/scripts/update_homelab_note.log 2>&1
```

This runs every Sunday at midnight. The log file at `update_homelab_note.log` captures any errors for debugging.

To trigger a manual refresh at any time:

```bash
python3 ~/scripts/update_homelab_note.py
```

---

## 4. Indexing Cloud Drives with rclone

With the server-side infrastructure covered, the next step was mapping cloud storage. The aim was to get a directory tree and storage figures for each service without downloading any content.

### Checking existing remotes

```bash
rclone listremotes
```

Only `gdrive:` was configured initially. The others needed setting up: Box, Dropbox, Mega, Proton Drive, and Filen.

### Upgrading rclone

The installed version was v1.60.1 — too old for Proton Drive (requires v1.65+) and Filen (requires v1.67+), and also the cause of Box OAuth failures due to stale app credentials.

```bash
sudo -v && curl https://rclone.org/install.sh | sudo bash
rclone version
```

This upgrades in place and preserves all existing remote configs.

### Configuring OAuth remotes (Box, Dropbox) — headless flow

The server has no browser, so the standard OAuth flow needs SSH port forwarding. The key insight is the order of operations:

1. **Open the tunnel first** — in a Windows terminal:

```bash
ssh -L 53682:localhost:53682 <user>@<server-ip>
```

2. **Then run the interactive config wizard** on the Linux server:

```bash
rclone config
# n → new remote → box → accept defaults → y for auto config
```

3. **Visit the URL in your Windows browser** — rclone prints something like:
```
http://127.0.0.1:53682/auth?state=xxxxx
```
The tunnel forwards your Windows browser's request back to the rclone server on Linux.

4. Authorise in the browser — rclone captures the token automatically.

> **Important:** Use `rclone config` interactively rather than `rclone config create` + `rclone authorize`. The two-step approach causes timing issues on headless servers — the auth webserver starts and stops before the browser can connect.

If you hit `address already in use`:

```bash
kill $(lsof -ti:53682)
```

Then retry.

### Configuring credential remotes (Mega, Proton Drive, Filen)

These use email/password rather than OAuth — no browser needed:

```bash
# Mega
rclone config create Mega mega \
  user YOUR_EMAIL \
  pass $(rclone obscure YOUR_PASSWORD)

# Proton Drive
rclone config create ProtonDrive protondrive \
  username YOUR_EMAIL \
  password $(rclone obscure YOUR_PASSWORD)
```

> For Proton Drive: if you have 2FA enabled, create an **app-specific password** in Proton Account Settings → Security → App Passwords. Use that instead of your login password.

### Filen — config pitfall

`rclone config create` doesn't work for Filen because the backend needs to make API calls during setup to fetch an auth token. Use the interactive wizard instead:

```bash
rclone config delete Filen
rclone config
# n → filen → enter email and password when prompted
```

Also make sure to use `rclone obscure` for any password passed on the command line — plain text passwords will fail with:

```
failed to reveal api key: input too short when revealing password - is it obscured?
```

### Running the cloud index

Once all remotes are configured:

```bash
# Storage summary for each remote
rclone about gdrive:
rclone about Box:
# etc.

# Directory tree (top 2 levels, folders only)
rclone tree gdrive: --level 2 --dirs-only
```

The full results for six remotes:

| Service | Remote | Total | Used | Free |
|---------|--------|-------|------|------|
| Google Drive | `gdrive:` | 17 GiB | 2.3 GiB | 5.6 GiB |
| Box | `Box:` | 50 GiB | 8.8 GiB | 41.2 GiB |
| Mega | `Mega:` | 20 GiB | 8.8 GiB | 11.2 GiB |
| Filen | `Filen:` | 20 GiB | 167 KiB | 20.0 GiB |
| Proton Drive | `ProtonDrive:` | 5 GiB | 5.5 MiB | 5.0 GiB |
| Dropbox | `Dropbox:` | 6.75 GiB | 477 KiB | 6.75 GiB |
| **Total** | | **~119 GiB** | **~20 GiB** | **~84 GiB** |

---

## 5. The Finished Note

The Homelab Manager note has the following sections, all auto-refreshed weekly:

- **Folders** — every directory in `/home/phil/` with clickable `file:///Z:/` links
- **Docker Containers** — name, image, ports, inferred compose file link, status
- **Config & YAML Files** — grouped by service folder, all clickable
- **Systemd Services** — all running services with known config paths
- **Cloud Drives** — storage map and directory trees for all rclone remotes
- **Useful Commands** — reference snippets for Docker, journalctl, rclone
- **Change Log** — auto-appended row on every script run

The note syncs to Windows via Syncthing, so it's always up to date on both machines without any manual steps.

---

## Security Note

The rclone config file at `~/.config/rclone/rclone.conf` stores credentials in rclone's obscured format (not encrypted). Consider encrypting it:

```bash
rclone config  # → s → Set configuration password
```

This encrypts the config at rest — you'll be prompted for the password when rclone starts.

---

## Useful References

- [rclone docs — box](https://rclone.org/box/)
- [rclone docs — dropbox](https://rclone.org/dropbox/)
- [rclone docs — protondrive](https://rclone.org/protondrive/)
- [rclone docs — filen](https://rclone.org/filen/)
- [rclone — remote setup (headless)](https://rclone.org/remote_setup/)
