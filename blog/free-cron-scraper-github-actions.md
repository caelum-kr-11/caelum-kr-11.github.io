---
layout: default
title: Run a web scraper on a schedule for free with GitHub Actions
description: A free, serverless cron scraper. GitHub Actions runs it and commits the data back to git (git scraping).
---
# Run a web scraper on a schedule for free with GitHub Actions (no server)

Need a scraper that runs every day, keeps a history, and costs nothing? GitHub Actions can do it. The workflow runs your script on a cron schedule and commits the output back to the repo. Your git history becomes a versioned dataset, and every change is a diff. This pattern is often called *git scraping*.

## The whole setup is one workflow file

```yaml
# .github/workflows/scrape.yml
name: scrape
on:
  schedule:
    - cron: '17 6 * * *'   # daily at 06:17 UTC
  workflow_dispatch:        # adds a "Run workflow" button
permissions:
  contents: write           # lets the job push commits
jobs:
  scrape:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: for f in scrapers/*.mjs; do node "$f"; done
      - run: |
          git config user.name "scraper-bot"
          git config user.email "actions@users.noreply.github.com"
          git add data
          git diff --cached --quiet && exit 0
          git commit -m "data: $(date -u +%F)" && git push
```

The last step only commits when the data actually changed, so the history stays clean.

## A scraper without any dependencies

Node 20 has `fetch` built in. You don't need `npm install`:

```js
// scrapers/hn-top.mjs: the Hacker News front page via the official API
import { writeFileSync, mkdirSync } from 'node:fs';
const ids = (await (await fetch('https://hacker-news.firebaseio.com/v0/topstories.json')).json()).slice(0, 30);
const items = await Promise.all(ids.map(async (id) =>
  (await fetch(`https://hacker-news.firebaseio.com/v0/item/${id}.json`)).json()));
mkdirSync('data', { recursive: true });
writeFileSync('data/hn-top.csv', 'rank,title,points\n' +
  items.map((it, i) => `${i + 1},"${(it.title ?? '').replace(/"/g, '""')}",${it.score}`).join('\n') + '\n');
```

Push both files, open the **Actions** tab, and click **Run workflow**. A minute later there's a `data: …` commit.

## Reading the history

```bash
git log -p data/hn-top.csv              # every change, as a diff
git log --format='%ad %s' -- data/      # when things changed
```

GitHub's web UI also shows each commit's diff, so you get "what changed since yesterday" for free.

## Gotchas
- **Push denied?** Settings → Actions → General → Workflow permissions → *Read and write*.
- **Cron pauses** after 60 days without repo activity. Re-enable it on the Actions tab.
- **Free minutes:** unlimited for public repos, 2,000 a month for private ones. A small daily scraper uses about 1 minute per run.
- **Be polite:** scrape public data only, respect each site's terms and robots.txt, prefer official APIs, and keep schedules gentle.

## Template
I packaged this as a template repo with a working example (a sandbox book catalogue → CSV): **[caelum-kr-11/git-scraper-kit](https://github.com/caelum-kr-11/git-scraper-kit)**. Click *Use this template* and you have a working scheduled scraper in about two minutes.

*This article and the template were written with AI assistance and tested on GitHub Actions.*
