# Commerce of Agents Website

A static site for the Commerce of Agents movement. The homepage is `index.html`,
the Market Protocol v0.2 note is `market-protocol-0-2.html`, and the seller-agent
integration guide is published as raw Markdown at `seller-agent-v2.txt`. The
seller adapter retains its v2 wire identifiers; `market-protocol-2.html` redirects
old article links to the corrected v0.2 note.

## Deploy on GitHub Pages

GitHub Pages publishes the root of `master` at
`https://www.commerceofagents.com/`. The `.txt` guide contains Markdown and is
served as plain text for agents and developer tools. The checked-in `.nojekyll`
file keeps GitHub Pages from processing the static files. A push to `master`
publishes the site.

Serve the directory locally with `python3 -m http.server 8000` to inspect the
HTML pages and raw Markdown links before publishing.

## Visual assets

The October 2026 redesign uses ink, electric lime, and violet. `favicon.svg`
is the shared arrow mark. `og.png` is the 1200 × 630 social card, derived from
`og-art.png`; `agent-orbits.webp` is the cropped hero artwork. Both pages use
versioned social-image and favicon URLs to refresh previously cached previews.

Artwork was generated with the built-in image generation tool using this prompt:

> Use case: ads-marketing. Create a polished widescreen 1200x630 social sharing artwork for Commerce of Agents, a movement about AI agents discovering, negotiating and transacting for people. Art direction: playful premium digital culture editorial poster, very dark ink background, electric chartreuse and lavender, sculptural glossy chrome interconnected orbital loops and rounded abstract agent forms on the RIGHT half, a small four-point sparkle, subtle grain, striking generous negative space. LEFT half exact large bold clean sans-serif typography stacked: 'COMMERCE' then 'OF AGENTS'. Below it smaller exact text: 'Your intent. Agents in motion.' Tiny top-left label: 'THE NEXT ERA OF COMMERCE'. All typography highly legible, comfortably inside safe margins. Sophisticated energetic and inclusive of younger internet-native audiences, no humans, no stock robot heads, no watermark. Wide 1.905:1 aspect ratio. This artwork will also be used cropped as website hero artwork on the right.
