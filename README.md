# Mr. Rizzo · Blog

Personal blog of Mr. Rizzo — technical notes (homelab, local AI, git, ServiceNow) and some everyday topics.

- **Live:** https://rizzoadm.github.io/blog/
- **Portfolio:** https://rizzoadm.github.io/
- **Languages:** English and Brazilian Portuguese

## Stack

- [Hugo](https://gohugo.io/) (extended) — static site generator
- Theme: [PaperMod](https://github.com/adityatelange/hugo-PaperMod) (git submodule in `themes/PaperMod`)
- Comments: [giscus](https://giscus.app/) (GitHub Discussions) — stage B4
- Newsletter: [Buttondown](https://buttondown.com/) (RSS-to-email) — stage B5
- Browser editor: [Pages CMS](https://pagescms.org/) — stage B6
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

A post can exist in one language only. If the two files ever live in different folders, give both the same `translationKey`.

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
- **B4** — giscus comments: next
- B5 — Buttondown · B6 — Pages CMS · B7 — link from the portfolio menu · B8 — final review
