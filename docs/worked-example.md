# Worked example: a planning document register

A concrete case, written up because the three obstacles in it are the same three that show up on most government document registers — and because each one defeats a different tool.

The target was a major-projects planning register. A development application's page lists its supporting documents: the environmental impact statement and several dozen technical appendices. The documents themselves are ordinary public PDFs on a plain URL. Getting the *list* of them was the problem.

## What didn't work

| Attempt | Result |
|---|---|
| `curl` the project page | 3 links — the public submissions, nothing else |
| Fetch-and-summarise tool | Reported "48 attachments exist" but no URLs |
| Guessing the JSON API (6 endpoint shapes) | 404 |
| Guessing the CMS node API | 404 |
| `chrome --headless --dump-dom` | Same 3 links as `curl` |

That last row is the interesting one. `--dump-dom` genuinely runs the page's JavaScript, so it is the standard advice for a JS-rendered page — and it still failed, because rendering isn't the obstacle. **Nothing had been clicked.**

## The three obstacles

**1. The list is inside collapsed `<details>` elements.**

```html
<details class="accordion">
  <summary id="accordion-single-10"><p>EIS (48)</p></summary>
  <div class="accordion__content">
    <div class="loader-container"><div id="loader"></div></div>
    <div id="ajax-content10"></div>          <!-- empty until opened -->
  </div>
</details>
```

The count is right there in the DOM — `EIS (48)` — which is why a summarising tool can confidently tell you 48 documents exist while having no idea what they are.

**2. Setting `details.open` opens an empty box.**

The obvious fix is to force every accordion open:

```js
document.querySelectorAll('details').forEach(d => d.open = true);
```

This does nothing useful. The content is lazy-loaded by a handler bound to the **click**, not to the open state. Set the property and you get an expanded, permanently empty div with a spinner in it. You have to click the `<summary>`:

```js
document.querySelectorAll('details:not([open]) > summary').forEach(s => s.click());
```

`shuck` also loops this until a round opens nothing new, because opening one section frequently lazy-loads another inside it.

**3. The document's name isn't in the link.**

Once the content loads, each row looks like this:

```html
<div class="row row--small">
  <div class="row__fill">
    <div class="row__results__category"> Appendix U - Noise Impact Assessment </div>
  </div>
  <div class="row__fill">
    <a href="https://…/getContent?AttachRef=SSD-…%2120260218T235840.826+GMT"> View </a>
  </div>
</div>
```

Every anchor says `View`. The name is in a *sibling* cell, and the href is an opaque attachment reference with no filename in it. Scrape hrefs alone and you get 67 indistinguishable URLs.

`shuck` walks up from the anchor until it finds an ancestor with text of its own once nested anchors are stripped out — which lands on `div.row` and yields the real name. Naïvely taking `a.closest('div')` doesn't work: that's the anchor's own wrapper cell, which is empty once you remove the anchor.

## The result

```console
$ shuck <project-url>
  expand 1: opened 27
  expand 2: opened 0
Appendix S - BDAR                            https://…AttachRef=…235840.285+GMT
Appendix T - Air Quality Impact Assessment   https://…AttachRef=…235839.570+GMT
Appendix U - Noise Impact Assessment         https://…AttachRef=…235840.826+GMT
Appendix V - Ground and Water Conditions     https://…AttachRef=…235838.012+GMT
…
shuck: 67 link(s)
```

And to take one:

```console
$ shuck <project-url> --match "noise" --get ./docs
  ✓ Appendix U - Noise Impact Assessment.pdf  (20.4 MB)
```

The filename comes from the row label, and the extension from the `Content-Type` — which matters here because the URL carries no filename at all.

## The general lesson

Before concluding that a public document is unreachable, work out which kind of obstacle you're facing:

- **Authentication** — a login, a paywall, a per-user entitlement. Genuinely needs a person, or a credential you've been given.
- **Bot defence** — Cloudflare, a CAPTCHA, IP blocking. Sometimes needs a person, often needs an archive mirror instead.
- **Interaction** — the content is public and unprotected, but a click has to happen first.

The third looks exactly like the first two from the outside: an empty response, no error, no explanation. It is the only one of the three that is purely a tooling gap, and it's the one `shuck` exists for.
