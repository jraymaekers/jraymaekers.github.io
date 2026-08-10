# Design: Hugo-driven terminal posts command

**Date:** 2026-08-07  
**Status:** Approved

## Problem

`data/terminal/commands/posts.yaml` contains a hardcoded `items[]` list of blog posts. This list is manually maintained and will drift out of sync with actual published content in `content/posts/`.

## Solution

Replace the static `items[]` list with a Hugo template that reads live content at build time. The YAML file retains command metadata only; the partial iterates `site.RegularPages` directly.

## Files changed

| File | Change |
|---|---|
| `data/terminal/commands/posts.yaml` | Remove `items[]` array; keep `name`, `label`, `description`, `quick`, `weight` |
| `layouts/_partials/terminal/commands/posts.html` | Replace `.items` loop with `site.RegularPages` — 5 most recent, sorted by date descending |

## No changes to

- `data/terminal/config.yaml` — command registration unchanged
- `layouts/_partials/terminal/command-manager.html` — no structural change
- Any other layout, data, or content file

## Behaviour

- The `posts` command and quick-command bar button continue to work exactly as before
- Output shows the 5 most recent non-draft posts, sorted newest first
- Each entry renders: title (linked to post URL), date, summary — same visual style as current partial
- Adding a new post automatically appears in the terminal on next build; no YAML edits required

## Constraints

- Hugo `site.RegularPages` excludes draft posts (`draft: true`) automatically
- The 5-post cap is a hardcoded `first 5` slice — not configurable via YAML (can be made configurable later if needed)
- Post front matter must include `title`, `date`, and `summary` for full output; missing `summary` suppresses the summary line entirely (no crash, no blank line)
