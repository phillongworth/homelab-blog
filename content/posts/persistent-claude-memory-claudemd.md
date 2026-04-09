---
title: "Giving Claude Code a Persistent Memory with CLAUDE.md"
date: 2026-04-09T00:00:00Z
draft: false
tags: ["claude", "obsidian", "automation", "productivity", "configuration"]
categories: ["Guides"]
description: "How to configure CLAUDE.md and a memory file system so Claude Code remembers your homelab setup, writing preferences, and security rules across every conversation."
showToc: true
---

By default, every Claude Code conversation starts with a blank slate. Claude has no memory of your server's IP, your preferred writing style, or the security rules you want applied before publishing. You end up re-explaining the same context at the start of every session.

This post documents how to fix that — using `CLAUDE.md` and a small memory file system so Claude picks up your full homelab context automatically, every time.

---

## What We Built

- A **`~/.claude/CLAUDE.md`** file that Claude reads automatically at the start of every conversation
- A **memory file system** with project-specific reference files linked from a central index
- Automatic enforcement of **security rules** when writing blog posts — stripping real IPs, usernames, and credentials before saving
- A **homelab blog style guide** so every post follows the same front matter format and writing conventions without having to ask

---

## How CLAUDE.md Works

Claude Code looks for `CLAUDE.md` files in a fixed set of locations and loads them automatically at the start of each session, before you type anything. Think of it as a system prompt you control.

There are three levels:

| File | Scope |
|------|-------|
| `~/.claude/CLAUDE.md` | Every conversation, on any project |
| `/path/to/project/CLAUDE.md` | Only when working in that project directory |
| `/path/to/project/subdir/CLAUDE.md` | Only when in that subdirectory |

For homelab work that spans multiple projects, the user-level file at `~/.claude/CLAUDE.md` is the right place.

---

## 1. Create the User-Level CLAUDE.md

Create `~/.claude/CLAUDE.md` with your homelab context and any standing instructions:

```markdown
# Claude Code — User Preferences

## Homelab Context

- Linux server: `<username>@<server-ip>` (SSH key auth)
- `/home/<username>` is mapped to `Z:` on Windows via Samba
- Obsidian vault synced between Linux and Windows via Syncthing
- See memory files for full details: `~/.claude/projects/.../memory/`

## Homelab Blog

When writing or editing a post for the homelab blog:

1. **Use the style guide** in `memory/project_homelab_blog.md`

2. **Always apply security rules before saving:**
   - Replace real server IPs → `<server-ip>`
   - Replace real SSH usernames in commands → `<username>`
   - Replace real email addresses → `YOUR_EMAIL`
   - Replace passwords/tokens → `YOUR_PASSWORD` / `YOUR_TOKEN`

3. **Save posts to:**
   `/home/<username>/ObsidianVaults/homelab-blog/content/posts/<kebab-case-title>.md`
   Use base64 over SSH to transfer.

4. **Set `draft: false`** when ready. Remind the user to push via Obsidian Git.

## General Preferences

- Prefer sequential SSH commands over parallel when > 3 connections
- Use base64 pipe over SSH when writing multi-line files to the server
- Format server file links as `file:///Z:/path` for Windows clickability
- Keep temp files in `C:/Users/<username>/.claude/` and clean them up after use
```

The instructions here are directives — Claude treats them as rules to follow, not just context to be aware of. Phrasing like "always apply security rules before saving" will be acted on, not just noted.

---

## 2. Create the Memory File System

`CLAUDE.md` is intentionally kept short — it loads every session so it shouldn't be a wall of text. Detailed reference material lives in separate memory files that Claude reads when relevant.

The structure used here:

```
~/.claude/
├── CLAUDE.md                          ← auto-loaded every session
└── projects/
    └── C--Users-username--claude/
        └── memory/
            ├── MEMORY.md              ← index of all memory files
            ├── project_homelab.md     ← server details, drive mapping, sync setup
            └── project_homelab_blog.md ← blog style guide, front matter, security rules
