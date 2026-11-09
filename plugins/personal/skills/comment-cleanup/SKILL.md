---
name: comment-cleanup
description: Move long code comments into a docs file and reduce in-code comments to concise one-liners, and strip planning identifiers (user-story numbers, task numbers like T5) from comments in unstaged changes. Use when the user runs /comment-cleanup or asks to clean up comments, shorten comments, move comments to docs, or remove story/task numbers from comments.
argument-hint: '[files/paths/globs | "all"] — omit to clean only unstaged changes'
---

# Comment Cleanup

Reduce in-code comments **inside the current unstaged changes** to concise
one-liners, relocate the substantive content into a project docs file, and strip
planning identifiers (user-story numbers, task numbers like `T5`) from those
comments.

**Core principle:** Code comments explain a single non-obvious *why* in one line.
Anything longer — rationale essays, design notes, multi-step explanations — belongs
in docs, not in the source file. Planning identifiers are mutable and go stale, so
they never belong in code comments. Every one-liner must be CONCISE: the shortest
phrasing that still names the *why* — no restated context, no hedge words, no
trailing clause repeating what the code already shows. "One line" means one
physical source line: a single `//`/`#`/`--` comment token followed by text with
no line break, not the same sentence wrapped or continued onto a second comment
line. If the concise phrasing still doesn't fit on one line, cut words until it
does — never wrap it onto a second line instead.

## Scope: unstaged changes only, and only the changed lines

**Default (no scope given): operate only on the current unstaged changes.** Two
independent narrowings apply, and both are mandatory:

- **Unstaged only.** Staged content is off-limits — it is already prepared for
  commit. If a file is fully staged with no further working-tree edits, skip the
  file entirely. If a file is partially staged, process only its unstaged portion.
- **Changed lines only.** Within an in-scope file, only touch comments that the
  unstaged diff actually adds or modifies. Pre-existing comments elsewhere in the
  file are out of scope even when they violate every rule in this skill. Never
  clean a whole file just because one hunk in it changed, and never work from the
  full branch diff against main.

Determine the scope this way:

1. Run `git diff --name-only` for unstaged changes to tracked files, and
   `git ls-files --others --exclude-standard` for untracked files. Do **not** use
   `git diff --cached`, and do not use bare `git status --porcelain` as the source
   of truth — it does not separate staged from unstaged.
2. Expand any listed directories to their contained files.
3. Filter to source/code files (skip binaries, lockfiles, generated output, and the
   docs file itself). Untracked listings routinely surface planning artifacts —
   `SPEC.md`, `plan.md`, `todo.md`, `tasks/` — which are never in scope; drop them
   here.
4. For each tracked file in the set, run `git diff -U0 -- <file>` and record the
   added/modified line ranges. Those ranges are the only lines you may edit.
   Untracked files are entirely new, so all of their lines are in scope.

If the result is empty, report that there are no unstaged changes to clean and stop.

### Comments that straddle the diff boundary

A multi-line comment block counts as in scope if the unstaged diff touches **any**
of its lines. Treat the whole block as the unit of work — collapse and relocate it
as a single comment, even though only part of it appears in the diff. This is the
one case where you may edit a line outside the recorded ranges, and it extends only
to the rest of that same comment block. It never licenses touching a neighboring
comment.

### Scope override (only when the user specifies one)

Honor an explicit scope in the invocation *instead of* the unstaged-change set — but
only when the user actually states one. Examples:

- **Explicit files / paths / globs** (e.g. `/comment-cleanup src/lib/api/` or naming
  a file): process exactly that set in full, whether or not those files changed. An
  explicitly named file lifts both the unstaged-only and changed-lines-only limits
  for that file.
