# arthur-agent-plugins

Arthur's shared skill bundle for Claude Code and Codex. The Claude marketplace can also be synced into the org plugin library for Claude Tag.

## Plugins

### arthur-tools

Skills for working with the Arthur platform.

1. **`arthur-dashboards`** - writes and debugs SQL for Arthur dashboard charts, stat cards and alert rules

### Skills

The `skills/` folder works with both Codex and Claude Code. Each skill's, `SKILL.md` is the single source of instructions, and the coding agent can read the any `references/` files when the task calls for them.

## Install in Codex

```bash
codex plugin marketplace add arthur-ai/arthur-agent-plugins
codex plugin add arthur-tools@arthur-agent-plugins
```

Start a new Codex session to use the installed skill.

## Install in Claude Code

```bash
claude plugin marketplace add arthur-ai/arthur-agent-plugins
claude plugin install arthur-tools@arthur-agent-plugins
```

## Add to Claude Tag

An organization owner can make the plugin available to a Claude Tag channel:

1. In **Organization settings → Plugins & skills → Marketplaces**, choose **Add plugins → Sync from GitHub** and enter `arthur-ai/arthur-agent-plugins`. The repository must be private or internal, and the Claude GitHub App must have access to it.
2. In **Organization settings → Claude Tag**, select the channel in the access editor and add **arthur-tools** under **Plugins**. Add it at the workspace or organization scope to make it available to more channels.
