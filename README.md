# shuck 🦪

**Get the documents out of a JavaScript shell.**

Some document registers only show you their files after you click a tab, expand an accordion, or wait for an AJAX call. `curl` sees an empty page. `wget` sees an empty page. Even `chrome --dump-dom` sees an empty page, because nothing has been clicked yet.

`shuck` renders the page in headless Chrome, **clicks open every tab, accordion and `<details>` element it can find**, and then tells you what's behind them — or downloads it.

```console
$ shuck planningportal.nsw.gov.au/major-projects/projects/project-mars-data-centre
  loading https://www.planningportal.nsw.gov.au/major-projects/projects/project-mars-data-centre
  expand 1: opened 27
  expand 2: opened 0
EIS - March 2026                          https://majorprojects.planningportal…AttachRef=SSD-82052708…
Appendix T - Air Quality Impact Assessment  https://majorprojects.planningportal…
Appendix U - Noise Impact Assessment       https://majorprojects.planningportal…
Appendix V - Ground and Water Conditions   https://majorprojects.planningportal…
…
shuck: 67 link(s)
```

Then take them:

```console
$ shuck <url> --match noise --get ./docs
  ✓ Appendix U - Noise Impact Assessment.pdf  (20.4 MB)
shuck: downloaded 1 file(s) to ./docs
```

## Why

Written while trying to get a noise assessment out of the NSW Planning Portal, which serves its 48-document EIS list only after an accordion click, hangs the fetch off the click handler (so setting `details.open` gets you an empty box), and puts the document's real name in a *sibling* cell to the download link. `shuck` handles all three, and the same three tricks cover most government document registers.

## Install

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

By default `shuck` prints only links that look like documents — a file extension, or a URL containing `download`, `attachment`, `getContent` and friends. `--all` turns that filter off.

Names come from the anchor text, unless the anchor says something useless like "View" or "Download", in which case `shuck` walks up the DOM until it finds a label with actual words in it.

## Examples

```sh
# what's on this page?
shuck example.gov/register --all

# just the acoustics, downloaded
shuck example.gov/register --match acoustic --get ./acoustics

# something needs clicking first
shuck example.gov/case/123 --click "Documents" --click "Show all" --get ./out

# keep the rendered HTML for later parsing
shuck example.gov/register --dom page.html
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
- Respect the terms of service of whatever you point it at. This is for getting at documents that are already public, faster.

## Licence

MIT © Jonathan O'Brien
