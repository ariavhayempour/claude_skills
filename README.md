# claude_skills

These are the [Claude Code](https://claude.com/claude-code) skills I use in my own day-to-day
development: committing, cleaning up comments, tracking work as issues, and writing up what
got done. I built them for myself, but they aren't tied to my setup or projects, so anyone is
welcome to install and use them. This is a personal project, so it doesn't take
contributions.

- [Installation](#installation)
- [The skills](#the-skills)
  - [`/personal:commit`](#personalcommit)
  - [`/personal:comment-cleanup`](#personalcomment-cleanup)
  - [`/personal:issue`](#personalissue)
  - [`/personal:dev-report`](#personaldev-report)
- [How they fit together](#how-they-fit-together)
- [Contributions](#contributions)

---

## Installation

### Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code/overview), installed and logged in
- `git`, which every skill relies on
- Optional: the [GitHub CLI](https://cli.github.com/) (`gh`), logged in with `gh auth login`.
  `/personal:issue` needs it only if you want it to file issues for you. Without it, the
  skill gives you text to paste in yourself.

### 1. Add the marketplace

Inside a Claude Code session, run:

```
/plugin marketplace add ariavhayempour/claude_skills
```

This registers the repo as a plugin marketplace called **`ariav-skills`**.

### 2. Install the plugin

```
/plugin install personal@ariav-skills
```

You can also run `/plugin` on its own, open **Browse marketplaces → ariav-skills**, and pick
`personal` from the menu.

### 3. Load it

```
/reload-plugins
```

Or restart Claude Code. Then type `/personal:` and the four skills should autocomplete.

### Installing from the terminal

Without opening a session first:

```bash
claude plugin marketplace add ariavhayempour/claude_skills
claude plugin install personal@ariav-skills
```

### Installing for a whole team or repo

To have the plugin offered automatically to everyone who opens a project, add this to that
project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "ariav-skills": {
      "source": { "source": "github", "repo": "ariavhayempour/claude_skills" }
    }
  },
  "enabledPlugins": {
    "personal@ariav-skills": true
  }
}
```

### Updating

```
/plugin marketplace update ariav-skills
```

Then run `/reload-plugins`.

### Uninstalling

```
/plugin uninstall personal@ariav-skills
/plugin marketplace remove ariav-skills
```

---

## The skills

Everything ships in one plugin called `personal`, so each skill is invoked as
`/personal:<skill>`. Apart from `commit`, they also load on their own when you ask for the
same thing in plain English, such as "write this up as an issue" or "give me today's dev
report".

| Skill | In one line |
|---|---|
| [`/personal:commit`](#personalcommit) | Commit with a message that matches the repo's convention, after you approve it |
| [`/personal:comment-cleanup`](#personalcomment-cleanup) | Collapse long comments to one-liners and move the detail into docs |
| [`/personal:issue`](#personalissue) | Turn a rough idea into a GitHub issue that's ready for a board |
| [`/personal:dev-report`](#personaldev-report) | Write an evidence-based report of one day's commits |

---

### `/personal:commit`

Writes a commit message that fits the project's existing style, shows it to you, and commits
only once you approve.

**Usage**

```
/personal:commit                  # use what's staged, or choose from your changes
/personal:commit src/auth.ts      # stage and commit only these paths
```

**How it works**

1. **Staging.** If you name files, it stages exactly those. If something is already staged,
   it uses that as is. Otherwise it groups your changes by area and asks whether to stage
   everything or one group at a time. It always stages explicit paths and never runs
   `git add .` or `git add -A`. It leaves out secrets, local config such as `.env*`, and
   build caches, and tells you what it skipped.
2. **Learning the convention.** It checks, in priority order: a commitlint config, commit
   rules in `CLAUDE.md`, and the style of the last 20 commits. If none of those apply, it
   uses [Conventional Commits](https://www.conventionalcommits.org/)
   (`feat`, `fix`, `docs`, `refactor`, ...).
3. **Writing the message.** It picks a scope only when one already exists in the history,
   adds a body only when the *why* isn't obvious from the diff, and keeps the header to
   about 50 characters (72 at most). It adds no `Co-Authored-By` or other attribution
   trailers.
4. **Confirming.** It shows the branch, the number of files, and the full message, and
   waits for you to pick **Commit** or **Edit**.
5. **Handling failures.** If a commit hook rejects the message, it fixes the message and
   retries once. If a formatter hook changes files, it re-stages them and makes a new commit.

**Guarantees:** it never pushes, never amends, and never uses `--no-verify`. It runs only
when you invoke it, so Claude never commits on its own.

---

### `/personal:comment-cleanup`

Keeps code comments short. Each comment states one non-obvious *why* on a single line, and
anything longer moves into a docs file.

**Usage**

```
/personal:comment-cleanup                 # only the lines you've changed but not staged
/personal:comment-cleanup src/lib/api/    # these files in full
/personal:comment-cleanup all             # the whole repo
```

**How it works**

1. **Scope.** By default it touches only comments on lines your **unstaged** diff adds or
   changes. It leaves staged work, older comments elsewhere in the file, and files you
   haven't changed alone. It says which files it will process before editing anything.
2. **Classifying each comment** as keep, shorten, move to docs, delete, or strip an ID:
   - Two or more consecutive comment lines always count as a "long comment". It moves them
     to docs and leaves one concise line in their place.
   - Comments that only restate *what* the code does are deleted.
   - It removes planning IDs such as `T5`, `Story 12` or `US-4`, replacing each with the
     technical reason it stood for.
   - It leaves alone: genuine one-line *why* comments, license headers, `TODO`/`FIXME`
     markers, lint and type pragmas, `@param`-style doc tags, and shebangs.
3. **Writing the docs.** Moved content goes into `docs/claude/<area>.md`, or `docs/` if that
   folder doesn't exist. It appends to an existing file for the same area when there is one.
   These docs are written for an AI to read: the fact comes first, with short headings,
   one fact per bullet, and exact symbol names.
4. **Verifying.** It confirms that only comments changed, that no file outside the
   announced set was touched, and that the staged snapshot is unchanged. Then it reports
   what it shortened, moved, deleted or stripped, plus anything it deliberately left alone.

**Guarantees:** it never changes executable code. It moves rationale to docs rather than
deleting it. Comments never point to the docs folder, because those pointers go stale.

---

### `/personal:issue`

Turns a rough description into a clean GitHub issue, with checkbox acceptance criteria that
show as progress on a kanban or project board.

**Usage**

```
/personal:issue login page flashes unstyled content on slow connections
/personal:issue                           # it asks what the issue is about
```

**How it works**

1. **Gathering input.** It reads your description and `CLAUDE.md` for the project's
   vocabulary, then classifies the issue as a **feature**, **bug**, **task/chore**, or
   **spike**. If something essential is missing, it asks rather than guessing: who, what
   and why for a feature; expected vs. actual behavior and steps to reproduce for a bug.
2. **Learning the repo's style.** This happens only when filing through `gh`. It follows
   `.github/ISSUE_TEMPLATE/` if the repo has one, and suggests only labels that already
   exist (`gh label list`).
3. **Drafting.** It writes an imperative title and the smallest body that fits: **Summary**,
   **Details**, **Acceptance Criteria** (`- [ ]` checkboxes), and optional **Tasks**.
4. **Delivering.** You choose one of:
   - **Copy manually** (the default): the title, plus the body in one fenced block to paste
     into GitHub.
   - **File via gh**: it runs `gh issue create`, with labels, assignee or project board if
     you name them, and prints the issue URL.
   - **Edit**: you give feedback and it redrafts.

**Guarantees:** it never makes up acceptance criteria or repro steps, and never files an
issue without asking you first. If `gh` fails, it diagnoses the error and retries once.
If `gh` isn't installed, it falls back to copy-paste.

---

### `/personal:dev-report`

Builds a Markdown report of one day's development from that day's git commits and the diffs
behind them.

**Usage**

```
/personal:dev-report               # today
/personal:dev-report 2026-11-08    # a specific day
```

**How it works**

1. **Finding the output folder.** It uses the first of these that exists:
   `docs/dev_reports/`, `docs/dev-reports/`, `dev_reports/` or `reports/`. If none exist,
   it creates `docs/dev_reports/`. When earlier reports are there, it copies their structure
   and tone.
2. **Gathering commits.** It lists every commit from that day in time order. If there are
   none, it says so and stops.
3. **Measuring the change.** It works out the net lines added and removed and the number of
   files touched. It also points out when generated or data files account for most of the
   churn.
4. **Reading the actual diffs.** It doesn't trust terse commit subjects. It groups commits
   into phases by time and theme, such as morning bug fixes and an afternoon feature.
5. **Checking the current state.** It checks whether the working tree is clean and runs the
   project's tests if it can find a test command.
6. **Writing** `<folder>/YYYY-MM-DD.md` with these sections:
   - **Scope:** commit count, time span, net change, and a sentence on the day's overall arc
   - **Timeline of Changes:** one table per phase, each row tied to a commit hash
   - **Improvements:** concrete wins, with numbers only where they were actually measured
   - **Issues Yet to Be Resolved:** sorted into high, medium and low impact
   - **Current State:** test results and the state of the working tree

**Guarantees:** everything in the report traces back to a commit, a test run, or code it
read. It invents no metrics. It rebuilds the report from git every time and never reuses an
earlier one, overwriting any existing report for that date.

---

## How they fit together

A typical day looks like this:

1. **Plan:** capture work as it comes up with `/personal:issue`.
2. **Build:** write code as usual.
3. **Tidy:** run `/personal:comment-cleanup` on your unstaged changes before you stage them.
4. **Commit:** use `/personal:commit` for a message that matches the repo's style.
5. **Wrap up:** run `/personal:dev-report` at the end of the day for a summary of what shipped.

---

## Contributions

This is a personal project, and I'm not accepting contributions, issues, or pull requests.
You're welcome to install and use the skills as they are.
