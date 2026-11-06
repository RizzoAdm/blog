# Mr. Rizzo · Blog

Personal blog of Mr. Rizzo — technical notes (homelab, local AI, git, ServiceNow) and some everyday topics.

- **Live:** https://rizzoadm.github.io/blog/
- **Portfolio:** https://rizzoadm.github.io/
- **Languages:** English and Brazilian Portuguese

## Stack

- [Hugo](https://gohugo.io/) (extended) — static site generator
- Theme: [PaperMod](https://github.com/adityatelange/hugo-PaperMod) (git submodule in `themes/PaperMod`)
- Comments: [giscus](https://giscus.app/) (GitHub Discussions)
- Browser editor: [Pages CMS](https://pagescms.org/) (`.pages.yml`)
- Hosting: GitHub Pages via GitHub Actions

## Design

| | Background | Cards | Borders | Accent |
|---|---|---|---|---|
| Day | `#fbf9f4` | `#ffffff` | `#e2ddd3` | ink blue `#1f3a68` |
| Night | `#121312` | `#1e201e` | `#30332f` | soft cyan `#7cc4c0` |

- Body and titles in [Literata](https://fonts.google.com/specimen/Literata) (variable serif), self-hosted in `static/fonts/literata/` — no requests to Google Fonts
- Code and tags in the system monospace font
- Reading column ~700 px; code blocks stay dark in both modes (syntax colors are designed for a dark background)
- Day/night follows the visitor's system setting, with a toggle in the header

### Where the design lives

| File | Purpose |
|---|---|
| `assets/css/extended/blog.css` | Color variables, typography, cards, tags, links. PaperMod loads it after the theme CSS, so it overrides without editing the theme |
| `layouts/_partials/extend_head.html` | `@font-face` for Literata and font preload |
| `static/fonts/literata/` | Literata `.woff2` files (Latin subset, weights 200–900, normal and italic) |

## Languages

English is the default language (no prefix); Brazilian Portuguese lives under `/pt-br/` with localized URLs.

| Page | English | Português |
|---|---|---|
| Home | `/blog/` | `/blog/pt-br/` |
| Posts | `/blog/posts/<folder>/` | `/blog/pt-br/artigos/<slug>/` |
| Tags | `/blog/tags/` | `/blog/pt-br/etiquetas/` |
| Archive | `/blog/archives/` | `/blog/pt-br/arquivo/` |
| Search | `/blog/search/` | `/blog/pt-br/busca/` |

- Only `tags` are used as taxonomy (no categories)
- The language switcher in the header opens the translation of the current page, or the other language's home when there is none (`layouts/_partials/header.html`, override of the theme partial)
- Interface strings: `i18n/pt-br.yaml` (full PT-BR set — the theme's `pt.yaml` is not used for `pt-br`) and `i18n/en.yaml` (small English overrides)
- Home text, description and menu are set per language in `hugo.toml`

### Bilingual posts

Put both versions in the same folder; Hugo links them as translations:

```
content/posts/my-post/
├── index.md         # English
└── index.pt-br.md   # Portuguese — add slug: "titulo-em-portugues"
```

Both files also carry `translationKey: "<folder>"` (the Pages CMS editor writes it and uses it to name the folder). A post can exist in one language only.

## Comments

Comments use [giscus](https://giscus.app/): each post is linked to a GitHub Discussion in the **Comments** category (Announcement format — only the maintainer and the giscus app open discussions; anyone with a GitHub account can comment and react).

- Shown on posts only (`posts` section). Add `comments: false` to a post's front matter to turn them off for that post
- One discussion per post, shared by both languages: the key is `posts/<translationKey>` — the folder name when `translationKey` is absent (giscus `specific` mapping, strict matching), so the English and Portuguese versions show the same comments and changing a slug or URL keeps them. `commentsTerm` in the front matter overrides the key
- The widget interface follows the page language (`en` / `pt`)
- Reactions enabled; the widget loads lazily when the reader scrolls near it
- The widget theme follows the blog's day/night toggle. Production builds use custom themes with the blog colors; `hugo server` falls back to giscus' built-in `light`/`dark`, because giscus cannot fetch CSS from `localhost`

| File | Purpose |
|---|---|
| `layouts/_partials/comments.html` | giscus widget (override of the theme's empty partial) |
| `static/giscus/light.css`, `static/giscus/dark.css` | Custom widget themes (blog colors + Literata) |
| `hugo.toml` → `[params.giscus]` | Repository, category and their IDs (from giscus.app). The widget is hidden while an ID is empty |

## Browser editor (Pages CMS)

Posts can be written and edited at https://app.pagescms.org (sign in with GitHub; the Pages CMS GitHub App is installed on this repository only). Configuration: `.pages.yml` at the repository root.

| Menu | Files | Notes |
|---|---|---|
| **Posts (English)** | `content/posts/<key>/index.md` | `key` = `translationKey` = folder name and English URL |
| **Artigos (Português)** | `content/posts/<key>/index.pt-br.md` | Same `key` as the English version; `slug` sets the Portuguese URL |

- Fields: title, key, date, draft, tags, summary, comments, body (visual editor with a Markdown source toggle). PT also has `slug`
- New posts start as drafts (`draft: true`); unticking *Draft* and saving publishes on the next deploy
- Every save is a commit to `main`, which triggers the deploy workflow. Run `git pull` before editing locally
- Images uploaded through the editor go to `static/images/` and are linked as `/blog/images/<file>`. Strip EXIF/GPS metadata before uploading
- Unknown front-matter keys are kept on save (`settings.content.merge: true`); key and folder cannot be renamed from the editor

## Local preview

```bash
hugo server
```

Then open http://localhost:1313/blog/

## Deployment

Every push to `main` triggers `.github/workflows/hugo.yml`, which builds the site with Hugo extended **0.169.0** and deploys it to GitHub Pages (Settings → Pages → Source: GitHub Actions). Drafts (`draft: true`) are never published.

After cloning on a new machine, fetch the theme with:

```bash
git submodule update --init --recursive
```

To include drafts in the local preview:

```bash
hugo server -D
```

## Status

- **B1** — Hugo + PaperMod, EN/PT-BR, GitHub Pages via Actions: **done**
- **B2** — Day/night colors and self-hosted Literata font: **done**
- **B3** — Languages, menus, PT-BR interface and localized URLs: **done**
- **B4** — giscus comments (shared EN/PT thread, day/night themes): **done**
- B5 — Newsletter: **cancelled** (readers can follow via RSS: `/blog/index.xml` and `/blog/pt-br/index.xml`)
- **B6** — Pages CMS browser editor (separate EN/PT lists, shared folder per post): **done**
- B7 — link from the portfolio menu · B8 — final review
