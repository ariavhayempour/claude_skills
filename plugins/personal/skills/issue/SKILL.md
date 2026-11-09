---
name: issue
description: Use when the user describes a feature, bug, task, or chore they want to track and wants it written up as a GitHub issue to drop onto a kanban / project board. Triggers on "make an issue", "write this up as a GitHub issue", "add this to my board", "create a ticket / card".
argument-hint: [what the issue is about]
allowed-tools: Bash, Read, Glob, AskUserQuestion
---

# Issue

Turn a rough description into a clean, copy-paste-ready GitHub issue for a kanban board.

Think of this as the `story` skill's sibling: same "clarify then draft" instinct, but the deliverable is a GitHub issue (title + markdown body with checkbox acceptance criteria) rather than a user-story file.

## Execution Flow

### Phase 1: Gather Input

If `$ARGUMENTS` is provided, use it as the description. Otherwise use `AskUserQuestion`: "What do you want to file an issue for?"

Read `CLAUDE.md` (if present) for project terminology, tech stack, and conventions so the issue reads like it belongs in this repo.

Classify the issue as **feature**, **bug**, **task/chore**, or **spike**. If the description is missing the core of any one — for a feature: who/what/why; for a bug: expected vs. actual + repro — ask focused questions with `AskUserQuestion`. Do NOT invent acceptance criteria or repro steps.

### Phase 2: Learn Repo Style (only if filing via gh)

If the repo has an issue template (`.github/ISSUE_TEMPLATE/`), follow it. Check existing labels with `gh label list` so any suggested labels actually exist. Skip this phase if the user only wants copy-paste text.

### Phase 3: Draft the Issue

Produce a **title** and a **markdown body**. Match the body shape to the type:

**Title** — concise, imperative, no trailing period. A `type:` prefix (`feat:`, `fix:`, `chore:`) is fine if the repo uses them; omit if unsure.

**Body** — use the smallest structure that fits. Acceptance criteria / tasks are checkboxes so they track as progress on the board.

```markdown
### Summary
One or two sentences: what and why.

### Details
Context, constraints, links. For a bug: **Expected**, **Actual**, **Steps to reproduce**.

### Acceptance Criteria
- [ ] testable condition 1
- [ ] testable condition 2

### Tasks (optional)
- [ ] implementation step
```

Suggest labels only when they map to real repo labels.

### Phase 4: Deliver

Present the drafted title and body, then use `AskUserQuestion`:

- **Copy manually** — print the title on one line, then the body inside a single fenced ```markdown block so it's one clean copy into GitHub's New Issue form or a project card.
- **File via gh** — run `gh issue create --title "<title>" --body "<body>"` (add `--label`, `--assignee`, or `--project "<board>"` when the user names them). Print the returned URL.
- **Edit** — take feedback, regenerate.

Default to **Copy manually** — the user's usual intent is to paste it onto the board themselves.

### Phase 5: Handle gh Failures

If `gh issue create` fails, read the error, diagnose (not authenticated → `gh auth status`; label/project doesn't exist → drop or correct it), fix, and retry once. Do NOT retry the same command unchanged. If gh isn't available, fall back to Copy manually.

## Anti-Patterns

| Don't | Instead |
| ----- | ------- |
| Invent acceptance criteria or repro steps | Ask with `AskUserQuestion` until they're concrete |
| Write a multi-paragraph essay for a small task | Summary + a couple of checkboxes is enough |
| Suggest labels that don't exist in the repo | Run `gh label list` first, or suggest none |
| Put implementation detail in the title | Keep the title about the outcome; detail goes in the body |
| File via gh without confirming | Always confirm through the Phase 4 question |
