---
title: "Hello Homelab"
date: 2026-04-03
draft: false
tags: ["homelab", "meta"]
categories: ["General"]
description: "First post — what this blog is for."
showToc: false
---

Welcome to the homelab blog. This is where I document configs, experiments,
and anything worth writing down.
And Im using Obsidian

```bash
# Example: check what's listening on a port
ss -tulpn | grep :8080
```

```yaml
# Docker Compose snippet
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
```
