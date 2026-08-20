---
name: legacy_site_to_admin_cms
description: Migrate a static HTML site into a server-rendered Node.js app, then build it out into a full admin CMS dashboard — structured catalog CRUD, a generic marketing-page content editor, a zero-dependency page cache, and safe dead-code cleanup. The proven pattern and gotcha catalog from actually shipping one end-to-end (Bongshai Housing), so the next one skips straight to a working build.
---

# Static HTML → Node.js → full admin CMS dashboard

## Goal

Take a site that's 100% static HTML (or already partially templated with no
database) and carry it all the way to a real admin-editable CMS — without a
framework rewrite, without new infra dependencies most shared hosts can't run, and
without breaking SEO/URLs at any step. This is the concrete pattern (and the bugs
that actually happened) from doing exactly this once already, end to end. Follow the
phases in order — each is independently shippable, and each has its own verification
before starting the next. Don't skip to the CMS phases because they're the exciting
part; the conversion phase is what everything else depends on being correct.

## Prerequisites / stack assumptions

Written against Express + a template engine (Nunjucks/EJS/Pug) + Knex-style query
builder + MySQL, deployed to shared/cPanel-style hosting reached only via FTP (no
SSH). The *pattern* (extract → templatize → structured CRUD → generic content
registry → cache → cleanup) is stack-agnostic; adapt the specific syntax (e.g.
Nunjucks's `or` operator, Knex's `.where()`) to whatever the target stack actually
uses. Before starting, confirm with the user: what's the deploy mechanism (FTP+restart
file? SSH? CI/CD?), does the host allow new npm/pip/etc. dependencies safely, is
there already a DB or does one need provisioning, and — critical — make sure work
happens on a branch that does NOT auto-deploy to production if one exists, so
half-built server code can't ship live mid-conversion.

## Phase 0 — Static HTML → server-rendered Node.js

Goal here is **identical output, real backend, zero content/URL change** — the
lowest-risk possible first step, since nothing a search crawler or visitor sees
should change yet.

1. **Audit what's actually live before converting anything.** Static sites
   accumulate dead weight: old server-side scripts (PHP form handlers, etc.) with
   zero references from any current page, orphaned generated files (old PDF/export
   output sitting in a public folder), partial-include files that look like shared
   partials but were never actually wired up (`_header.html` that's copy-pasted
   everywhere instead of included is a real, common pattern — grep for actual
   references, don't assume a file with that name is doing its apparent job). Drop
   the confirmed-dead stuff rather than migrating it.
2. **Extract, don't hand-write, one page at a time.** Write a script (jsdom or
   equivalent DOM parser) that walks every static HTML file, pulls out the
   `<head>` metadata (title, description, OG/Twitter tags, canonical) and the main
   content body, and emits: a template file per page (or per page-family, if many
   pages share near-identical structure) + one central registry (JSON) mapping
   URL → template + metadata. The registry is what makes "add a converted page"
   mean "run the script again," not "hand-write a route." Watch for genuine
   one-off pages (usually the homepage) with enough unique head content that
   forcing them through the generic template wastes more effort fighting the
   abstraction than just giving that one page its own override block.
3. **Build real shared layout/partials for the first time** — nav, footer,
   meta boilerplate that was copy-pasted into every static file becomes one
   `{% extends %}` chain. This is usually the actual pain point driving the whole
   migration (a sitewide CSS/JS version bump or nav change previously meant
   scripting a find-and-replace across hundreds of files); confirm it's genuinely
   fixed by checking a sitewide change now touches one file, not many.
4. **Audit and migrate server-side form/script handlers.** Same rule as step 1:
   check what's actually wired to a live form/button before porting it — don't
   assume every script file in the old backend is used. What's real usually needs:
   an Express route, an email-sending library (check if the old host's mail()-style
   free ride exists on the new setup too, or if SMTP credentials need provisioning
   explicitly), file upload handling if any form takes attachments, and — if the
   old flow generated PDFs — a lightweight PDF library over a headless-browser one
   unless there's a real pixel-fidelity requirement, since headless-Chromium's
   memory footprint is a common OOM trigger on memory-capped shared hosting.
5. **Port whatever the old web server did implicitly.** A static site's
   `.htaccess`-equivalent usually does more than URL rewrites: security headers,
   compression, cache lifetimes, a real (non-soft) 404. All of it needs an explicit
   equivalent once a request stops being served directly by Apache/Nginx and starts
   hitting the Node process — a security-headers middleware library, a compression
   middleware, explicit `Cache-Control` on static asset routes. Don't assume the old
   web server is still "in front" unless it genuinely still serves those exact
   paths.
