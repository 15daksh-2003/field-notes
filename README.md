# Field Notes

A tiny blog. Plain Jekyll, no theme gems — GitHub Pages builds it on push.

## Write a post

Drop a Markdown file in `_posts/` named `YYYY-MM-DD-title.md` with this header:

```markdown
---
layout: post
title: "Your title"
date: 2026-09-17
---

Write here.
```

That's the whole workflow.

## Publish

1. Create a repo on GitHub named `field-notes` (or `<username>.github.io` for a root site).
2. Push this folder to it.
3. Repo → Settings → Pages → Source: **Deploy from a branch**, branch `main`, folder `/`.
4. It goes live at `https://<username>.github.io/field-notes/`.

If you use a root `<username>.github.io` repo instead, set `baseurl: ""` in `_config.yml`.

## Preview locally (optional)

Needs Ruby + Bundler:

```bash
gem install bundler jekyll
jekyll serve
```

Open http://localhost:4000/field-notes/.
