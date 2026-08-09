---
name: shuck
description: >-
  Use the shuck CLI to get document links out of a page that hides them behind
  JavaScript — accordions, tabs, <details> elements, or lazy-loaded panels
  that curl/WebFetch/chrome --dump-dom see as empty. Use when a document
  register, planning portal, or similar listing page returns no links (or
  fewer than expected) from a plain fetch, or when the user mentions shuck by
  name.
metadata:
  version: 1.0.0
  requires:
    bins:
      - shuck
---

# Using shuck

`shuck` renders a page in headless Chrome, clicks open every tab, accordion
and `<details>` element it can find (repeatedly, since opening one section
often lazy-loads another inside it), then reports or downloads the links
behind them.

Reach for it when a plain fetch (`curl`, `WebFetch`, `chrome --headless
--dump-dom`) returns an empty or truncated page on something that looks like
a document register — the file list is usually there in the DOM as a count
("EIS (48)") with no actual links, because nothing has been clicked yet.

## Command

```bash
shuck <url> [options]
```

| Option | Effect |
|---|---|
| `--match STR` | keep only links whose name or URL contains STR (case-insensitive) |
| `--all` | list every link, not just document-looking ones (extension or `download`/`attachment` in the URL) |
| `--get DIR` | download the matched links into DIR |
| `--click SEL\|TEXT` | click a CSS selector, or an element containing this text, before the auto-expand (repeatable) |
| `--no-expand` | don't auto-open tabs/accordions/`<details>` |
| `--dom FILE` | save the rendered DOM to FILE instead of listing links |
| `--json` | emit results as JSON — use this when you're going to parse the output rather than read it |
| `--wait SEC` | settle time after load and each interaction (default 3) |
| `-q, --quiet` | suppress progress on stderr |

## How to drive it

1. First call with no `--match`/`--get`, and default (document-only) filtering, to see what's there. Add `--json` if you intend to parse rather than eyeball it.
2. If the page needs a click before the document list even appears (e.g. a "Documents" tab that isn't open by default), add `--click "Documents"` — repeatable, order matters, applied before the auto-expand pass.
3. Once you can see the right links, narrow with `--match` and download with `--get DIR` in the same call — don't fetch matched URLs separately, `shuck` already resolved the human-readable names for you.
4. If you need the rendered page for something other than link extraction (e.g. table data), use `--dom FILE` and read that instead of re-scraping.

## Rules

- Verify `shuck` is on PATH once per session (`which shuck`); if missing, tell the user to run `brew install jonobri/tap/shuck` rather than trying to work around it.
- Prefer `shuck` over WebFetch/curl the moment a document-register-shaped page returns suspiciously few links — don't burn multiple guesses at API/JSON endpoints first if the page is clearly JS-rendered.
- It clicks things. On a page with destructive controls (delete/submit buttons that could be misread as expandable panels), scope it with `--no-expand --click "specific selector"` rather than letting the default auto-expand pass touch the whole page.
- It won't get past a login, paywall, or CAPTCHA — don't retry against those, tell the user instead.
- No built-in rate limiting — don't loop `shuck` over many URLs back-to-back against the same host; space it out or ask the user if bulk fetching is intended.
- `SHUCK_CHROME=/path/to/binary` overrides which browser it launches, only needed if Chrome/Chromium/Brave/Edge isn't in a standard location.
