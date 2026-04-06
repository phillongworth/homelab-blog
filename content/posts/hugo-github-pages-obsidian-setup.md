---
title: "Hugo Blog with GitHub Pages and Obsidian — Setup Guide"
date: 2026-04-06T00:00:00Z
draft: false
tags: ["hugo", "obsidian", "github-pages", "homelab", "meta"]
categories: ["Guides"]
description: "A complete walkthrough for setting up a self-hosted Hugo blog with PaperMod, deployed automatically to GitHub Pages, with Obsidian as the writing interface."
showToc: true
---

This post documents exactly how this blog was set up — from zero to a fully automated publish pipeline with Obsidian as the writing interface. Follow these steps to replicate it from scratch.

---

## Prerequisites

- A [GitHub](https://github.com) account
- [Git](https://git-scm.com/downloads) installed
- [Obsidian](https://obsidian.md) installed
- Windows machine (steps are Windows-specific but easily adapted)

---

## 1. Install Hugo (Extended)

Hugo Extended is required — the standard version does not support all theme features.

Open an elevated PowerShell terminal and run one of the following:

```powershell
# Winget (built into Windows 11)
winget install Hugo.Hugo.Extended

# OR Scoop
scoop install hugo-extended

# OR Chocolatey
choco install hugo-extended
```

Verify the install:

```powershell
hugo version
# Should output: hugo v0.146.0+extended ...
```

> The version must be **0.146.0 or higher** for the PaperMod theme to work.

---

## 2. Create the Site Structure

Hugo isn't strictly needed to create the folder structure — create it manually or run:

```powershell
hugo new site D:\homelab-blog
cd D:\homelab-blog
```

The directory structure used by this blog:

```
homelab-blog/
├── .github/workflows/hugo.yml   ← GitHub Actions deploy pipeline
├── archetypes/                  ← front matter templates
├── content/
│   ├── posts/                   ← long-form write-ups
│   ├── notes/                   ← quick references
│   ├── snippets/                ← configs and one-liners
│   ├── Templates/               ← Obsidian Templater templates
│   └── search.md
├── themes/PaperMod/             ← git submodule
└── hugo.toml                    ← site config
```

---

## 3. Install the PaperMod Theme

Initialise a Git repo and add PaperMod as a submodule:

```powershell
cd D:\homelab-blog
git init
git checkout -b main
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

---

## 4. Configure hugo.toml

Create `D:\homelab-blog\hugo.toml` with the following content, replacing the placeholders with your own values:

```toml
baseURL = "https://<YOUR_GITHUB_USERNAME>.github.io/homelab-blog/"
languageCode = "en-us"
title = "Homelab"
theme = "PaperMod"
googleAnalytics = ""

[pagination]
  pagerSize = 10

enableRobotsTXT = true
ignoreFiles = ["/content/Templates/"]
buildDrafts = false
buildFuture = false
buildExpired = false

[minify]
  disableXML = true
  minifyOutput = true

[params]
  env = "production"
  title = "Homelab"
  description = "Notes, configs, and experiments from the homelab."
  author = "<YOUR_NAME>"
  defaultTheme = "dark"
  ShowReadingTime = true
  ShowShareButtons = false
  ShowPostNavLinks = true
  ShowBreadCrumbs = true
  ShowCodeCopyButtons = true
  ShowRssButtonInSectionTermList = true
  UseHugoToc = true
  showtoc = true
  tocopen = false

  [params.homeInfoParams]
    Title = "Welcome to the Homelab"
    Content = "Self-hosted services, infrastructure notes, and configs."

  [[params.socialIcons]]
    name = "github"
    url = "https://github.com/<YOUR_GITHUB_USERNAME>"
  [[params.socialIcons]]
    name = "rss"
    url = "/index.xml"

[markup]
  defaultMarkdownHandler = "goldmark"

  [markup.goldmark.extensions]
    definitionList = true
    footnote = true
    linkify = true
    strikethrough = true
    table = true
    taskList = true
    typographer = false

  [markup.goldmark.parser]
    autoHeadingID = true
    autoHeadingIDType = "github"

  [markup.goldmark.parser.attribute]
    block = true
    title = true

  [markup.goldmark.renderer]
    hardWraps = false
    unsafe = false

  [markup.highlight]
    codeFences = true
    guessSyntax = true
    lineNos = false
    noClasses = false
    style = "dracula"
    tabWidth = 4

  [markup.tableOfContents]
    startLevel = 2
    endLevel = 4
    ordered = false

[outputs]
  home = ["HTML", "RSS", "JSON"]

[permalinks]
  posts    = "/posts/:slug/"
  notes    = "/notes/:slug/"
  snippets = "/snippets/:slug/"

[[menus.main]]
  identifier = "posts"
  name = "Posts"
  url = "/posts/"
  weight = 10

[[menus.main]]
  identifier = "notes"
  name = "Notes"
  url = "/notes/"
  weight = 20

[[menus.main]]
  identifier = "snippets"
  name = "Snippets"
  url = "/snippets/"
  weight = 30

[[menus.main]]
  identifier = "search"
  name = "Search"
  url = "/search/"
  weight = 40

[[menus.main]]
  identifier = "tags"
  name = "Tags"
  url = "/tags/"
  weight = 50
```

> **Common pitfalls:**
> - Do not use `paginate =` — it was deprecated in Hugo 0.128.0. Use `[pagination] pagerSize` instead.
> - `[[params.socialIcons]]` must use double brackets. A single `[params.socialIcons]` above it will cause a TOML parse error.
> - `googleAnalytics = ""` must be present even if blank — PaperMod references this partial and will error if it is missing.

---

## 5. Create a .gitignore

Create `D:\homelab-blog\.gitignore`:

```
/public/
/resources/_gen/
/.hugo_build.lock
.DS_Store
Thumbs.db
.obsidian/
```

---

## 6. Create the GitHub Actions Workflow

Create `.github/workflows/hugo.yml`:

```yaml
name: Deploy Hugo to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: "pages"
  cancel-in-progress: true

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: "0.146.0"
    steps:
      - name: Checkout (with submodules for theme)
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Install Hugo (extended)
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: ${{ env.HUGO_VERSION }}
          extended: true

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Build
        env:
          HUGO_ENVIRONMENT: production
          HUGO_BASEURL: ${{ steps.pages.outputs.base_url }}/
        run: |
          hugo \
            --minify \
            --baseURL "$HUGO_BASEURL"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

> Use `peaceiris/actions-hugo@v3` rather than manually downloading a `.deb` — it handles versioning reliably and surfaces clear errors.

---

## 7. Push to GitHub

Create a new **empty** repository on GitHub named `homelab-blog`, then:

```powershell
git remote add origin https://github.com/<YOUR_GITHUB_USERNAME>/homelab-blog.git
git add .
git commit -m "feat: initial Hugo homelab blog"
git push -u origin main
```

Then go to your repo on GitHub: **Settings → Pages → Source → GitHub Actions**.

Your site will be live at `https://<YOUR_GITHUB_USERNAME>.github.io/homelab-blog/` after the first Action completes.

---

## 8. Set Up Obsidian

### Open the vault

In Obsidian: **Open folder as vault** → select `D:\homelab-blog\content`

> Open `content/` as the vault root — **not** the repo root. Obsidian needs to be inside `content/` so it can see the `Templates` folder and your posts side by side.

### Install plugins

Go to **Settings → Community plugins → Browse** and install:

| Plugin | Purpose |
|---|---|
| **Templater** | Auto-fills front matter on new notes |
| **Git** | Push to GitHub directly from Obsidian |

### Create the Templater template

Create `content/Templates/post.md`:

```markdown
---
title: "hugo-github-pages-obsidian-setup"
date: 2026-04-06T15:35:32+01:00
draft: true
tags: []
description: ""
showToc: true
---

```

### Configure Templater

**Settings → Templater:**

1. **Template folder location** → `Templates`
2. Enable **Trigger Templater on new file creation**
3. **Folder templates → Add new:**
   - Folder: `posts`
   - Template: `Templates/post`

### Configure Git plugin

**Settings → Git:**

- **Sync method** → `Commit-and-sync`
- **Auto pull interval** → `10` minutes
- **Commit message** → `docs: {{date}} {{hostname}}`

---

## 9. Writing and Publishing

1. In Obsidian, create a new file inside `posts/` — front matter fills automatically
2. Write your post in Markdown
3. When ready to publish, change `draft: true` → `draft: false`
4. Press `Ctrl+P` → **Git: Commit-and-sync**
5. GitHub Actions builds and deploys — live in ~30 seconds

---

## Syntax Highlighting

Code blocks use the **Dracula** theme via Hugo's built-in Chroma highlighter. Specify the language after the opening fence for full highlighting:

````markdown
```bash
ss -tulpn | grep :8080
```

```yaml
services:
  nginx:
    image: nginx:alpine
```
````

To change the theme, update `style` in `hugo.toml` under `[markup.highlight]`. A full list of available styles can be found at [xyproto.github.io/splash/docs](https://xyproto.github.io/splash/docs/all.html).