6. **Verify by diffing content, not by trusting the extraction script.**
   Whitespace/comment-normalized diff of every converted page's rendered output
   against the original static file, before trusting the script at scale — spot-
   checking a handful of "representative" pages is not enough; a real conversion
   found systemic gaps (wrong OG image dimensions on most product pages, page-
   specific `<script>` blocks silently dropped on most pages, a UI toggle missing
   on most pages) that only a full diff caught. Also watch for: a template-engine
   gotcha where `{% block %}` overrides defined inside an `{% include %}`'d
   partial silently do nothing — blocks only resolve through an actual
   `{% extends %}` chain, not across an include boundary; and "which variant wins
   when pages split roughly down the middle" — don't default to majority behavior
   when the split isn't lopsided, check the actual count and make it conditional
   instead.
7. **Set up the real deploy pipeline before going further**, including a staging
   environment separate from production if at all possible. Solve however this
   host actually restarts a running Node process after a deploy (a magic
   restart-marker file some Node-hosting setups watch for is common on shared
   cPanel-style hosts with no SSH) — this is a separate question from "does the
   host support Node at all," confirm it explicitly rather than assuming.
8. **Verify end-to-end**: every original URL still resolves to equivalent content
   (redirect-diff script, re-run after every later phase too, not just once at
   cutover); any migrated forms send correctly end-to-end including attachments;
   security headers/compression/cache lifetimes present on responses; keep the old
   static file tree deployable in parallel for a burn-in period rather than
   deleting it immediately — a tested rollback path beats a theoretical one.

## Phase 1 — Audit before building the CMS

1. Re-map every page/route now that Phase 0's registry exists. Which are already
   DB-backed vs. still template-only with no database behind them? Don't assume —
   grep the actual route definitions, don't trust old docs/comments (a stale "not
   started" note can be wrong by months; verify against current code every time).
2. Check for dead code from a half-finished later migration too: does a newer
   DB-driven route shadow older per-page templates that are no longer reachable but
   still sitting in the repo? (Real example hit: 141 old per-product template files,
   superseded by one DB-driven route mounted earlier in the middleware chain,
   silently dead for weeks.)
3. Check for an existing admin panel/CRUD before building a new one — don't
   duplicate.

## Phase 2 — Structured catalog CRUD

For content that's genuinely tabular (products, categories, projects/case-studies,
team members, testimonials, FAQs, service areas/locations, leads/inquiries) — build
real DB tables + admin list/create/edit/delete pages. This is the "obvious" 80% of a
CMS and most AI coding tools get this part right without much guidance. The checklist
below is what a **complete** version of this looks like (missing pieces are the most
common way a "dashboard" ships feeling incomplete):

**Dashboard build checklist:**
- [ ] Every content type has: list view (search/filter/sort), create form, edit form,
      delete (with a guard against deleting something still referenced elsewhere,
      e.g. a category with products still in it), publish/unpublish toggle
- [ ] Image/media upload (not just a URL text field) with automatic resize/responsive
      variants, and a shared media library view if the site has enough images to
      warrant one
- [ ] SEO fields per content type where relevant (meta title, meta description, alt
      text) — and if any AI-assisted generation is involved, route it through a
      **review queue** (draft → pending → approved/rejected), never auto-write to a
      live page. This project had a real incident with bad auto-generated data before
      adopting that rule — treat it as load-bearing, not optional caution.
- [ ] CSRF protection on every state-changing admin route, and a consistent pattern
      for how the token reaches the form (check for an existing sitewide convention —
      e.g. a layout-level script that reads a `<meta>` tag and auto-injects a hidden
      field into every form — before inventing a new one per feature)
- [ ] Role/permission gating if more than one admin user type exists (admin vs.
      editor vs. superadmin) — check per-route, not just per-section
- [ ] Activity log for who-changed-what, at minimum for destructive actions (delete)
- [ ] Bulk actions on list views once there's more than ~10 rows of anything (bulk
      publish/approve/delete) — a real feature gap, not a nice-to-have, once content
      volume grows past hand-clicking-each-row
- [ ] Site-wide settings section (contact info, social links, theme/color tokens,
      announcement banner) if the site has any of these hardcoded in templates today
- [ ] Analytics/dashboard home summarizing key counts (leads this week, published
      products, pending reviews) — the first thing an admin sees on login

