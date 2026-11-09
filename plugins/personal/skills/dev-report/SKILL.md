---
name: dev-report
description: Generate a daily development report from a day's git commits — a time-phased timeline, verified improvements, and open issues. Use when the user asks for a dev report, daily report, end-of-day summary, or "summarize today's commits". Accepts an optional date argument (YYYY-MM-DD); defaults to today.
---

# Daily Development Report

Generate a Markdown development report for one day, reconstructed from that day's
git commits and the actual diffs behind them. The report is **evidence-based**:
every claim traces to a commit, a test result, or code you read — never invent
metrics.

**Core rule — every run:** the report is ALWAYS rebuilt fresh from the target
day's git commits (Step 2). Never reuse, copy, or lightly edit a previous report,
and never write a report from memory or conversation context. If a report file for
that date already exists, regenerate it from the commits and overwrite it. The
day's commits are the sole source of truth for what the report contains; if there
are no commits for that day, there is no report (Step 2).

## 1. Resolve the target date and output location

- **Date:** if the user passed a `YYYY-MM-DD` argument, use it. Otherwise use today
  (`date +%F`). Convert relative words ("yesterday") to an absolute date.
- **Output folder:** look for an existing dev-report folder, in this order:
  `docs/dev_reports/`, `docs/dev-reports/`, `dev_reports/`, `reports/`. If one
  exists, match its filename convention (the example uses `YYYY-MM-DD.md`). If none
  exists, default to `docs/dev_reports/` and create it.
- **Read one existing report** in that folder first (if any) and mirror its exact
  section structure, heading style, and tone. The format below is the default when
  the folder is empty.

## 2. Gather the commits (the spine of the report)

```bash
DAY=<resolved date>
# Commits authored that calendar day, oldest→newest, with time + subject.
git log --since="$DAY 00:00" --until="$DAY 23:59" --reverse \
  --pretty=format:"%h %ad %s" --date=format:"%H:%M"
```

If that returns nothing, tell the user there are no commits for that date and stop.

Capture the time span (first→last commit time) and the commit count for the Scope line.

## 3. Measure the net change

```bash
FIRST=$(git log --since="$DAY 00:00" --until="$DAY 23:59" --reverse --format=%H | head -1)
LAST=$(git log  --since="$DAY 00:00" --until="$DAY 23:59"          --format=%H | head -1)
git diff --shortstat "$FIRST^" "$LAST"   # files changed, insertions, deletions
```

Use this for the Scope line's "Net change: +X / −Y lines across N files". If a huge
delta is dominated by data/generated files, note that (e.g. "large deletions are
checked-in exports being removed") — check with `git diff --stat "$FIRST^" "$LAST"`.

## 4. Understand what actually changed

Don't summarize from commit subjects alone — they're often terse or have typos.
For non-trivial commits, read the real change:

```bash
git show <hash> --stat            # files touched
git show <hash>                   # full diff when the subject is unclear
```

Read enough to write an accurate one-line description and to judge impact. Group
commits into **time-based phases** (e.g. morning correctness work, afternoon
feature) — cluster by time gaps and theme, and give each phase a descriptive title
with its time range.

## 5. Assess current state (for the "Current State" section)

- `git status --short` → is the working tree clean?
- If the project has a test command (check README / pyproject / package.json), run it
  and report the pass/fail count. Only state numbers you actually observed.

## 6. Write the report

Write to `<folder>/<DAY>.md`. Reproduce this structure (the example report's format):

```markdown
# Development Report — <DAY>

**Scope:** <N> commits, <first>→<last>. Net change: **+<ins> / −<del> lines** across
<files> files (<one-clause note on any caveat like data-file churn>). <One sentence
on the day's overall arc — where the project started and ended.>

---

## Timeline of Changes

### Phase 1 — <theme> (<start>–<end>)
| Commit | Change |
|---|---|
| `<hash>` | <accurate one-line description; **bold** the key idea> |
...
(repeat per phase)

---

## Improvements
Numbered list of concrete wins. Where you can cite a measured number (test count,
coverage lift, rows verified), do — and only if you actually observed it. Otherwise
describe the qualitative improvement without inventing figures.

---

## Issues Yet to Be Resolved
Group by severity: **High impact**, **Medium / accepted**, **Low / latent**. Pull
these from: TODO/FIXME comments touched that day, limitations noted in commit diffs
or docs, failing/skipped tests, and any code-review findings. Be specific (file,
function, why it matters). Don't pad — omit a tier if it's empty.

---

## Current State
- Test status (e.g. "32/32 tests pass") and working-tree state, as observed.
- Any other one-line facts about where things stand.
```

## 7. Rules

- **Evidence only.** Never fabricate metrics, test counts, or line numbers. If you
  didn't run it or read it, don't assert it.
- **Trace every timeline row to a real commit hash.** Use the short hashes from `git log`.
- **Match an existing report's style** in the folder over this template when they differ.
- After writing, tell the user the path and give a 2–3 line summary of the day.
