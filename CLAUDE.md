# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a pure static website (no build step, no dependencies) for the Commerce of Agents movement, hosted on GitHub Pages at `www.commerceofagents.com`.

## Development

No build, install, or test commands exist. To preview locally, open `index.html` directly in a browser or use any static file server:

```bash
python3 -m http.server 8000
```

## Deployment

The site deploys automatically via GitHub Pages from the `master` branch root. Pushing to `master` publishes the changes live.

## Architecture

The main pages and shared styles are:

- **`index.html`** — All markup. Single-page layout with anchor-based navigation (`#thesis`, `#protocol`, `#book`, `#contact`).
- **`market-protocol-2.html`** — Long-form Market Protocol 2.0 note.
- **`seller-agent-v2.txt`** — Raw Markdown Seller Commerce API v2 guide, linked from `llms.txt`.
- **`styles.css`** — All styling. Uses CSS custom properties for theming, CSS Grid for layouts, and a `clamp()`-based fluid type scale. Mobile breakpoint is at `880px`.

There is no JavaScript.

## Key Customization Points

- **Contact email**: `index.html` line 192 — replace `hello@example.com`
- **Color theme**: CSS variables at the top of `styles.css` (background `#f7f4ee`, accent `#b36b2c`)
- **Domain**: `CNAME` file contains `www.commerceofagents.com`
