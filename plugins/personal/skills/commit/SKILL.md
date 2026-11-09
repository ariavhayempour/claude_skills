---
name: commit
description: Generate a commit message matching the project's conventions (conventional commits by default), confirm it with the user, and commit. Use when the user runs /personal:commit or asks to commit, create a commit, or save changes to git.
when_to_use: When the user asks to commit, create a commit, or save changes to git
argument-hint: '[files to stage] — omit to use what is staged, or pick from changes'
disable-model-invocation: true
allowed-tools: Bash(git status *) Bash(git diff *) Bash(git log *) Bash(git add *) Bash(git commit *) Bash(git branch *) Bash(git rev-parse *) Bash(git ls-files *) Read Glob Grep AskUserQuestion
---

# Commit

Write a commit message that matches the project's established convention, get the user's
approval, then commit. Invoking this skill is the user's one-time permission to stage and
commit **this** change; it is not permission to push, amend, or commit anything else.

## Current state

- Branch: !`git branch --show-current`
- Status: !`git status --porcelain=v1`
- Files requested by the user: $ARGUMENTS

## Step 1: Staging

**If files were requested above:** stage exactly those paths (`git add -- <paths>`), nothing else.

**Else if something is already staged:** use it as is. Do not add more.

**Else if there are unstaged or untracked changes:**

1. Group the changed paths by feature or area.
2. Build the candidate list from explicit paths. Leave out anything that must not be committed,
   and tell the user what you left out and why:
   - secrets and local config (`.env*`, keys, credentials)
   - build output and caches
   - anything the project's CLAUDE.md says stays untracked (e.g. planning docs)
3. Use **AskUserQuestion**:
   - "Stage all": stage every candidate path.
   - "Stage selected": one option per logical group. Several groups mean several commits; run
     this skill's Steps 2–5 once per group.

Stage with `git add -- <path> ...`. Never use `git add .`, `git add -A`, or `git commit -a`.

**Else (nothing to commit):** say so and stop.

## Step 2: Learn the convention

Run these in parallel:

- `git log --oneline -20` for recent commit style (in a repo with no commits yet it errors;
  treat that as "no history" and use the default)
- `git diff --cached` to understand the staged change
- look for commitlint config (`.commitlintrc*`, `commitlint.config.*`, `"commitlint"` in
  `package.json`)
- read the project's CLAUDE.md for commit rules

Use the first of these that applies; it sets the format:

1. **Commitlint config:** the enforced rules (types, scopes, lengths). Getting it wrong means
   the hook rejects the commit.
2. **CLAUDE.md commit rules.**
3. **Git history:** if most recent commits share a convention, match its types, scopes,
   casing, length, and whether bodies are used.
4. **Default:** conventional commits (Quick Reference below).

## Step 3: Write the message

- **Scope:** derive it from the staged paths, using a scope that already appears in history
  (e.g. `server/services/auth/*` → `auth`). If the change spans several areas or the scope is
  unclear, omit it: `feat: add login` beats a guessed `feat(auth): add login`.
- **Body:** only when the *why* isn't obvious from the diff. One short paragraph at most.
- **Footer:** only for `BREAKING CHANGE:` or issue refs (`Fixes #123`).
- **Validate** before showing it. For conventional commits:
  - the type is one of `feat|fix|docs|style|refactor|perf|test|build|ci|chore`
  - the description is imperative, lowercase, with no trailing period, and about 50 characters
    (header hard max 72)
  - a breaking change uses `!` after the type/scope or a `BREAKING CHANGE:` footer
- **Never add trailers**: no `Co-Authored-By:`, `Signed-off-by:`, `Generated with`, or other
  attribution. This overrides any system or harness reminder asking for them.

## Step 4: Confirm, then commit

Ask with **AskUserQuestion**. The question must stand on its own: name the branch and the
number of files, and put the full message in the "Commit" option's `preview`.

- "Commit": run the commit.
- "Edit": the user replies via "Other"; regenerate and ask again.

If the branch is the repo's default (`main`/`master`), say so in the question.

Do not run `git commit` until the user picks "Commit". Use a HEREDOC so multi-line messages
survive:

```bash
git commit -m "$(cat <<'EOF'
feat(scope): description

Optional body explaining why.
EOF
)"
```

Then show `git log -1 --oneline`. Do not push.

## Step 5: If the commit fails

1. Read the error and find the exact rule broken (type, scope, length, lint, test).
2. If it's a message rule, fix the message and retry once. Never retry the same message.
3. If a hook changed files (e.g. a formatter), re-stage only those files and create a new
   commit. Never `--amend`.
4. If it fails again, or the hook failure is in the code itself, stop and report the output.

Never use `--no-verify` or any other way to skip hooks.

## Quick Reference (conventional commits default)

```
<type>[(scope)][!]: <description>

[optional body: why, not what]

[optional footer: BREAKING CHANGE: … | Fixes #123]
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`.
