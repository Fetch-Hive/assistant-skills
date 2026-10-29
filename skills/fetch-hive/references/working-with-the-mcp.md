# Working in a workspace with the Fetch Hive MCP

The connector works in one workspace, chosen when the person connected it. Start with `whoami` to see which workspace and account you are in, the person's role (and so what they can change), and how many assistant calls are left today.

## Spend and plan limits

- `get_usage` shows what runs cost over `today`, `7d`, `30d` (default) or `billing_period`, by workflow and by agent, plus the tasks and hosted model wallet left. Costs are the same total as **Dashboard → Analytics**. Use it for "what am I spending?" and "which workflow costs most?".
- `get_plan_limits` lists what the plan allows (steps per workflow, iteration size, runs at once, workspaces, team members, storage and more) and what is in use. When something is refused for a plan limit, check here and explain the limit instead of retrying. Plan changes happen on the billing page; the tool gives the link.

## Find before you build

1. `search_workflows` and `search_agents` for what the person already has. Open one with `get_workflow` or `get_agent` before running or changing it.
2. `search_templates` for the template library and the account's own templates. `get_template` explains a template and its inputs; `use_template` copies it into the workspace, the same as the **Use template** button.
3. Only when nothing fits, `build` makes a new workflow or agent from the person's request and saves it as a draft.
   - If `build` asks questions, put them to the person and pass their answers back. Don't answer for them.
   - A build can take a couple of minutes. Check it with `get_build` and tell the person when it's ready.

Reuse a template when it does most of the job and the rest is a change of inputs or a step. Build when the request needs a different chain of steps or apps.

Choose a **workflow** for a fixed sequence that runs the same way each time (reports, enrichment, scheduled jobs, bulk runs). Choose an **agent** for a conversation or a task where the model decides which tools to call.

## Apps and connections

Check a workflow's or agent's apps with `list_connections`. If one is missing, `connect_app` returns a link; the person connects the app on the dashboard. Never ask for an app password or token in chat.

## Changing a workflow or agent

- `update_workflow` renames a workflow, changes its description, or adds and removes start inputs.
- `get_step`, `add_step`, `edit_step` and `remove_step` change steps. `list_step_types` and `get_step_type` give each step's settings and the placeholder rules for using earlier outputs.
- Every step change is saved as a version. `list_versions` and `restore_version` undo a change; `duplicate_workflow` makes a copy first if the person wants to experiment.
- `update_agent`, `set_agent_tools` (keys from `list_agent_tools`) and `set_agent_knowledge` change an agent. They return the previous values so you can undo.
- A step can't be removed while another step uses its output. Change the other step first.
- Assistants can't add app actions that delete data. The person adds those in the editor.

## Testing

- `test_run_draft` runs a workflow as it stands with sample data for steps that read from apps, and with sends and review steps switched off, so nothing reaches anyone. It is paid: quote first, then `approved: true` after the person's yes. It is stopped if it goes over the cost limit it quotes.
- `get_test_run` shows each step's result, the step that failed and the cost. `list_test_runs` lists recent test runs of a workflow.
- `run_qa` scores a finished test run out of 100 (80 passes) and says what to fix. Read the result with `get_test_run`.

## Running

- `estimate_cost` gives what a workflow or agent run usually costs.
- `run_workflow` runs the current version once, like the editor's **Run** button. `run_agent` sends a message to an agent, optionally with a `response_format` JSON schema for this call. Both quote without `approved` and run with `approved: true`.
- Poll `get_run` until `done` is true, then give the person the output. `list_runs` lists recent runs.
- When a run fails, `diagnose_run` says why and whether trying again may help. `retry_run` repeats it (paid, so ask). `cancel_run` stops a workflow run that is still going.
- `list_run_files` gives download links for files a run made.

## Deploying, scheduling and sharing

- `deploy_workflow` publishes the current version so the API, webhooks and schedules use it. Ask first.
- `list_schedules`, `set_schedule`, `pause_schedule`, `resume_schedule`. Setting or resuming a schedule needs the person's yes; it shows when it will run and the cost per run. Pausing happens at once.
- `share_workflow` makes a public link anyone can run, billed to the account. Show the person the title, description and input guidance first, and publish only after they approve. `update_share` changes one; `run_share` checks it works.

## Calling it from the person's app

`get_integration_code` with `workflow_id` (deployed) or `agent_id` returns ready-to-paste curl, TypeScript and Python with the real names and inputs. See [calling-from-code.md](calling-from-code.md).

## Approvals

`list_approvals` shows app actions waiting for a person. Show the exact action, then call `decide_approval` with the person's own answer: `allow_once` or `deny`. To always allow an action, give the person the `review_url`.

## MCP gateways

A gateway turns an OpenAPI spec into a hosted MCP server: `create_gateway`, then `list_gateway_issues` and show the issues and suggested fixes, `fix_gateway_issues` or `update_gateway_tools` with the person's OK, `publish_gateway` with their yes, then `test_gateway_tool`. The API's credentials and MCP client tokens are entered on the dashboard link, never in chat. Only account owners and admins can change gateways.

## When unsure

`search_docs` searches the Fetch Hive docs.
