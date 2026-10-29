---
name: fetch-hive
description: Use when the person works with Fetch Hive workflows or agents, either through the Fetch Hive MCP connector (find, build, change, test, run, schedule, share) or in their own code (call a workflow or agent from an app, handle callbacks, webhooks and tool approvals). Covers when to reuse a template, asking before anything paid or public, and adding a Fetch Hive call to a codebase with no secrets in it.
---

# Fetch Hive

Fetch Hive runs AI workflows (a chain of steps such as search, AI prompt, Slack or Notion) and agents (a model with tools that answers messages). This skill covers two jobs:

1. **Working in the person's workspace** through the Fetch Hive MCP connector (`https://connect.fetchhive.com/mcp`). Read [references/working-with-the-mcp.md](references/working-with-the-mcp.md).
2. **Calling Fetch Hive from the person's own code**, for example "add this workflow to my Next.js app". Read [references/calling-from-code.md](references/calling-from-code.md), and [references/callbacks-and-approvals.md](references/callbacks-and-approvals.md) when the run is long, uses callbacks or webhooks, or can wait for a tool approval.

When something fails, read [references/errors-and-limits.md](references/errors-and-limits.md).

## Rules that always apply

- **Ask before anything paid or public.** Runs, test runs, retries, deploys, schedules, share pages, prompt runs and knowledge base sources cost credits or act in the outside world, and archiving or a new webhook secret changes what is live. Call the tool without `approved` first: it answers with the cost (and the inputs it would use). Show that to the person, and pass `approved: true` only after they say yes. Never start the same paid run twice without asking.
- **Never handle secrets in chat.** Don't ask the person to paste an API key, a webhook secret, an app password or an MCP client token. Connecting apps and creating keys happen on the dashboard links the tools return.
- **Keys live in environment variables.** Code reads the workspace API key from `FETCH_HIVE_API_KEY`, and a callback signing secret from its own variable such as `FETCH_HIVE_WEBHOOK_SECRET`. Never write a real key into code, a commit, a log line or a client-side bundle. Call Fetch Hive from server code only.
- **Reuse before building.** Look for an existing workflow or agent, then a template, before building something new.
- **Approvals are the person's call.** When a run waits for a tool approval, show the exact action and decide only with the person's own answer (`allow_once` or `deny`). "Always allow" is set by the person on the dashboard.

## From "build me X" to "add X to my app"

1. Find or build it with the MCP: `search_workflows` / `search_agents`, then `search_templates` and `use_template`, else `build`.
2. Check its apps with `list_connections`; `connect_app` gives the person a link for a missing one.
3. Prove it works: `test_run_draft` (sample data, sends switched off; paid, so ask) and `check_test_run` for a score and what to improve; offer to fix it and test again. Or a real `run_workflow` / `run_agent` with the person's yes.
4. A workflow must be deployed before code can call it: `deploy_workflow` (ask first). Agents need no deploy.
5. Call `get_integration_code` with the `workflow_id` or `agent_id`. It returns curl, TypeScript and Python with the real deployment, variant and inputs, and the webhook trigger URL for a workflow. Use it as the source of truth for names; don't guess them.
6. Write the code into the person's project following [references/calling-from-code.md](references/calling-from-code.md): server-side only, key from `FETCH_HIVE_API_KEY`, add the variable to `.env.example` (empty value) and tell the person to set it from **Settings → API Keys**.
7. If the workflow can run longer than a short request, or contains a Human in the Loop step, use callback delivery and verify the signature ([references/callbacks-and-approvals.md](references/callbacks-and-approvals.md)).

## Without the MCP connector

The code paths still work. Ask the person for the deployment name and variant (shown under **Get Code** on the workflow) or the agent ID (shown under **Get Code** in the agent editor), never for the key itself.

Docs: https://docs.fetchhive.com
