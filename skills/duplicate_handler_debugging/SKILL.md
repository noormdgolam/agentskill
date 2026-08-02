---
name: duplicate_handler_debugging
description: Find and eliminate silently-conflicting duplicate logic (JS event handlers, CSS rule blocks, or mis-scoped media-query sections) before spending time debugging symptoms that are actually two implementations fighting over the same DOM/CSS state.
---

# Duplicate Handler / Mis-scoped Rule Debugging

## Goal
Catch the specific bug class where the same feature was implemented twice — once in a
dedicated file/section and once copy-pasted into a shared "global" file/section, or a
CSS section whose comment header claims a scope (e.g. "mobile only") that its actual
`@media` nesting doesn't match. These bugs are invisible from reading either copy in
isolation; a fix to one copy looks correct in review and still doesn't resolve the
user's report, because the *other* copy is still running.

## Proof
Found and fixed 8 separate instances of this exact pattern in one session on
bongshaihousing.com:
- 3D tilt effect duplicated in `global-upgrades.js` **and** `3d-tilt.js`, both rotating
  the same elements independently on every `mousemove`.
- A whole separate `lightbox.js` file duplicating a lightbox already scoped correctly
  inside `global-upgrades.js`.
- Page-fade-transition duplicated in `global-upgrades.js` **and** `page-transition.js`,
  both intercepting every `<a href>` click on the page.
- Two separate `::-webkit-scrollbar` CSS blocks styling the browser scrollbar with
  different colors — the later one silently won the cascade, so the user kept seeing a
  colored scrollbar after the "first" one was fixed.
- A back-to-top progress-ring widget and a scroll-progress bar the user never asked to
  keep, still wired up from an earlier phase of the project.
- A CSS section literally titled "Mobile Layout Shift & Floating UI Movement Fixes" that
  wasn't actually wrapped in its own `@media (max-width: 768px)` block for half its
  rules — `.chat-panel { display: none !important; }` was firing on **every** viewport,
  silently breaking the panel's close-fade animation on desktop, not just mobile.

Every one of these produced a real user complaint ("still see two scrolling bars", "it
just changed color") that looked, from the fix author's side, like it should already be
resolved — because it *was*, for the one copy they'd found.

## Steps
1. When a bug is reported as "still happening" after a fix that looked correct, or a
   widget/animation behaves inconsistently across pages or viewports, grep the exact
   selector/class/id/function name across the **whole** codebase — not just the file
   that seems responsible: `grep -rn "the-exact-name" js/ css/`.
2. If more than one file — or more than one top-level block within one CSS file —
   declares logic for the same target, read both in full before touching either one.
3. Check whether a nearby comment already explains a *previous* fix in this exact spot
   ("removed duplicate handler that was fighting..."). If so, the current bug may be a
   **new** instance of the same category, not the one already resolved — don't assume
   it's stale/dead just because it references something already fixed.
4. For CSS specifically: don't trust a section's comment header. Write (or reuse) a
   brace-depth-counting script to verify whether a rule is actually nested inside the
   `@media` block its title implies — copy-paste edits can silently land new rules
   above or below the intended wrapper.
5. Delete the redundant implementation entirely rather than patching around it. Leave
   exactly one, correctly-scoped source of truth, with a one-line comment noting what
   used to live there and why it was removed (future greps will find it).
6. After deleting: rebuild any minified bundle, bump cache-busting `?v=` params and the
   service worker `CACHE_NAME` (see `web_performance_seo` skill), verify brace/tag
   balance, and only then commit.
