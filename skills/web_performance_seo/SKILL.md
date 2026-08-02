---
name: web_performance_seo
description: Guidelines and step-by-step workflows for optimizing Core Web Vitals, CSS/JS minification, non-blocking font loading, cache-busting, DNS prefetch, and local SEO (Dhaka/Bangladesh) for static HTML sites.
---

# Web Performance & SEO Skill

## Principles
1. **Structured Data**: Include accurate Schema.org JSON-LD on every page.
2. **Local SEO Meta Tags**: Standardize geo.region (BD-C), geo.placename (Dhaka, Bangladesh).
3. **Page Load Speed**: Use `loading="lazy"` on all images. Specify explicit `width` + `height`.

## CSS & JS Minification Workflow

### Install tools (one-time)
```bash
npm install -g clean-css-cli terser
```

### Minify CSS
```bash
cleancss -o css/style.min.css css/style.css
# Verify: check for WARNING about skipped @import lines
```

### Minify JS
```bash
terser js/global-upgrades.js --compress --mangle --output js/global-upgrades.min.js
```

### Update all HTML files to use minified assets + cache-bust version
```python
import glob, re
for f in glob.glob('*.html'):
    c = open(f, encoding='utf-8').read()
    c = re.sub(r'css/style\.css(\?v=[\d.]+)?', 'css/style.min.css?v=X.X', c)
    c = re.sub(r'js/global-upgrades\.js(\?v=[\d.]+)?', 'js/global-upgrades.min.js?v=X.X', c)
    open(f, 'w', encoding='utf-8').write(c)
```
> **CRITICAL**: Bump X.X version number each time to bust Cloudflare/browser cache.

---

## Non-Blocking Google Fonts

### ❌ WRONG (render-blocking — never do this in CSS):
```css
@import url('https://fonts.googleapis.com/css2?family=Inter...');
```

### ✅ CORRECT (non-blocking preload in HTML `<head>`):
```html
<link rel="preconnect" href="https://fonts.googleapis.com" crossorigin>
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="preload" as="style" href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap"
      onload="this.onload=null;this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap"></noscript>
```

---

## Resource Hints (add to all HTML `<head>`)
```html
<link rel="dns-prefetch" href="https://unpkg.com">
<link rel="dns-prefetch" href="https://fonts.googleapis.com">
<link rel="dns-prefetch" href="https://fonts.gstatic.com">
<link rel="preload" href="css/style.min.css?v=X.X" as="style">
```

---

## Ongoing Rebuild Workflow
After ANY change to `style.css` or `global-upgrades.js`:
1. Re-minify CSS: `cleancss -o css/style.min.css css/style.css`
2. Re-minify JS: `terser js/global-upgrades.js --compress --mangle -o js/global-upgrades.min.js`
3. Bump version: update `?v=X.X` across all HTML files
4. Commit: `git add . && git commit -m "Rebuild minified assets vX.X" && git push`

---

## Performance Benchmarks Achieved (bongshaihousing.com)
| Asset | Before | After | Savings |
|---|---|---|---|
| style.css | 100.7 KB | 67.9 KB | -32.8 KB |
| global-upgrades.js | 48.7 KB | 23.8 KB | -24.9 KB |
| Google Fonts | Render-blocking | Non-blocking | Faster FCP |
| **Total** | **149.4 KB** | **91.7 KB** | **-57.7 KB** |

---

## Service Worker Cache Invalidation Discipline

**Goal**: Make sure every CSS/JS fix actually reaches *returning* visitors, not just
first-time visitors or a hard-refreshed dev browser.

**Proof**: On bongshaihousing.com, a real bfcache back-button fix (`page-transition.js`)
and a real duplicate-scrollbar CSS fix were both deployed to the live server and *still*
didn't reach the user — purely because `sw.js`'s `CACHE_NAME` hadn't been bumped. Its
cache-first fetch strategy for `.css`/`.js` kept serving the stale, previously-cached file
to anyone who'd visited before. The user reported "still going back needs refresh" and
"still same" on two separate, correctly-written fixes before this was root-caused.

**Steps**:
1. Any time a file listed in `STATIC_ASSETS`, or any `.css`/`.js` referenced with a
   `?v=` query string, changes — bump that `?v=X.X` on every HTML page that references it.
2. Bump `CACHE_NAME` in `sw.js` (`bongshai-cache-vN` → `vN+1`) in the **same commit**.
   That's what triggers the `activate` handler's
   `caches.keys().then(...).filter(name => name !== CACHE_NAME).map(name => caches.delete(name))`
   purge of every old cache.
3. Update the matching entries inside `STATIC_ASSETS` to the new `?v=` value too, so the
   fresh service worker precaches the new version on install.
4. Verify before committing: a sitewide grep for the OLD version string
   (`style.min.css?v=<OLD>`) must return zero files.
5. Tell the user a hard refresh (Ctrl+Shift+R) or close/reopen the tab may still be
   needed once — their currently-installed service worker won't self-update until its
   own next fetch of `sw.js` detects a byte difference.

---

## Responsive Image (srcset) Retrofit at Scale — No Build System

**Goal**: Add real `srcset`/`sizes` responsive images across hundreds of hand-written
static HTML pages without a build system, without corrupting the other 95% of each file
that isn't being touched.

**Proof**: Applied across 219 static HTML pages / 3,936 `<img>` tags on
bongshaihousing.com in one pass. Zero regressions, verified by re-running the same
JSON-LD-parses-as-JSON and broken-internal-link checks used earlier in the session as a
byte-fidelity smoke test, rather than trusting a full HTML reserialize to be safe.

**Steps**:
1. Check whether an image library is already a project dependency before adding one
   (`node -e "require.resolve('sharp')"` or check `package.json`) — don't add a new
   dependency just for a one-off script.
2. Generate 2-3 resized WebP variants per source image with `sharp`, named predictably
   (`name-400w.webp`, `name-700w.webp`). Never upscale — skip any tier wider than the
   image's own natural width, and skip images already smaller than the smallest tier.
3. **Do not** parse-and-reserialize the whole HTML document with a DOM library
   (jsdom/cheerio) just to inject two attributes. Full reserialization risks silently
   changing quote style, self-closing syntax, or whitespace across every *other*
   untouched tag in the file — a huge, unauditable diff for a two-attribute change.
4. Instead, regex-match only the `<img ...>` tag substrings and splice
   `srcset="..." sizes="..."` in as new text immediately after the existing `src="..."`
   attribute. Every other byte of the file stays untouched.
5. Pick `sizes` from a small number of known layout contexts (e.g. detect a
   `class="hero-bg"` wrapper immediately preceding the `<img>` → `sizes="100vw"`;
   everything else gets one shared grid/detail default). `sizes` only affects which file
   the browser *fetches* — it never changes the rendered box size (that's still CSS) —
   so an imperfect guess cannot break layout. That's what makes this safe to apply
   broadly without hand-tuning every template.
6. Re-run existing content validators (JSON-LD parse check, broken-link check) afterward
   as a regression check, even though they weren't written for images — they still catch
   "did I accidentally corrupt a file."

