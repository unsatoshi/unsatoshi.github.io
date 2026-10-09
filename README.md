# unsatoshi.github.io

Personal site of **notSatoshi** — smart contract security research and DeFi
bug-hunting field notes.

> Trust God. Fuzz Code.

- **Live:** https://unsatoshi.github.io
- **X:** [@0xunsatoshi](https://x.com/0xunsatoshi)

## Writing a post

Add a Markdown file under `_posts/` named `YYYY-MM-DD-title.md`:

```markdown
---
layout: post
title: "Your Title"
date: 2026-10-09 09:00:00 +0000
categories: [patterns, evm]
tags: [lending, defi]
description: "One-line summary for SEO and the card."
---

Intro paragraph.

<!--more-->

Rest of the post.
```

Commit and push to `gh-pages`; GitHub Pages builds and deploys automatically.

## Local preview (optional)

Requires Ruby and Bundler:

```bash
bundle install
bundle exec jekyll serve
```

## Credits

Built on the [White Paper](https://github.com/vinitkumar/white-paper) Jekyll
theme (MIT). See `LICENSE`.
