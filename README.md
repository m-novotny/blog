# isaagi.cloud

Source for my personal site. Built with [Zola](https://www.getzola.org/).

## Build

```bash
zola build
```

Output goes to `public/`. Deploy with Cloudflare Pages or GitHub Pages.

## Structure

```
content/     — markdown posts
templates/   — Tera templates
static/      — CSS, images
config.toml  — site config
```
// 2024-06-15 — Add RSS feed configuration and metadata in config.toml
// 2025-03-22 — Update Zola to 0.19 and fix deprecated config options