## Phase 3 — Generic marketing-page content editor

This is the part that's usually missing even after Phase 2 ships, and it's the actual
point of the whole exercise: making it possible to edit prose/marketing copy
(About Us, location/landing pages, homepage hero, etc.) without a code deploy. A
one-column-per-field DB schema doesn't scale (one page can easily have 25-35 editable
regions); a single raw-HTML-blob-per-page is unsafe (breaks layout structure, no
guardrails, and rendering it with a "mark as safe/don't escape" filter is a bigger
blast radius than scoped fields). The right shape:

1. **One table**, e.g. `page_content(url_path PRIMARY KEY, title, content_json, updated_at)`
   — a JSON blob per page holding named fields, not one column per field.
2. **A hand-authored field registry** (one file, keyed by page path/slug, each entry
   an ordered list of `{ key, label, type }`) — don't try to auto-introspect fields
   from the template's HTML structure; too fragile against future template edits.
3. **Template integration via fallback, not replacement**: every field renders as
   `{{ pc.field_name or "<original hardcoded copy>" }}` (or the template language's
   equivalent optional-chaining/default pattern) — so a page with no DB row yet, or a
   field not yet in the registry, keeps rendering exactly what it renders today. This
   makes rollout zero-risk per page: migrate one page, verify, then the next — never
   all at once.
4. **Extract, don't retype**: reuse Phase 0's DOM-extraction approach — a one-off
   script that pulls each registered field's current text out of the live template by
   structural position, and seeds the initial DB row from it. Nothing gets hand-typed
   from scratch, so nothing gets lost or subtly altered in transcription.
5. **Admin UI**: one list page + one dynamic edit form (rendered from the registry —
   a `<textarea>`/`<input>` per field, in registry order), modeled on whatever list/
   form pattern Phase 2's CRUD already established. Don't invent a second UI pattern.
6. Route the DB fetch through a **try/catch that degrades to `{}`** on any failure
   (table doesn't exist yet, DB offline, whatever) — the public page must never 500 or
   go blank because of this feature; it should silently fall back to the hardcoded
   template text.

## Phase 4 — Zero-dependency page cache

Once pages are DB-backed, every request hits the database — fine at low traffic, a
real problem on shared/latency-constrained hosting. Before reaching for Redis or an
npm cache package, check: does this host actually support new native-binary/infra
dependencies safely? (Shared cPanel hosting commonly has tight memory caps that
silently OOM-kill `npm install` of anything non-trivial — confirm before assuming.)
If constrained, a hand-rolled in-memory cache costs zero new dependencies:

- A plain `Map<url, { html, cachedAt }>`. Hand-roll LRU using the Map's
  insertion-order guarantee (delete+re-set on a hit moves an entry to the end; evict
  from the front once past a cap). TTL checked lazily on read, not a timer sweep.
- Wire it in via the template engine's own "give me the rendered string instead of
  auto-sending" mechanism if one exists (e.g. Express + most engines support
  `res.render(view, vars, callback)` — the callback form suppresses auto-send and
  hands back the string) rather than monkey-patching the response object.
- **Invalidation: full flush on any admin write**, not per-URL mapping. Precisely
  tracking "this edit affects exactly these cached pages" is a real correctness trap
  (editing one product can affect a category page's card AND other products' related
  rails) — a full flush is simpler, safer, and costs nothing meaningful given how
  infrequent admin edits are relative to visitor traffic. Add a manual "clear cache"
  button in the admin UI too, as a cheap escape hatch.

## Phase 5 — Safe dead-code cleanup

Only after Phases 0-4 are stable: remove whatever old static templates/routes are now
genuinely superseded.

1. **Add a safety net before deleting anything**: wrap the old generic
   static-rendering fallback in a try/catch that degrades to a clean 404 on failure,
   not a raw 500. This protects against exactly the scenario cleanup creates: a
   content row gets deleted from the DB later, its route falls through to the old
   static path, which no longer exists.
2. **Verify against the live DB**, not a seed/fixture file, that every static
   template slated for deletion truly has no route depending on it as a fallback.
3. Delete the confirmed-dead files. Leave any page-registry/route-map metadata
   entries alone unless something else genuinely reads them — check first (sitemap
   generators, search-index builders, and admin preview dropdowns are the usual
   suspects for still referencing old entries even after the route itself moved on).
4. Verify: a real content slug still renders, a deliberately fake one now 404s
   cleanly instead of 500ing.

## Gotcha catalog

