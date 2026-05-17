# my-claude-plugins

Personal Claude Code plugin marketplace. Private — install on your own machines via `gh` auth.

## Plugins

| Plugin | Description |
|---|---|
| `code-review` | Context-aware code-review orchestrator with parallel sub-skill dispatch (Java/Spring, Flask, MySQL, MongoDB, Redis, RabbitMQ). Line-anchored findings, no overall recaps. |

## Install on a new machine

Prereqs: `gh auth login` (so git can clone the private repo).

In a Claude Code session:

```
/plugin marketplace add git@github.com:motaher13/my-claude-plugins.git
/plugin install code-review@my-claude-plugins
```

Then restart Claude Code (or `/plugin reload`). Verify with `/plugin list` — `code-review` should show as installed, and the skill should auto-trigger on prompts like "review this PR".

## Update on each machine

```
/plugin marketplace update my-claude-plugins
/plugin upgrade code-review@my-claude-plugins
```

## Add a new plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json`
2. Drop skill folders under `plugins/<name>/skills/<skill-name>/SKILL.md` (auto-discovered)
3. Add an entry to `.claude-plugin/marketplace.json` under `plugins[]`
4. Bump `version` in `plugin.json`
5. Commit and push

## Local development

To test changes without pushing, point a Claude Code session at this directory directly:

```
/plugin marketplace add /Users/motaher/Projects/my-claude-plugins
/plugin install code-review@my-claude-plugins
```

After editing files, `/plugin marketplace update my-claude-plugins` picks up the changes.
