# Homelab Blog

A personal homelab blog built with [Hugo](https://gohugo.io/) (PaperMod theme), written in [Obsidian](https://obsidian.md/), and automatically deployed to [GitHub Pages](https://pages.github.com/).

🌐 **Live site:** https://phillongworth.github.io/homelab-blog/

---

## Stack

| Component | Tool |
|---|---|
| Static site generator | Hugo Extended 0.146.0+ |
| Theme | PaperMod |
| Writing interface | Obsidian |
| Deployment | GitHub Actions → GitHub Pages |
| Syntax highlighting | Chroma (Dracula theme) |

---

## Repository Structure

```
homelab-blog/
├── .github/
│   └── workflows/
│       └── hugo.yml          # Build and deploy pipeline
├── archetypes/               # Hugo front matter templates
├── content/
│   ├── posts/                # Long-form write-ups
│   ├── notes/                # Quick references
│   ├── snippets/             # Configs and one-liners
│   ├── Templates/            # Obsidian Templater templates
│   └── search.md
├── themes/
│   └── PaperMod/             # Git submodule
├── .gitignore
├── .gitattributes
└── hugo.toml                 # Site configuration
```

---

## Local Development

### Requirements

- Hugo Extended 0.146.0+
- Git

### Install Hugo (Windows)

```powershell
winget install Hugo.Hugo.Extended
```

### Clone and run

```powershell
git clone --recurse-submodules https://github.com/phillongworth/homelab-blog.git
cd homelab-blog
hugo server -D
```

Open http://localhost:1313 in your browser. The `-D` flag includes draft posts.

---

## Writing Posts

Posts are written in Obsidian with the vault rooted at `content/`.

### Obsidian plugins required

| Plugin | Purpose |
|---|---|
| Templater | Auto-fills front matter on new notes |
| Git | Commit and push without leaving Obsidian |

### Templater configuration

- **Template folder location:** `Templates`
- **Trigger Templater on new file creation:** on
- **Folder templates:** `posts` → `Templates/post`

### Git plugin configuration

- **Sync method:** Commit-and-sync
- **Auto pull interval:** 10 minutes
- **Commit message:** `docs: {{date}} {{hostname}}`

### Publish workflow

1. Create a new file in `content/posts/` — front matter fills automatically
2. Write content in Markdown
3. Set `draft: false` when ready to publish
4. `Ctrl+P` → **Git: Commit-and-sync**
5. GitHub Actions deploys the site in ~30 seconds

---

## Deployment

Deployment is fully automated via GitHub Actions (`.github/workflows/hugo.yml`).

- Triggers on every push to `main`
- Installs Hugo Extended via `peaceiris/actions-hugo@v3`
- Builds with `--minify`
- Deploys to GitHub Pages via `actions/deploy-pages@v4`

### GitHub Pages setup (one-time)

In the repository: **Settings → Pages → Source → GitHub Actions**

---

## Configuration Notes

Key decisions in `hugo.toml` worth knowing:

- **`ignoreFiles`** — excludes `content/Templates/` from Hugo builds so Obsidian template files are never processed
- **`googleAnalytics = ""`** — must be present even if blank; PaperMod errors without it
- **`[[params.socialIcons]]`** — must use double brackets; single brackets cause a TOML parse error
- **`[pagination] pagerSize`** — replaces the deprecated `paginate` key removed in Hugo 0.128.0
- **`ignoreFiles` must be at root level** — placing it after a `[table]` header puts it inside that table and silently breaks it

---

## Theme

[PaperMod](https://github.com/adityatelange/hugo-PaperMod) is installed as a Git submodule. To update it:

```powershell
git submodule update --remote --merge
git commit -am "chore: update PaperMod theme"
git push
```

Requires **Hugo 0.146.0 or higher**.