Real bugs hit doing this, in the order they're likely to bite:

### `SyntaxError: Identifier 'X' has already been declared`
Happens when a new `require`/`import` of the DB client gets added to a file's top
during cache/feature wiring, duplicating an existing conditional
`let db; try { db = require(...) } catch { db = null }` pattern lower in the same
file. This is a hard crash on process boot (or `require()` of the file), not a
runtime error — every restart fails until fixed. **Fix**: search the file for an
existing DB-import pattern before adding a new one; if a "DB is optional, degrade
gracefully" pattern already exists, reuse it rather than adding a second import.
Cheap prevention: `node -c <file>` (or equivalent syntax-only check for the language)
on every touched file before every deploy — this exact bug slipped through twice in
one session before making that check mandatory.

### Template `safe`/"mark as trusted" filter combined with a fallback operator, applied on the wrong side
`{{ value | safe or "fallback" | safe }}` (Nunjucks/Jinja-family syntax) does **not**
do what it looks like it does. The filter binds tighter than `or`, so this evaluates
as `(value | safe) or (fallback | safe)` — and wrapping `undefined`/`null` in a
"trusted string" filter typically produces an always-truthy wrapper object, so the
`or` never fires even when `value` is genuinely empty. Net effect: the field renders
**blank**, not the fallback — silent content loss, not an error. **Fix**: group the
fallback first, apply the trust/safe filter once to the result:
`{{ (value or "fallback") | safe }}`. Verify by actually rendering the template with
an empty/undefined value and checking the output string, not just reading the source
and assuming it's correct — this is exactly the kind of bug that reads fine at a
glance.

### A cron/cleanup job's process-matching pattern collides across environments
If staging and production apps share a naming convention where one name is a string-
prefix of the other (e.g. `myapp` and `myapp-prod`), a process-matching pattern
(`pgrep -f`, `ps aux | grep`, etc.) written for the shorter name will **also** match
the longer one as a substring hit. A "keep only the newest process, kill the rest"
cleanup script run from either environment then pools both apps' processes together
and can kill the *other* environment's only live worker. Symptom looks like random,
inconsistent staleness — one route serves fresh content, another serves stale content
from the same deploy, because two different worker processes are alive and requests
land on whichever one didn't get reaped. **Fix**: derive the match pattern from the
script's own actual deployed location (not a hardcoded shared string), and require the
match be followed by a path separator or end-of-string, not another name character —
reject a bare prefix match.

### AJAX-ifying an existing form-POST admin route
Converting a route that always did a redirect-on-success into one that also serves
JSON for a new inline/AJAX caller: the route needs to detect which caller it is
(`Accept` header, a query flag, whatever) and branch — don't just change its response
type globally, or the original full-page form flow breaks. Also: reuse the *exact*
existing write/validation logic for the new bulk/inline caller rather than
reimplementing it — a second, slightly-different copy of "how to create a category"
or "how to approve a suggestion" is how the two paths quietly drift.

### Blind "fix everything" bulk-action requests
When asked for a one-click "auto-fix all SEO/content issues" type feature: check
whether the issues in question actually need human judgment (thin content, broken
images, anything requiring real writing) before building an auto-apply button for
them. If the project already routes AI-suggested changes through a review queue
(see Phase 2's checklist item on this), a blind bulk-auto-apply directly undermines
that safeguard. The safe version of "one click, many fixes" is **bulk-approve on the
existing review queue** (a human still picks the batch, just not one row at a time) —
not bulk-generate-and-auto-apply.

## Deploy checklist (per phase, every time)

- [ ] Syntax-check every touched file (`node -c` or language equivalent) — cheap,
      catches the #1 recurring failure above
- [ ] Run whatever sitewide structural/render check already exists in the repo (e.g.
      "render every page template, count that footer/body/html tags open-close
      correctly"), or build one during Phase 0 if none exists yet — don't skip
      because "it's just a small change"
- [ ] Deploy to staging first if one exists; gate production on staging's smoke test
      passing, don't push both blind
- [ ] After deploy, verify the *specific* things this phase changed — a generic
      "homepage returns 200" smoke test does not catch a blank hero title or a broken
      fallback. Check the actual rendered content of what changed, not just the
      status code.
- [ ] If a DB migration/table-creation script is part of the phase and there's no way
      to run it remotely (no SSH), say so explicitly and hand over the exact
      copy-paste command rather than assuming it happened — confirm the routes that
      depend on the new table degrade gracefully (try/catch → empty fallback) until
      it's actually run.
