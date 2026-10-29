# Fetch Hive for AI assistants

Teach your AI assistant how to work with [Fetch Hive](https://fetchhive.com): find, build, test, run and schedule workflows and agents, and add them to your own app with working code and no secrets in it.

There are two parts:

- **The Fetch Hive connector** (MCP server at `https://connect.fetchhive.com/mcp`) does the work in your workspace.
- **The Fetch Hive skill** in this repo is the know-how: when to reuse a template, asking you before anything that costs credits, calling workflows and agents from code, handling callbacks and approvals, and what errors mean.

## Claude Code

```text
/plugin marketplace add Fetch-Hive/assistant-skills
/plugin install fetch-hive@fetch-hive
```

This installs the skill and adds the Fetch Hive connector. The first time Claude Code uses the connector, it opens Fetch Hive in your browser: sign in, choose the workspace and click **Allow**.

To add only the skill, copy `skills/fetch-hive` into `~/.claude/skills/` (all projects) or `.claude/skills/` in one project.

## Claude (web and Desktop)

1. Download [fetch-hive.zip](https://github.com/Fetch-Hive/assistant-skills/raw/main/fetch-hive.zip).
2. In Claude, open **Settings → Capabilities**, and under **Skills** upload the zip.
3. Connect Fetch Hive as a connector if you haven't: **Settings → Connectors → Add custom connector**, with the URL `https://connect.fetchhive.com/mcp`. See [Connect an AI assistant](https://docs.fetchhive.com/workspace/connect-assistants).

## Cursor

1. Copy [`cursor/fetch-hive.mdc`](cursor/fetch-hive.mdc) into your project's `.cursor/rules/` folder.
2. Add the connector to `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "fetch-hive": { "url": "https://connect.fetchhive.com/mcp" }
  }
}
```

## ChatGPT and other assistants

ChatGPT has no skills. Connect Fetch Hive as a connector and ask it to "get the code to call this workflow from my app"; the connector's `get_integration_code` tool gives ready-to-paste code. Other assistants can use `cursor/fetch-hive.mdc` as plain instructions (it's Markdown with a short header).

## Try it

- "Which Fetch Hive workflows do I have?"
- "Find a template that summarises Hacker News posts about AI, test it, and tell me what it costs."
- "Add my lead enrichment workflow to this Next.js app, with a callback route that checks the signature."

Your assistant asks before anything that costs credits or acts in your connected apps. API keys stay in your environment (`FETCH_HIVE_API_KEY`); never paste one into a chat.

## Contributing

Edit the Markdown in `skills/fetch-hive/`, then run `scripts/build` to rebuild `fetch-hive.zip` and `cursor/fetch-hive.mdc`, and commit all of it together. Keep the content in step with the [Fetch Hive docs](https://docs.fetchhive.com).

Privacy: [fetchhive.com/privacy](https://fetchhive.com/privacy). License: MIT.
