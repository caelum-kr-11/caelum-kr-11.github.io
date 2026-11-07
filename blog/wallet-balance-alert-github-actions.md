---
layout: default
title: "Free crypto wallet alert: Base and Ethereum balance changes via GitHub Actions"
description: Get a phone or Discord alert when a Base or Ethereum wallet balance (ETH or USDC) changes. Free, read-only, no API key or server.
---
# Free crypto wallet alert: get a ping when a Base or Ethereum balance changes (GitHub Actions)

Want to know when a payment lands, when a hot wallet gets drained, or when a treasury moves? You don't need an API key, a paid monitoring bot, or a server. A scheduled GitHub Actions workflow can read balances from a public RPC, keep the last values in git, and alert you only when one changes.

Read-only: you only give it **public addresses**, never a key or seed phrase.

## 1. The checker (no dependencies)

```js
// balances.mjs: native + USDC balance for each address on Base and Ethereum
import { writeFileSync } from 'node:fs';
const NETS = {
  base: ['https://mainnet.base.org', '0x833589fcd6edb6e08f4c7c32d4f71b54bda02913'],
  ethereum: ['https://ethereum-rpc.publicnode.com', '0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48'],
};
const rpc = async (url, method, params) => {
  const r = await fetch(url, { method: 'POST', headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }) });
  const j = await r.json(); if (j.error) throw new Error(j.error.message); return BigInt(j.result);
};
let out = 'chain,address,eth,usdc\n';
for (const addr of process.env.ADDRESSES.split(',')) {
  for (const [chain, [url, usdc]] of Object.entries(NETS)) {
    const eth = await rpc(url, 'eth_getBalance', [addr, 'latest']);
    const bal = await rpc(url, 'eth_call', [{ to: usdc, data: '0x70a08231' + addr.slice(2).toLowerCase().padStart(64, '0') }, 'latest']);
    out += `${chain},${addr},${(Number(eth) / 1e18).toFixed(6)},${(Number(bal) / 1e6).toFixed(2)}\n`;
  }
}
writeFileSync('balances.csv', out);
console.log(out);
```

`0x70a08231` is the ERC-20 `balanceOf(address)` selector, so the same call works for any token. Just swap in its contract address and decimals.

## 2. The workflow: check, compare, alert

```yaml
# .github/workflows/wallet.yml
name: wallet
on:
  schedule: [{ cron: '*/15 * * * *' }]   # every 15 minutes
  workflow_dispatch:
permissions: { contents: write }
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: node balances.mjs
        env: { ADDRESSES: '0xYourAddress,0xAnotherAddress' }
      - name: Alert and commit on change
        env: { WEBHOOK: '${{ secrets.ALERT_WEBHOOK }}' }
        run: |
          git diff --quiet balances.csv && exit 0
          curl -s -d "Balance changed: $(git diff -U0 balances.csv | grep '^+[^+]')" "$WEBHOOK"
          git config user.name bot && git config user.email bot@users.noreply.github.com
          git add balances.csv && git commit -m "balances: $(date -u +%FT%TZ)" && git push
```

Commit an empty `balances.csv`, then run the workflow once by hand.

## 3. Where the alert goes
Set the repo secret `ALERT_WEBHOOK` to `https://ntfy.sh/<long-random-topic>` and subscribe to that topic in the **[ntfy](https://ntfy.sh)** phone app. Topics are public, so pick an unguessable name. Discord and Slack webhooks need a JSON body instead.

## Notes and limits
- **Privacy:** in a public repo, your addresses and balance history are public too. Use a private repo (2,000 free Actions minutes a month; each run takes about 15 seconds, so every 30 minutes fits easily) or accept that, since balances are on-chain anyway.
- GitHub cron can run several minutes late. This suits "tell me today", not trading.
- Public RPCs are rate-limited. A few addresses every 15 minutes is fine. Use your own RPC URL for more.
- Other EVM chains work the same way: change the RPC and the USDC contract address.

## Want it ready-made?
The **[Pro Pack](https://caelum-kr-11.github.io/#buy)** (12 USDC on Base) includes this as `wallet-watch`, with labels per address, alerts with proper Discord, Slack and ntfy formatting, and 7 more scrapers (crypto prices, page-watch for prices and stock, GitHub releases, RSS and more). It builds on the free **[git-scraper-kit](https://github.com/caelum-kr-11/git-scraper-kit)** template.

*Written with AI assistance; the code was tested before publishing.*
