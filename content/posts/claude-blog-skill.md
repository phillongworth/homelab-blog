---
title: Turning Claude Code into a Blog Post Writer with a Custom Skill
date: 2026-04-21T00:00:00Z
draft: false
tags:
  - claude
  - claude-code
  - automation
  - blogging
  - hugo
  - obsidian
categories:
  - Notes
description: How I created a custom Claude Code slash command that writes, sanitises, and saves blog posts directly into my Obsidian/Hugo vault from a conversation.
showToc: true
---

One friction point with documenting homelab work is that the interesting stuff happens in a terminal session, and writing it up afterwards means context-switching to a text editor and reconstructing what you did from memory. This post covers a small automation that removes that step: a custom Claude Code skill that turns any conversation into a finished blog post, saved directly into my Obsidian vault.

---

## What We Built

- A custom Claude Code slash command (`/blog`) that writes a full Hugo-formatted blog post based on the current conversation
- Automatic front matter generation (title, date, tags, description)
- A built-in security check that strips sensitive data (IPs, hostnames, tokens) before saving
- Direct file write to the Obsidian vault so the post is one Git sync away from being live

---

## What is a Claude Code Custom Skill?

Claude Code supports user-defined slash commands stored as Markdown files in `~/.claude/commands/`. When you type `/commandname` in a Claude Code session, Claude reads the corresponding `.md` file as a system prompt and executes it against the current conversation context.

The Markdown file is just an instruction document — it tells Claude what to do, what format to use, and what rules to follow. You can pass arguments too: `/blog My Post Title` makes the string `My Post Title` available as `$ARGUMENTS` inside the prompt.

---

## 1. Create the Commands Directory

```bash
mkdir -p ~/.claude/commands
```

---

## 2. Write the Skill File

`~/.claude/commands/blog.md`:

````markdown
# Homelab Blog Post Writer

You are writing a blog post for a homelab blog built with Hugo and PaperMod.

## Blog details

- **Posts directory:** `/path/to/ObsidianVaults/homelab-blog/content/posts/`
- **Audience:** Homelab enthusiasts — assume Linux/Docker/Nginx familiarity, explain the non-obvious

## Front matter template

```yaml
---
title: "Title Case Title Here"
date: YYYY-MM-DDT00:00:00Z
draft: true
tags: ["tag1", "tag2"]
categories: ["Guides"]
description: "One sentence summary."
showToc: true
---
```

## Writing style

- Open with one paragraph explaining what was built and what the key challenge was
- Use a `## What We Built` summary section near the top
- Use a `## Prerequisites` section
- Number the main sections: `## 1. First Step`, `## 2. Second Step`, etc.
- Every code block must have a language tag
- Use blockquotes for warnings and gotchas
- End with a `## Useful References` section

## Security check — REQUIRED before saving

Before saving, replace all of the following with generic placeholders:

- Real domain names → `yourdomain.com`
- Tailscale hostnames → `YOUR-TAILSCALE-HOSTNAME.ts.net`
- Private/LAN IPs → `192.168.x.x`
- API tokens, passwords, secrets → `YOUR_TOKEN_HERE`
- Email addresses → `your@email.com`
- Server hostnames → `your-server`

List what was redacted after saving.

## Your task

Write a complete blog post based on the current conversation (or $ARGUMENTS if provided),
run the security check, save to the posts directory, and report the filename and redactions.
````

The security check section is the most important part for a public blog. Claude reads the whole conversation history, so without it a post could contain IPs, hostnames, or tokens that were mentioned during the actual setup work.

---

## 3. Trigger the Skill

With Claude Code open in any terminal session, type:

```
/blog
```

Claude writes a post based on everything in the current conversation — the commands run, the errors hit, the fixes applied — and saves it directly to the vault.

To write a post on a specific topic rather than the current session:

```
/blog Setting up Pi-hole with Docker
```

The `$ARGUMENTS` value is passed into the prompt and used as the topic and title.

---

## 4. The Resulting Workflow

1. Do the homelab work in a Claude Code session as normal
2. At the end of the session, type `/blog`
3. Claude writes and saves the post — including a gotchas section based on what actually went wrong
4. Open Obsidian, review the post, set `draft: false`
5. `Ctrl+P` → **Git: Commit-and-sync**
6. GitHub Actions deploys the site in ~30 seconds

The post is written from the actual conversation, so it captures the real errors and fixes rather than a sanitised happy-path walkthrough. That tends to make for more useful documentation.

---

## Customising for Your Own Blog

The skill file is plain Markdown — adjust the posts directory path, front matter template, and style rules to match your own setup. The security check table is worth keeping regardless of your stack; it's easy to forget that a hostname mentioned casually in a terminal session is about to appear in a public Git repository.

---

## Useful References

- [Claude Code custom slash commands documentation](https://docs.anthropic.com/en/docs/claude-code/slash-commands)
- [Hugo front matter reference](https://gohugo.io/content-management/front-matter/)
- [PaperMod theme](https://github.com/adityatelange/hugo-PaperMod)
