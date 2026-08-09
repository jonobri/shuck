# shuck 🦪

**Get the documents out of a JavaScript shell.**

Plenty of document registers only reveal their files after you click a tab, expand an accordion, or wait for an AJAX call. `curl` sees an empty page. `wget` sees an empty page. Even `chrome --dump-dom` sees an empty page, because nothing has been clicked yet.

`shuck` renders the page in headless Chrome, **clicks open every tab, accordion and `<details>` element it can find**, and then tells you what's behind them — or downloads it.

```console
$ shuck example.org/records/case/1234
  loading https://example.org/records/case/1234
  expand 1: opened 12
  expand 2: opened 3
  expand 3: opened 0
Application form                    https://example.org/files/get?ref=a41f…
Statement of environmental effects  https://example.org/files/get?ref=b902…
Appendix C — Traffic assessment     https://example.org/files/get?ref=c115…
Appendix D — Acoustic assessment    https://example.org/files/get?ref=d773…
…
shuck: 38 link(s)
```

Then take them:

```console
$ shuck example.org/records/case/1234 --match acoustic --get ./docs
  ✓ Appendix D — Acoustic assessment.pdf  (4.2 MB)
shuck: downloaded 1 file(s) to ./docs
```

## Why

Three things routinely defeat a plain fetch, and a register usually does all three at once:

1. The file list lives inside collapsed `<details>` or tab panels, so it isn't in the initial HTML.
2. Setting `details.open = true` opens an **empty** box, because the fetch hangs off the *click* handler rather than the open state.
3. The link's anchor text is just "View" or "Download", with the document's real name in a neighbouring cell.

`shuck` handles all three. There's a [worked example](docs/worked-example.md) if you want to see what that looks like against a real register.

## Install

```sh
brew install jonobri/tap/shuck
```

Or from source:

```sh
git clone https://github.com/jonobri/shuck
ln -s "$PWD/shuck/shuck" /usr/local/bin/shuck   # or anywhere on your PATH
```

Needs Python 3.8+ and Chrome, Chromium, Brave or Edge. **No pip install** — the WebSocket client is stdlib sockets, so it keeps working in sandboxes and locked-down environments where you can't add dependencies.

If your browser lives somewhere unusual, set `SHUCK_CHROME=/path/to/binary`.

## Usage

```
shuck <url> [options]

  --match STR       keep only links whose name or URL contains STR
  --all             list every link, not just document-looking ones
  --get DIR         download the matched links into DIR
  --click SEL|TEXT  click a CSS selector, or an element containing this text
                    (repeatable — happens before the auto-expand)
  --no-expand       don't auto-open tabs, accordions and <details>
  --dom FILE        save the rendered DOM to FILE
  --json            emit results as JSON
  --wait SEC        settle time after load and each interaction (default 3)
  -q, --quiet       suppress progress on stderr
```

By default `shuck` prints only links that look like documents — a file extension, or a URL containing `download`, `attachment` and friends. `--all` turns that filter off.

Names come from the anchor text, unless the anchor says something useless like "View" or "Download", in which case `shuck` walks up the DOM until it finds a label with actual words in it.

## Examples

```sh
# what's on this page?
shuck example.org/register --all

# just the acoustics, downloaded
shuck example.org/register --match acoustic --get ./acoustics

# something needs clicking first
shuck example.org/case/123 --click "Documents" --click "Show all" --get ./out

# keep the rendered HTML for later parsing
shuck example.org/register --dom page.html

# machine-readable
shuck example.org/register --json | jq -r '.[].href'
```

## What it does, in order

1. Launches headless Chrome on a free port with a throwaway profile.
2. Drives it over the DevTools Protocol (raw WebSocket, no library).
3. Loads the page and waits for it to settle.
4. Runs your `--click`s, in order.
5. Repeatedly clicks every `<summary>`, `[aria-expanded=false]` and unselected tab until a round opens nothing new — because opening one section often lazy-loads another inside it.
6. Reads the DOM and every `<a href>`, resolving each one's human-readable name.
7. Filters, prints, or downloads.

Then it kills Chrome and deletes the profile.

## Caveats

- **It clicks things.** On a page with destructive controls, that's a bad idea — scope it with `--no-expand --click` instead.
- It won't get past a login, a paywall, or a CAPTCHA, and doesn't try to.
- It's polite by default but has no rate limiting; don't point it at someone's server in a loop.
- Respect the terms of service of whatever you point it at. This is for reaching documents that are already public, faster.

## Licence

MIT © Jonathan O'Brien
