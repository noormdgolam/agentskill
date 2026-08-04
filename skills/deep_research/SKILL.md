---
name: deep_research
description: Run any multi-subject research task (competitor/market research, technology or vendor comparison, literature review) as a fixed items-x-fields grid instead of a freeform report. Makes coverage gaps visible instead of hidden inside prose, marks uncertain cells instead of guessing, ranks sources by reliability tier, and stays resumable across a long or interrupted session.
---

# Deep Research (Grid Method)

## Goal
Prevent the two failure modes that freeform research reports fall into: (1) silently
skipping a subject or a data point because nothing forced you to notice the gap, and
(2) writing a confident-sounding sentence for something you actually couldn't verify.
Do this by fixing the research **grid** — the list of subjects and the list of fields to
collect for each — before searching, not after.

Learned from three research-skill repos (`lingzhi227/agent-research-skills`,
`Weizhena/Deep-Research-skills`, ComposioHQ's `content-research-writer`), which build
this pattern for academic literature reviews. Adapted here for market/competitor
research, technology comparisons, and vendor evaluation — the kind of research this
project actually needs.

## Proof
`Skill_Manpower_Bangladesh_Research.md` in this project — market research on Bangladeshi
manpower/training-center competitor sites — already hit exactly the failure modes this
method targets, before the method existed:
- The competitor table (10 sites x type/sectors/notes) is, in effect, an items x fields
  grid already. That's the shape the domain wants.
- Several sites (`asiattcbd.com`, `starbanglattc.com`, `bashundhara-ttc.com`,
  `bashundhara-ttc.com`'s Laravel-rendered pages) are JS-rendered and returned a blank
  shell to a plain fetch. The report had to call this out as a footnote rather than
  silently listing those rows as "no notable info" — which is exactly the "mark
  uncertain, don't guess" discipline below, just applied ad hoc instead of by default.
- The report was delivered as a single hand-assembled document. A 40-row grid assembled
  by hand risks item #35 getting less careful treatment than item #2 — the "compile
  deterministically" step below exists to prevent that drift.

## Steps

### 1. Fix the grid before searching
Write down the **rows** (subjects: companies, tools, agencies, papers) and the
**columns** (fields to collect for every row) as an explicit list — even just a markdown
table header — before running a single search. An empty cell is visible; a missing
paragraph in a narrative report isn't. If a client/user gave you a template or asked
specific questions, those become your columns directly.

### 2. Rank sources by tier, and say which tier you used
For market/competitor research, in order of trust:
1. Official/government registries — e.g. a regulator's licensed-agency directory
   (for Bangladesh manpower work, `oep.gov.bd/agencies` is exactly this — the
   authoritative source for "every licensed recruiting agency")
2. The subject's own site/filings/press releases (self-reported — read as promotional)
3. Established trade/industry press
4. General news coverage (good for dates/corroboration, weak for technical detail)
5. Directory/aggregator sites, forums — fine for *discovery* of more candidates, not fine
   as the sole source for a factual claim

A claim resting only on tier 4-5 should be marked uncertain rather than stated flatly,
unless two or more independent tier-4/5 sources agree.

### 3. Mark uncertain cells instead of guessing
When a fetch/search genuinely can't confirm a field, write `[uncertain]` (or leave the
cell empty with a footnote) rather than inferring a plausible-sounding value. This is
the single highest-leverage rule in the whole method — an honest gap is useful
information; a guess dressed as fact is a liability once someone acts on it.

### 4. Flag JS-rendered / fetch-blind sites explicitly — don't silently drop them
A plain `WebFetch` against a JS-rendered (React/Vue/Laravel-SPA) site returns an empty
shell, not "no information exists." Treat that as a distinct outcome from "searched and
found nothing": log it as `needs browser-based pass` and either do that pass (if you
have browser tooling available) or hand it back to the user as an explicit open item.
Silently treating a blank fetch as a data-free row is how real competitors go missing
from a competitor analysis.

### 5. Fan out one search per row, and make it resumable
For anything beyond a handful of rows, research one item at a time (one search pass /
one agent per row) rather than trying to hold the whole grid in your head at once. Save
each row's result as soon as it's found (a JSON file, a row appended to a working table)
so that if the session is interrupted, resuming means "skip rows that already have a
result," not "start over."

### 6. Compile the final report deterministically
Once every row has a result, assemble the final table/report by iterating the same
saved per-row results the same way for every row — don't hand-write item #2 carefully
and item #35 sparsely. If the row count is large, write a short script that reads the
per-row data and emits the table/report mechanically; that's what guarantees uniform
treatment and is trivial to re-run if a row gets corrected later.

## Related
- `multi_surface_data_consistency_audit` — once a grid like this exists, the same
  cross-referencing discipline applies to catching whether business data drifted across
  your *own* site's surfaces, not just competitors'.
