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
- **`market-protocol-0-2.html`** — Long-form Market Protocol v0.2 note. The former `market-protocol-2.html` path redirects here.
- **`seller-agent-v2.md`** — Raw Markdown seller integration guide for Market Protocol v0.2; its Seller Commerce API v2 wire identifiers remain in use, linked from `llms.txt`.
- **`styles.css`** — All styling. Uses CSS custom properties for theming, CSS Grid for layouts, and a `clamp()`-based fluid type scale. Mobile breakpoint is at `880px`.

There is no application JavaScript. The Tally embed script resizes the waitlist iframe.

## Key Customization Points

- Contact email: `hello@commerceofagents.com` in the homepage and article footer.
- Color theme: custom properties at the top of `styles.css`; warm off-white, ink, electric lime, and violet.
- Artwork: `agent-orbits.webp` for the hero; `og.png` is the 1200 × 630 social card, with `og-art.png` as its source.
- Brand mark: `favicon.svg`, shared with the header and footer.
- Waitlist: direct Tally iframe plus the resize script and a fallback signup link.
- Domain: `CNAME` contains `www.commerceofagents.com`.
