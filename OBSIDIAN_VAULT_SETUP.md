# Obsidian → Hugo Integration

## Point your Vault at the content directory

Open Obsidian and set the vault root to:

```
homelab-blog/content/
```

That's it. Every note you save is a valid Hugo content file as long as you keep
front matter at the top.

## Front matter template

Install the **Templater** plugin and save this as your default template:

```markdown
---
title: "{{tp.file.title}}"
date: {{tp.date.now("YYYY-MM-DDTHH:mm:ssZ")}}
draft: true
tags: []
---

```

## Folder → section mapping

| Obsidian folder | Hugo section | URL                         |
|-----------------|--------------|-----------------------------|
| `posts/`        | Posts        | `/posts/<slug>/`            |
| `notes/`        | Notes        | `/notes/<slug>/`            |
| `snippets/`     | Snippets     | `/snippets/<slug>/`         |

## Wiki-link handling

Obsidian `[[wiki-links]]` are **not** natively supported by Hugo.
Install the **obsidian-export** CLI or the **Enveloppe** plugin to convert
them to standard Markdown links before pushing.

```bash
# obsidian-export (Rust CLI)
obsidian-export --frontmatter=never vault/ content/notes/
```

## Publish workflow

1. Write in Obsidian, set `draft: false` when ready.
2. `git add content/ && git commit -m "new note: <title>"`
3. `git push` — GitHub Actions builds and deploys automatically.
