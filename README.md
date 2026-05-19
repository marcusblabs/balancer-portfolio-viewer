# Balancer Portfolio Viewer

Read-only portfolio viewer for any Ethereum-style wallet on Balancer v2 / v3, across every supported chain. Paste an address — no wallet connection needed.

Data comes from Balancer's public GraphQL API (`https://api-v3.balancer.fi/`). No backend, no build step, no API keys.

## Features

- **Address-based lookup** — paste any wallet to see its Balancer LP positions
- **Cross-chain** — pulls positions from Ethereum, Arbitrum, Base, Polygon, Optimism, Gnosis, Avalanche, Sonic, Fraxtal, Mode, HyperEVM, Monad, and more
- **24h volume column** for every pool the wallet is in
- **Balance breakdown tooltip** — total at a glance, hover for wallet vs. staked (gauge, Aura, veBAL, Reliquary)
- **Filters** — chain, pool type, version (v2/v3), minimum balance, and full-text search
- **CSV export** of the currently filtered positions
- **Shareable URL** — the address is appended to the URL hash so a link captures the view (e.g. `…/#0x849D…039d`)
- **Responsive** layout for desktop, tablet, and phone

## Run locally

Just open `index.html` in a browser, or serve the directory:

```bash
python3 -m http.server 8080
# then open http://localhost:8080/
```

## Sibling explorers

- [balancer-pool-explorer](https://marcusblabs.github.io/balancer-pool-explorer/) — all Balancer pools across all chains
- [Monad-lending-opportunities](https://marcusblabs.github.io/Monad-lending-opportunities/) — Monad lending vault rates

## Deploy

Any static host works (GitHub Pages, Netlify, Vercel, Cloudflare Pages). This repo is set up for GitHub Pages from the `main` branch root.
