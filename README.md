# Commerce of Agents Website

A static site for the Commerce of Agents movement. The homepage is `index.html`,
the Market Protocol 2.0 note is `market-protocol-2.html`, and the seller-agent
integration guide is published as raw Markdown at `seller-agent-v2.txt`.

## Deploy on GitHub Pages

GitHub Pages publishes the root of `master` at
`https://www.commerceofagents.com/`. The `.txt` guide contains Markdown and is
served as plain text for agents and developer tools. The checked-in `.nojekyll`
file keeps GitHub Pages from processing the static files. A push to `master`
publishes the site.

Serve the directory locally with `python3 -m http.server 8000` to inspect the
HTML pages and raw Markdown links before publishing.
