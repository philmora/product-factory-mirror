---
title: "Brand-correct OpenGraph cards for a Framer + GitHub-mirror site"
slug: og-card-generation
dek: "Generate per-surface OG cards from a single HTML template, host on raw.githubusercontent.com, bind via Framer CMS Variables. Static-HTML output that bot scrapers can read without JS."
public: true
tags: [framer, opengraph, seo, puppeteer, github-mirror, design-system]
applies-to: [framer-sites, github-mirror-content-sites, terminal-aurora-design-system]
created: 2026-05-08
updated: 2026-05-08
status: validated
order: 2
category: recipes
---

# Brand-correct OpenGraph cards for a Framer + GitHub-mirror site

Generate per-surface OG cards from a single HTML template, host on raw.githubusercontent.com, bind via Framer CMS Variables. Static-HTML output that bot scrapers can read without JS.

## When to apply

- Framer-built site with multiple URL surfaces (static pages + CMS-driven detail pages)
- URL previews are landing blank or sparse on LinkedIn, Slack, iMessage, X, Discord
- Site content lives in a public GitHub repo (essays.json, wiki.json, recipes.json) that the Framer canvas fetches at runtime
- You want **one visual identity** across all preview cards, with per-page title/dek/eyebrow rendered into the card itself

The bot-scraper constraint that drives this whole approach: LinkedIn, Slack, iMessage, X scrape **static HTML only** — no JS execution. So the meta tags must be in Framer's static-published HTML, not injected by a code component. Code-component approaches won't work.

## Steps

### 1. Build the HTML template

Single `template.html`, 1200×630 viewport, with placeholders the generator fills in:

```html
<body class="{{CARD_CLASS}}">
  <div class="card">
    <div class="eyebrow">{{EYEBROW}}</div>
    <div class="body-block">
      <h1 class="title">{{TITLE}}</h1>
      <p class="dek">{{DEK}}</p>
    </div>
    <div class="footer">
      <span class="brand"><span class="sigil"></span>BRAND</span>
      <span class="url">DOMAIN.COM<span class="signal">{{ROUTE_SUFFIX}}</span></span>
    </div>
  </div>
</body>
```

Use **two card classes**: `body.site` (large brand identity for static pages) and `body.content` (eyebrow + title + dek + footer for detail pages). Auto-size title font based on length (`title`, `title-md`, `title-sm`) so most titles fit at 4 lines.

Honor your design system in CSS — color tokens, type scale, optical sizing variation-settings. Don't reinvent. This card IS the brand surface.

### 2. Build the generator

Python script that reads your public mirrors and fills the template:

```python
def make(name, card_class, eyebrow, title, dek, route_suffix):
    html_doc = TEMPLATE.replace("{{CARD_CLASS}}", card_class)\
                       .replace("{{TITLE}}", title)\
                       .replace("{{DEK}}", html.escape(dek))\
                       ...
    (HTML_DIR / f"_{name.replace('/', '_')}.html").write_text(html_doc)

# Hardcode the static-page identity copy (it's site-identity, not data)
make("site-home", "site", eyebrow_html, headline_html, dek_text, "")

# Loop the public-mirror JSON for detail pages
essays = fetch_json("https://raw.githubusercontent.com/<org>/essays/main/essays.json")
for e in essays["essays"]:
    make(f"essays/{e['slug']}", "content", eyebrow, e["title"], e["dek"], f"/ESSAYS/{e['slug'].upper()}")
```

Encode subdirectories as `_dir_slug.html` (e.g. `_essays_the-pm-is-dead.html`). The renderer reverses this to write into the right output subdirectory.

### 3. Render PNGs with puppeteer-core

**Critical gotcha:** Chrome's `--screenshot` CLI takes the screenshot before web fonts swap in — even with `&display=block` in the Google Fonts URL. Result: fallback-typeface PNGs.

The fix is `puppeteer.page.evaluate(() => document.fonts.ready)` before `page.screenshot()`. This is the **central reason** to use puppeteer-core instead of headless Chrome CLI:

```js
const browser = await puppeteer.launch({
  executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
  headless: "new",
  defaultViewport: { width: 1200, height: 630 },
});
const page = await browser.newPage();

for (const f of htmlFiles) {
  await page.goto(`file://${f}`, { waitUntil: "networkidle0" });
  await page.evaluate(() => document.fonts.ready);  // ← the missing piece
  await new Promise(r => setTimeout(r, 150));        // tiny settle for variation-settings
  await page.screenshot({ path: outPath, type: "png" });
}
```

Use `puppeteer-core` (not `puppeteer`) so you can point at system Chrome instead of bundling a 200MB Chromium. Add `puppeteer-core: ^22` to `package.json`.

### 4. Host on raw.githubusercontent.com

Commit the rendered PNGs to a path in your site repo:

```
<org>/<site-repo>/og/
├── site-{home,thoughts,...}.png    # static surfaces
├── essays/<slug>.png                # CMS detail pages
└── work-brain/<slug>.png            # CMS detail pages
```

Reference them as `https://raw.githubusercontent.com/<org>/<site-repo>/main/og/<path>.png`. Public, edge-cached, no auth, no asset-pipeline complexity.

Framer auto-rehosts externally-referenced images to its own CDN (`framerusercontent.com`) on first render. Both URL forms work for bot scrapers.

### 5. Bind in Framer

The Framer MCP plugin **does not expose a per-page SEO surface** — it can read/write CMS items, code files, and project XML, but not Page Settings. So the binding is half-MCP, half-UI:

**Static pages (manual UI work):**
1. Open Framer project → click page in sidebar → click `…` → `Settings`
2. Scroll to `Page Images → Social Preview`
3. Drag-drop the rendered PNG into the upload zone, OR paste the raw.githubusercontent.com URL via `Choose Image → Upload`

**CMS-bound `:slug` templates (one-time setup):**
1. Add an `OG Image` field (image type) to the CMS collection — must be done in Framer UI, MCP cannot add fields to user-managed collections
2. Open the `:slug` template → `Settings` → `Page Settings`
3. Title field: `{{Title}} - <site brand suffix>` (CMS Variables work in plain-text fields)
4. Description field: `{{Dek}}`
5. `Social Preview → Choose Image → CMS Variables → OG Image`
6. Save

**Per-CMS-item OG image URLs (automatable via MCP):**

```js
// For each item, call upsertCMSItem with the OG Image field set to:
// https://raw.githubusercontent.com/<org>/<site-repo>/main/og/<dir>/<slug>.png
mcp.upsertCMSItem({
  collectionId: "<id>",
  itemId: "<id>",
  fields: { "<og-image-field-id>": "https://raw.githubusercontent.com/.../og/essays/the-pm-is-dead.png" }
})
```

### 6. Publish in Framer + verify

Click Publish in Framer (one-time after binding setup; Framer caches meta tags). Then verify with:

```bash
curl -sL https://<site>/<surface> | grep -ioE '<meta (property|name)="(og:[a-z]*|twitter:[a-z]*)"[^>]*>'
```

Should return `og:title`, `og:description`, `og:image`, `og:url`, `twitter:card="summary_large_image"`, `twitter:image`, `twitter:title`, `twitter:description`. CMS-bound surfaces should show per-item titles and per-item image URLs, not the site defaults.

Eyeball-check via [LinkedIn Post Inspector](https://www.linkedin.com/post-inspector/) for at least one URL of each surface type.

### 7. Wire into a regen workflow

Once shipped, automate the regeneration:

- New essay → push to `<org>/essays/content/<slug>.md`, run `sync-og-cards.sh`, upsert CMS item with new OG URL via MCP, publish in Framer
- New CMS-detail concept/recipe → same flow
- Site copy edited for a static page → edit hardcoded entries in `generate.py`, run `sync-og-cards.sh`, upload via Framer UI

The script lives in the Product Factory at `~/BaseCamp/factory/scripts/og-cards/`. See its README for operator details.

## Gotchas

1. **Fallback fonts in rendered PNG** → Chrome `--screenshot` CLI doesn't wait for web fonts. Use puppeteer-core with `await page.evaluate(() => document.fonts.ready)`. (See step 3.)
2. **Framer UI required for first-time binding** → MCP has no per-page SEO surface and cannot add CMS fields. Plan ~10 min of one-time manual Framer work; the bindings carry forward to all future content.
3. **Code-component meta tags don't work** → bot scrapers don't execute JS. Only Framer Page Settings → Static HTML output is read. Don't try to inject `<meta>` from a `.tsx` component.
4. **`raw.githubusercontent.com` 30s propagation** → after `git push`, the CDN takes ~30s. Don't immediately retry, just wait.
5. **Existing CMS items keep cached `framerusercontent.com` URLs** → if you regenerate the PNG and push, existing items don't auto-update. To refresh: re-upsert the CMS item via MCP with the same raw.githubusercontent.com URL — Framer re-rehosts.
6. **Title clipping** → if your titles vary widely in length, auto-scale font size in CSS based on a length-bucketed body class set by the generator (`title`, `title-md`, `title-sm`). Don't try to make one font size fit everything.
7. **Framer's React file-upload doesn't react to standard `input.change`** → if driving the UI via Chrome MCP for upload automation, simulate drag-drop with the DataTransfer API + DragEvent sequence (`dragenter`, `dragover`, `drop`) on the parent of the "Drop image" zone. Standard file-upload-via-input is silently ignored.

## Validation

Concrete numbers from the philmora.com rollout (2026-05-08):

- 26 cards rendered (5 static + 8 essays + 12 work-brain concepts + 1 factory recipe)
- All 7 surface types verified emitting complete `og:*` and `twitter:*` meta tags via curl
- CMS-bound dynamic surfaces correctly resolve per-item titles, deks, and per-item OG image URLs
- One-time Framer UI setup: ~10 min (add 2 CMS fields, paste 5 static-page settings, bind 2 dynamic templates)
- Per-page render time: ~2s under puppeteer-core
- Total regeneration time for the full set: ~60s

## Validated on

- `philmora.com` · `2026-05-08` · all 7 surface types live, verified via curl + LinkedIn Post Inspector

## References

- Generator + renderer: `~/BaseCamp/factory/scripts/og-cards/`
- Operator README: `~/BaseCamp/factory/scripts/og-cards/README.md`
- Strategy ADR: `~/BaseCamp/home/decisions/2026-05-08_philmora-og-metadata.md`
- philmora-site backup commit: `philmora/philmora-site` SHA `739773d` (cards) + `616974d` (snapshot doc)
- The puppeteer fonts.ready insight: [GitHub puppeteer/puppeteer issue #422](https://github.com/puppeteer/puppeteer/issues/422)

## Updates

<!-- Append-only. Date each addition. -->

- 2026-05-08 — Created. Validated on philmora.com initial rollout.