- **Whole repo** — only when the user clearly asks for it ("all files", "the whole
  codebase", "every file"): sweep the repo.

Never *widen* scope on your own. Absent an explicit instruction, the unstaged changed
lines above are the scope. If a stated scope is ambiguous, confirm before running
rather than defaulting to a broader sweep.

## Where the docs go

Relocated content goes into the project's documentation directory:

- If `docs/claude/` exists (or the project's `CLAUDE.md`/conventions point to a
  docs location), use it. This is the default for this convention.
- Otherwise use `docs/`, creating it if needed.
- Prefer **one topically-named file per source area** (e.g. `docs/claude/<area>.md`)
  over a single dumping ground. If a relevant doc file already exists for that area,
  append to it rather than creating a duplicate.

Never move planning artifacts, and never touch `SPEC.md`, `plan.md`, `todo.md`, or
a `tasks/` folder.

## Docs are for AI, not humans — format accordingly

Files under `docs/claude/` (or an equivalent project-specific AI-context
directory) are read by an LLM, not skimmed by a person, and get pulled into
context wholesale. Optimize for retrieval and machine parsing over prose
readability:

- Lead each entry with the concrete fact/rule, not narrative build-up — an LLM
  reading a chunk out of order needs the payload in the first sentence.
- Use short, declarative headings per topic (`##`/`###`) so a section can be
  grepped or retrieved on its own without needing the surrounding sections.
- Prefer bullet lists and short paragraphs over long flowing prose; keep one
  fact per bullet so it's independently quotable.
- Name the exact symbols (`function`, `type`, `event` names, file paths) the
  content describes — verbatim identifiers are what makes a doc chunk
  matchable back to the code that links to it.
- Keep each doc file topically scoped (see above) so a retrieval hit pulls in
  only relevant content, not an entire mixed-topic file.
- Still write for correctness and completeness — optimizing for AI retrieval
  means structure and density, not omitting information a human moved here
  would have wanted.

## What counts as a "long comment"

**Two or more consecutive comment lines are always a long comment**, no matter how
short each individual line reads — a genuine one-line *why* is the only comment
that may stand as-is. There is no "these are short enough to leave stacked"
exception: two three-word `//` lines back to back are still two physical lines,
so they still get collapsed and relocated by this rule.

Move to docs and replace with a one-liner (or delete) when a comment:

- Spans **more than one physical line** — a multi-line block, consecutive `//`/`#`
  lines forming a paragraph, *or* one sentence deliberately wrapped/continued across
  two or more comment lines to stay under a length limit — OR
- Is a single line but packs multiple sentences / a multi-step explanation, OR
- Restates *what* the code does rather than *why* (these usually just get deleted,
  not moved — don't preserve noise in docs).

A relocated comment's replacement one-liner is exactly one physical line. Never
split its *why* across two shortened comment lines to keep each one short —
shorten the wording instead until it fits on the one line. Concatenating the
original lines onto one line is not a substitute for relocating: if the merged
text still packs more than a single concise *why*, it still needs to move to
docs, replaced by a distilled one-liner — not a squeezed-together transcript of
the original lines.

Leave alone: a genuine one-line *why*, license headers, `TODO`/`FIXME` markers,
type/eslint pragmas, doc-comment tags a tool consumes (e.g. `@param` that generates
API docs), and shebangs.

## Planning identifiers to strip

Remove references in comments to mutable planning identifiers, regardless of comment
length:

- User-story numbers (e.g. `Story 12`, `US-4`, `#123` when it denotes a story)
- Task numbers (e.g. `T5`, `Task 3`, `step 2` referring to a plan)
- Sprint / ticket-in-plan references used to pin code to a planning doc

Replace the identifier with the **technical reason** it stood for (behavior,
constraint, dependency). If the comment was *only* the identifier and carried no
technical meaning, delete it. Do not touch identifiers in commit messages, docs, or
string literals — comments only.

## Never reference the docs directory in comments

No in-code comment may reference `docs/claude/` (or the project's equivalent
AI-context docs directory) or name a specific file inside it — not as a pointer,
not as a "see docs" note, not even after a move-to-docs relocation. That directory
is mutable and reorganized independently of the code; a comment pointing into it
goes stale exactly like a planning identifier does. When relocating a long comment
per this skill, replace the in-code comment with a plain one-line *why* only —
never append a `// see docs/claude/<area>.md`-style pointer.

## Workflow

1. **Determine scope, then announce** the exact file set you'll process. Use the
   unstaged-change set by default; use an explicit scope only if the user gave one
   (see Scope override). State which of the two you're using so it's visible before
   edits, and note any file you skipped for being fully staged.
2. **For each file**, scan only the comments within the recorded unstaged line
   ranges (plus any block those ranges straddle). Classify each as: keep-as-is,
   shorten-in-place, move-to-docs, delete, or strip-identifier. Comments outside
   those ranges are not yours to touch — leave them even if they clearly violate
   this skill's rules.
3. **For move-to-docs:** write the substantive content to the appropriate
   `docs/claude/<area>.md` under a clear heading, formatted per "Docs are for AI,
   not humans" above, then replace the in-code comment with a single concise line
   capturing the *why* only — never a pointer back to the docs file or directory
   (see "Never reference the docs directory in comments"). Preserve the technical
   meaning — never silently drop rationale; relocate it. Keep the line CONCISE:
   state the *why*, cut everything else. The replacement is one physical comment
   line — if it doesn't fit, trim the wording further; don't wrap it onto a second
   line.
4. **Preserve behavior:** only comments change. Never alter code, logic, or
   formatting of executable lines. If shortening a comment would lose information,
   the lost information goes to docs, not the bit bucket.
5. **Verify:** re-read the comments you edited; confirm each is now a single line and
   free of planning identifiers. Then run `git diff --stat` and confirm you touched
   no file outside the announced set, and `git diff --cached --stat` to confirm the
   staged snapshot is unchanged. If the project has a build/lint/test gate, note that
   only comments changed so behavior is unaffected.
6. **Report:** list each file touched, comments shortened/moved/deleted, identifiers
   stripped, and the docs file(s) written. Also note anything deliberately left
   alone — files skipped as staged, and out-of-scope comments that would have
   qualified — so the untouched violations are visible rather than looking overlooked.

## Common mistakes

- **Sweeping the whole repo.** Unstaged changed lines only, unless the user
  explicitly named a broader scope. Re-check the scope step and never widen it on
  your own.
- **Cleaning a whole file because one hunk in it changed.** The file is the
  container, not the unit of work. Only the unstaged added/modified lines are in
  scope; a long comment 200 lines away from your hunk stays exactly as it is.
- **Touching staged content.** Staged changes are already prepared for commit and
  are out of scope. Editing them silently rewrites what the user was about to
  commit. Skip fully-staged files; in partially-staged ones, work only on the
  unstaged portion.
- **Using `git status --porcelain` to build the file set.** It merges staged and
  unstaged state into one listing and will pull staged-only files into scope. Use
  `git diff --name-only` plus `git ls-files --others --exclude-standard`.
- **"Fixing" a nearby violation while you're in there.** Out-of-scope comments get
  reported, not edited — bundling unrelated cleanups into the user's working diff
  is the exact problem this scoping prevents.
- **Deleting rationale instead of moving it.** Long ≠ worthless. Move substance to
  docs; only delete pure *what*-restating noise.
- **Editing code, not just comments.** This skill never changes behavior.
- **Leaving a multi-line block because each line is short.** Two or more
  consecutive comment lines are always a long comment, regardless of how short
  each line is — collapse and relocate, no "short enough to stack" exception.
- **Squeezing two comment lines onto one instead of relocating.** Joining the
  original lines together isn't shortening — if the merged text still carries
  more than a single concise *why*, move it to docs and write a distilled
  one-liner in its place.
- **Wrapping a shortened comment across two lines to keep each one short.** A
  one-liner is one physical line, period — trim the wording, don't split it onto
  a continuation line.
- **Creating a new docs file when a fitting one exists.** Append to the existing
  area doc.
- **Leaving a `// see docs/claude/...` pointer after relocating a comment.** Never
  reference the docs directory or a file inside it from a code comment — replace
  with a plain one-line *why* only.
