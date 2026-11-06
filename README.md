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

- Day: cream background `#fbf9f4`, white cards, ink-blue accent `#1f3a68`
- Night: near-black background `#121312`, cards `#1e201e`, soft-cyan accent `#7cc4c0`
- Body and titles in [Literata](https://fonts.google.com/specimen/Literata) (serif)

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
- **B2** — Day/night colors and Literata font: next
- B3 — Languages and menus · B4 — giscus · B5 — Buttondown · B6 — Pages CMS · B7 — link from the portfolio menu · B8 — final review
