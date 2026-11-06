---
layout: default
title: Get a free alert when a web page changes, using GitHub Actions
description: Watch a price, stock status or version on a schedule with GitHub Actions and get a phone or Discord alert only when it changes.
---
# Get a free alert when a web page changes (price, stock, version), using GitHub Actions

Want a ping on your phone or in Discord when a product's price drops, an item comes back in stock, or a new version ships? You don't need a paid monitoring service. GitHub Actions can check the page on a schedule, keep the last value in git, and alert you only when it changes.

## 1. Workflow: check, compare, alert

```yaml
# .github/workflows/watch.yml
name: watch
on:
  schedule: [{ cron: '*/30 * * * *' }]   # every 30 minutes
  workflow_dispatch:
permissions: { contents: write }
jobs:
  watch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: node watch.mjs
        env:
          URL: https://nodejs.org/dist/index.json
          PATTERN: '"version":"(v\d+\.\d+\.\d+)"'
      - name: Alert and commit if the value changed
        env: { WEBHOOK: '${{ secrets.ALERT_WEBHOOK }}' }
        run: |
          git diff --quiet value.txt && exit 0
          curl -s -d "Changed: $(cat value.txt)" "$WEBHOOK"
          git config user.name bot && git config user.email bot@users.noreply.github.com
          git commit -am "value: $(cat value.txt)" && git push
```

## 2. The checker (no dependencies)

```js
// watch.mjs: write the first regex capture group from URL to value.txt
import { writeFileSync } from 'node:fs';
const html = await (await fetch(process.env.URL, { headers: { 'user-agent': 'page-watch (personal use)' } })).text();
const value = html.match(new RegExp(process.env.PATTERN))?.[1] ?? 'NOT_FOUND';
writeFileSync('value.txt', value + '\n');
console.log(value);
```

Commit an empty `value.txt`, then run the workflow once by hand.

## 3. Where the alert goes
`curl -d "text" URL` works with **[ntfy.sh](https://ntfy.sh)** out of the box. Set the secret `ALERT_WEBHOOK` to `https://ntfy.sh/<long-random-topic>` and subscribe in the ntfy phone app. Topics are public, so use an unguessable name. Discord and Slack webhooks need a JSON body instead: `curl -H 'content-type: application/json' -d '{"content":"…"}' URL`.

## Useful patterns
| What | Regex |
|---|---|
| Price in JSON-LD (most shops) | `"price":\s*"?([\d.]+)` |
| Stock status | `(InStock\|OutOfStock)` |
| Page title | `<title>([^<]+)</title>` |

## Notes
- Prefer an official API or feed when there is one; it's more stable than HTML.
- Check the site's terms and robots.txt, and keep the interval reasonable. Every 30–60 minutes is plenty for prices.
- Public repos get unlimited Actions minutes. Private repos get 2,000 a month, and each check takes about 15 seconds.

## Want it ready-made?
The free **[git-scraper-kit](https://github.com/caelum-kr-11/git-scraper-kit)** template does scheduled scraping with history. The **[Pro Pack](https://caelum-kr-11.github.io/#buy)** (12 USDC) adds a multi-page watcher like the one above, alerts with proper Discord, Slack and ntfy formatting that only fire for the files you choose, and six more ready-made scrapers.

*Written with AI assistance; the code was tested before publishing.*
