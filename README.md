# UAI Protocol Documentation

Source for [docs.unifiedai.org.in](https://docs.unifiedai.org.in) — the official documentation site for the Unified AI Interface (UAI) Protocol. Built with [Hugo](https://gohugo.io/) and the [Docsy](https://www.docsy.dev/) theme, deployed to GitHub Pages on every push to `master`.

---

## Prerequisites

| Tool | Required version | Install |
|------|-----------------|---------|
| **Hugo (extended)** | **0.163.3** | See below |
| Go | 1.22+ | https://go.dev/dl/ |
| Node.js | 22+ | https://nodejs.org/ |
| Git | any recent | https://git-scm.com/ |

> **Hugo must be the _extended_ variant.** The extended build includes the SCSS/Sass compiler that Docsy requires. The standard build will fail with a missing-CSS error.

### Installing Hugo extended 0.163.3

**macOS (Homebrew)**
```bash
brew install hugo
# verify — output must say "extended"
hugo version
# hugo v0.163.3+extended ...
```

If Homebrew ships a different version, install directly from the release:
```bash
# macOS arm64
curl -L https://github.com/gohugoio/hugo/releases/download/v0.163.3/hugo_extended_0.163.3_darwin-universal.tar.gz \
  | tar xz hugo
sudo mv hugo /usr/local/bin/hugo
```

**Linux (amd64)**
```bash
curl -L https://github.com/gohugoio/hugo/releases/download/v0.163.3/hugo_extended_0.163.3_linux-amd64.tar.gz \
  | tar xz hugo
sudo mv hugo /usr/local/bin/hugo
```

**Windows** — download `hugo_extended_0.163.3_windows-amd64.zip` from the [releases page](https://github.com/gohugoio/hugo/releases/tag/v0.163.3) and add to your `PATH`.

---

## Fresh clone and first build

```bash
# 1. Clone
git clone https://github.com/<org>/uai_handbook.git
cd uai_handbook

# 2. Install PostCSS toolchain (required by Docsy)
npm install

# 3. Pull the Docsy theme via Hugo Modules
hugo mod download

# 4. Build the site into ./public
hugo --minify
```

A successful build prints a summary like:
```
Pages            │ 57
...
Total in ~700 ms
```

The generated site is in `./public/`. Open `public/index.html` in a browser to verify locally, or use `hugo server` (see below).

---

## Local development

```bash
hugo server
```

Hugo starts a live-reload server at **http://localhost:1313/**. Every file save triggers an instant rebuild — no manual refresh needed.

Useful flags:

| Flag | Purpose |
|------|---------|
| `--buildDrafts` | Include pages with `draft: true` |
| `--disableFastRender` | Full rebuild on each change (slower but safer for layout edits) |
| `--port 1314` | Use a different port |

```bash
# Example: serve drafts on a custom port
hugo server --buildDrafts --port 1314
```

---

## Branch and PR workflow

The `master` branch is the published branch — every push deploys to production automatically.

**Never commit directly to `master`.** The workflow for all changes:

```bash
# 1. Create a feature branch from master
git checkout master
git pull origin master
git checkout -b docs/my-new-page

# 2. Make changes, verify locally
hugo server

# 3. Commit and push
git add content/docs/...
git commit -m "docs: add page on <topic>"
git push -u origin docs/my-new-page

# 4. Open a PR against master
#    CI will not run a preview build — verify locally before merging.

# 5. After review, merge to master → GitHub Actions deploys automatically
```

Branch naming conventions:

| Prefix | When to use |
|--------|-------------|
| `docs/` | New or updated content pages |
| `fix/` | Broken links, typos, formatting |
| `style/` | CSS/SCSS changes (`assets/scss/`) |
| `ci/` | Workflow or build changes |

---

## Adding a new documentation page

### 1. Choose the right section

Content lives under `content/docs/`. Current sections:

```
content/docs/
├── introduction/     # What UAI is, architecture
├── concepts/         # Trust tiers, corridors, DIDs, consent, receipts
├── protocol/         # Flows, JWT profile, DPoP
├── schemas/          # JSON Schema definitions
├── services/         # Registry, Trust Server, Audit Ledger, Adapters
├── api-reference/    # HTTP endpoint reference
├── guides/           # Getting started, provider implementation
└── reference/        # Make targets, env vars, compliance, performance
```

### 2. Create the file

```bash
# Add a new page to an existing section
hugo new content/docs/concepts/my-new-concept.md
```

Or create the file manually — the front matter format is:

```markdown
---
title: "My New Concept"
linkTitle: "My New Concept"
weight: 5
description: "One-sentence description shown in section indexes and SEO."
---

Page content in Markdown goes here.
```

- **`title`** — full title shown at the top of the page
- **`linkTitle`** — shorter label shown in the sidebar (omit if same as `title`)
- **`weight`** — controls sort order within the section; lower = higher in the list
- **`description`** — shown in the section index card and as the meta description

### 3. Adding a new section

To add a top-level section (e.g. `content/docs/security/`):

```bash
mkdir content/docs/security
```

Create an `_index.md` inside it — this is the section landing page:

```markdown
---
title: "Security"
linkTitle: "Security"
weight: 9
description: "Security model, threat considerations, and hardening guidance."
---

Section overview content here.
```

Hugo picks it up automatically; the sidebar entry appears ordered by `weight`.

### 4. Verify before pushing

```bash
hugo server --buildDrafts
```

Check the new page renders correctly, appears in the sidebar, and has no broken links.

---

## Project structure

```
uai_handbook/
├── content/docs/          # All documentation pages (Markdown)
├── assets/scss/           # Custom style overrides
│   ├── _variables_project.scss   # Bootstrap/Docsy variable overrides
│   └── _styles_project.scss      # Free-form CSS additions
├── static/                # Static files copied as-is (favicon, CNAME)
├── assets/icons/          # Site logo SVG
├── hugo.toml              # Hugo configuration
├── go.mod                 # Hugo module dependencies (Docsy theme)
├── package.json           # Node dependencies (PostCSS toolchain)
└── .github/workflows/     # GitHub Actions CI/CD
    └── hugo.yml           # Build and deploy to GitHub Pages
```

## Deployment

Deployment is fully automated. Merging to `master` triggers `.github/workflows/hugo.yml`, which:

1. Builds the site with `hugo --minify --baseURL <github-pages-url>`
2. Uploads the `./public` artifact
3. Deploys to GitHub Pages

The live site is served from the custom domain configured in `static/CNAME` (`docs.unifiedai.org.in`).

No manual deploy step is needed.
