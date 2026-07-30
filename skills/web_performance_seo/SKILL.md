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

