# Get a free alert when a web page changes (price, stock, version), using GitHub Actions

{% raw %}

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
      - name: Keepalive (GitHub pauses cron after 60 days without commits)
        run: |
          [ $(( $(date +%s) - $(git log -1 --format=%ct) )) -lt 4320000 ] && exit 0
          git config user.name bot && git config user.email bot@users.noreply.github.com
          git commit --allow-empty -m "keepalive: still watching, last value $(cat value.txt)" && git push
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
- GitHub pauses scheduled workflows in public repos after 60 days without activity. A price that stays flat would then look exactly like "no change". The keepalive step makes an empty commit once 50 days have passed without one, which keeps the schedule alive and doubles as a "still watching" heartbeat.
- Scheduled runs can be delayed or skipped when Actions is busy, so treat `*/30` as a target, not a guarantee.
- Public repos get unlimited Actions minutes. Private repos get 2,000 a month, and each run is billed as a full minute, so every 30 minutes (about 1,440 a month) fits.

## Want it ready-made?
The free **[git-scraper-kit](https://github.com/caelum-kr-11/git-scraper-kit)** template does scheduled scraping with history. The **[Pro Pack](https://caelum-kr-11.github.io/#buy)** (12 USDC) adds a multi-page watcher like the one above, alerts with proper Discord, Slack and ntfy formatting that only fire for the files you choose, and six more ready-made scrapers.

*Written with AI assistance; the code was tested before publishing.*

Thanks to holdalarms on DEV for pointing out the 60-day pause.

{% endraw %}
