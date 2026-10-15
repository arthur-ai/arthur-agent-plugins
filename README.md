# arthur-agent-plugins

Arthur's shared skill bundle for ChatGPT, Claude, Claude Code, and Codex. The Claude marketplace can also be synced into the org plugin library for Claude Tag.

## Plugins

### arthur-tools

Skills for working with the Arthur platform.

1. **`arthur-dashboards`** - writes and debugs SQL for Arthur dashboard charts, stat cards and alert rules

### Skills

The `skills/` folder works with Codex and Claude Code. Each skill's `SKILL.md` is the single source of instructions, and the agent can read its `references/` files when the task calls for them.

## Use in ChatGPT or Claude on the web

Opening a chat with the skill's GitHub link lets the assistant read it for that conversation. It does not install the skill for future chats. Install it once in each account where you want to use it.

From the repository root, package the complete skill, including `references/`. Ex:

```bash
(cd plugins/arthur-tools/skills && zip -r ../../../arthur-dashboards.zip arthur-dashboards)
```

### ChatGPT

1. Open **Plugins → Skills → Installed** and check for `arthur-dashboards`. If you already installed the **arthur-tools** plugin, you can use its skill instead.
2. If it is missing, select **Create → Upload from your computer** and upload the `arthur-dashboards.zip` created in the repository root.
3. Once installed, start a new chat and ask ChatGPT to use `arthur-dashboards`.

Skill uploading is available to eligible Business, Enterprise, Healthcare, and Edu users, subject to workspace settings. If **Create** or **Upload from your computer** is unavailable, check with your workspace administrator. See [OpenAI's current skill instructions](https://help.openai.com/en/articles/20001066-skills-in-chatgpt).

### Claude

1. Open **Customize → Skills → + → Create skill → Upload a skill**.
2. Upload the `arthur-dashboards.zip` created in the repository root and enable the skill.
3. Start a new chat and ask Claude to use `arthur-dashboards`.

If your organization has added **arthur-tools** to its plugin catalog, you can instead open **Customize → Plugins → Discover**, select **arthur-tools**, and click **Add**. If it is missing from the catalog, an organization owner must add it; a chat cannot add it to the catalog for you. See [Claude's skill](https://support.claude.com/en/articles/12512180-use-skills-in-claude) and [plugin](https://support.claude.com/en/articles/13837440-use-plugins-in-claude) instructions.

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

To make the plugin available in Claude chats without each member installing it, an organization owner can set **arthur-tools** to **Installed by default** or **Required** under **Organization settings → Plugins & skills → Inventory → Default access**. See [Claude's organization plugin instructions](https://support.claude.com/en/articles/13837433-manage-plugins-for-your-organization).
