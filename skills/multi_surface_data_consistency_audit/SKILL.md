---
name: multi_surface_data_consistency_audit
description: Catch stale or reverted business data (pricing, specs, tier structures) that's independently repeated across multiple pages/surfaces of a site — hub cards, detail pages, calculators, contact forms, structured data — by cross-referencing every surface against every other one instead of trusting any single page in isolation.
---

# Multi-Surface Data Consistency Audit

## Goal
Find places where the same business fact (a price, a bedroom count, a size tier list, a
per-sqft rate) is repeated across several independently-maintained surfaces of a site,
and one of them has silently gone stale or been reverted while the others haven't —
before a customer notices the contradiction first.

## Proof
On bongshaihousing.com this exact technique caught two real, live pricing bugs in one
pass:
1. All 12 Steel House model pages had been silently reverted to wrong tier/pricing data
   by a concurrent edit (see `duplicate_handler_debugging`-adjacent root cause: a broad
   `git add` swept in someone else's in-progress edit), while the hub page, the cost
   calculator, and the contact form's dropdown all still correctly advertised the
   newer, confirmed tier structure. Caught because cross-referencing the hub page
   against its own linked detail pages showed the hub promising one thing and the detail
   pages delivering another.
2. The Duplex Premium hub cards advertised a stale "starting from" price that predated a
   tier-structure standardization, while all 4 of that product's own detail pages had
   the correct, already-updated price.

A systematic sweep of 126 product cards across 14 hub pages against their linked detail
pages, plus cross-checking a cost calculator's per-category rate table and a contact
form's per-category size-tier dropdown against actual detail-page data, surfaced exactly
these 2 real mismatches out of hundreds of data points checked — and positively
confirmed everything else was consistent, which is itself a useful result (not just "no
news").

## Steps
1. Identify every surface where the same business fact is independently repeated —
   typically: a product-family hub/index page's summary cards, each item's own detail
   page, any calculator/estimator tool, any lead-capture form's dropdown options, and
   structured-data (JSON-LD) schema blocks. Each is a separate place a human — or a
   different agent working the same repo concurrently — could edit one without
   updating the others.
2. Write a small script rather than spot-checking by eye: extract the hub page's
   per-card claim (price, bed count, ...) via its card markup pattern, extract the same
   field from the linked detail page via its element-ID naming convention, and diff
   them programmatically across *all* instances at once. A regression that only shows
   up on 1 card out of 126 is exactly what a manual spot-check misses and a full sweep
   catches.
3. Expect a small number of ID-detection false positives when the site's internal
   element IDs don't perfectly track the URL slug (e.g. four different detail pages —
   `dv-110.html` through `dv-113.html` — that all happen to reuse the same internal id
   `101` in their element IDs and JS function names). Investigate every flagged
   mismatch manually before concluding it's a real bug — a naive script can't
   distinguish "genuinely different data" from "my regex assumed the wrong ID."
4. When two *different*, legitimately-separate product lines happen to display the same
   number (e.g., two unrelated products both priced at "30 Lakh"), don't assume that
   means they're linked or that a mismatch elsewhere must be an error. Verify against a
   third, independent, already-agreeing surface before deciding which of two
   disagreeing values is the correct one.
5. Once a real mismatch is found, trust whichever value has the most independent
   corroboration (agrees with the most *other* surfaces), rather than assuming "the
   newer edit" or "the older data" is automatically right. Three surfaces agreeing
   against one outlier is strong evidence of which one drifted.
6. Extend the habit to *computed* values, not just literal repeated text: a JSON-LD
   price range should be independently re-derived from the tier math (e.g. max-tier
   sqft × per-sqft rate) and compared against what's actually declared — a schema block
   is just another surface that can silently drift from the real pricing logic.