```

### MEMORY.md — the index

```markdown
# Memory Index

- [Homelab Setup](project_homelab.md) — server IP, Z: drive mapping, Syncthing sync
- [Homelab Blog](project_homelab_blog.md) — Hugo/PaperMod site, front matter, style, security rules
```

Claude uses this as a map — when a task relates to the blog or the server, it loads the relevant detail file.

### project_homelab.md — server reference

```markdown
## Server

- `<username>@<server-ip>` — SSH key auth
- `/home/<username>` → `Z:` via Samba
- Syncthing syncs `/home/<username>/ObsidianVaults/` to Windows

## File Transfer

Use base64 over SSH for multi-line files:
  base64 file.md | ssh <username>@<server-ip> "base64 -d > /target/path.md"
```

### project_homelab_blog.md — blog style guide

This is the most detailed file. It contains:

```markdown
## Front Matter Template

---
title: "Post Title Here"
date: YYYY-MM-DDT00:00:00Z
draft: false
tags: ["tag1", "tag2"]
categories: ["Guides"]
description: "One sentence summary."
showToc: true
---

## Writing Style

- Practical, first-person, step-by-step
- Numbered top-level sections
- Lead with what was built before how to build it
- Use > **Note:** callouts for gotchas
- End with Useful References section
- Always specify language on code blocks

## Security Rules

- Replace real IPs with <server-ip>
- Replace SSH usernames with <username>
- Replace emails with YOUR_EMAIL
- /home/<username>/ paths in prose are fine
- Cloud storage amounts are fine to publish

## Post Filename Convention

kebab-case-title.md → /posts/kebab-case-title/
```

---

## 3. How It Applies in Practice

With this in place, starting a new blog post is as simple as:

```
Write a blog post about setting up Paperless-ngx with Docker on my homelab server.
```

Claude will automatically:

1. Load the blog style guide from `project_homelab_blog.md`
2. Write in the established format with correct front matter
3. Apply security rules before saving — no real IPs or usernames in the output
4. Transfer the file to the correct path on the server via base64 over SSH
5. Remind you to push via Obsidian Git when done

No preamble required. No re-explaining the setup each time.

---

## 4. Keeping Memory Files Up to Date

The memory files are plain Markdown — you can edit them directly in Obsidian or any editor. When something changes (new server, different blog path, updated style preferences), just update the relevant file.

A few useful moments to update them:

- **New service installed** → add it to `project_homelab.md`
- **New blog post published** → add it to the existing posts table in `project_homelab_blog.md`
- **Style preference changes** → update the writing style section
- **New security rule** → add it to the security checklist

You can also ask Claude to update them during a session:

```
Add a note to my homelab memory that I've switched from Syncthing to rsync for vault sync.
```

---

## 5. Project-Level CLAUDE.md

For repos or projects with their own specific rules, you can place a `CLAUDE.md` directly in the project directory. It merges with your user-level file — both are loaded.

For example, a `CLAUDE.md` at the root of your `homelab-blog` repo could contain:

```markdown
# Homelab Blog

- Hugo site — do not edit files outside content/
- Never commit with draft: true
- Run `hugo server` to preview before publishing
```

This only loads when Claude Code is working in that directory.

---

## The Result

The investment is about 10 minutes of setup. The return is that Claude arrives at every conversation already knowing:

- Your server address and how to connect
- How to safely transfer files
- Your blog's front matter format and writing style
- Which details to strip before any post goes public
- Where to save files and how to publish them

It effectively closes the gap between a generic AI assistant and one that knows your specific setup.

---

## Useful References

- [Claude Code — CLAUDE.md documentation](https://docs.anthropic.com/en/docs/claude-code/memory)
- [Claude Code — Settings and configuration](https://docs.anthropic.com/en/docs/claude-code/settings)
