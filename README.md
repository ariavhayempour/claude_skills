# claude_skills

A Claude Code plugin marketplace for my personal skills.

## Install

```
/plugin marketplace add ariavhayempour/claude_skills
/plugin install personal@ariav-skills
```

To pick up changes later: `/plugin marketplace update ariav-skills`.

## Skills

All skills live in the `personal` plugin, so they are invoked as `/personal:<skill>`.

| Skill | What it does |
|---|---|
| `commit` | Writes a commit message in the project's style, confirms it, and commits. Manual only. |
| `comment-cleanup` | Moves long code comments into docs, shortens what's left, strips task/story IDs. |
| `dev-report` | Builds a daily dev report from a day's git commits. |
| `issue` | Writes up a feature, bug, or chore as a GitHub issue for a kanban board. |

## Layout

```
.claude-plugin/marketplace.json     # marketplace manifest
plugins/personal/
  .claude-plugin/plugin.json        # plugin manifest
  skills/<skill>/SKILL.md
```

To add a skill, create `plugins/personal/skills/<name>/SKILL.md` and bump the version in
both `plugin.json` and `marketplace.json`.

