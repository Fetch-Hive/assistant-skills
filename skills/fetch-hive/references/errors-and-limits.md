# Errors and limits

## Reading an API error

Errors are JSON with a human-readable `message`, a stable `error_code`, and `error` (same text, kept for older clients). Branch on `error_code`, not on the message text.

```json
{ "error": "Invalid access.", "message": "Invalid access.", "error_code": "invalid_api_key" }
```

| Status | Common `error_code` | What to do |
| --- | --- | --- |
| 400 / 422 | `validation_error`, `invalid_request` | Fix the request: missing `deployment` or `variant`, wrong input types, nested `metadata`, `async.enabled` without a valid `callback_url`, or a direct call to a workflow with a Human in the Loop step |
| 401 | `invalid_api_key`, `unauthorized` | `FETCH_HIVE_API_KEY` is missing, wrong or revoked. Ask the person to check it in **Settings → API Keys**; don't ask for the value |
| 402 | `insufficient_hosted_llm_credits`, plan limits | The workspace is out of credits or over a plan limit. The person tops up or upgrades in Billing |
| 403 | `forbidden`, `tool_approval_override_not_permitted` | The key isn't allowed to do this. For auto-approve, an owner or admin must allow the key, or drop the field |
| 404 | `not_found` | The deployment, variant, agent or run doesn't exist in this workspace. Check the names with `get_integration_code` |
| 409 / 410 | | Tool approval already decided / expired |
| 422 | | Also "You've reached your credit cap." |
| 429 | `api_rate_limit_exceeded`, `concurrency_limit_reached`, `too_many_requests` | Back off and retry later. For many runs at once, use callback delivery |
| 502 | | A model provider failed. Retrying may help |
| 504 | | A direct workflow call ran past the wait window and was cancelled. Use callback delivery |

## Run outcomes

A request can succeed and the run still fail. Check `run_status` (workflows) or `status` (agents):

- `completed`: use `output` / `response`.
- `failed`: show a friendly message and log the `request_id`; the person can open the run in **Logs** in the dashboard, or ask the assistant to `diagnose_run`.
- `cancelled`: final, someone stopped it.
- `running` / `queued`: only with callback delivery; wait for the callback.
- `awaiting_approval` (agents): see [callbacks-and-approvals.md](callbacks-and-approvals.md).

Don't retry a failed paid run in a loop. Retry only errors that say retrying may help (429, 502), with backoff and a small cap.

## In the MCP

- Paid tools answer with a quote and `needs_approval` until called with `approved: true`. That's not an error: show the cost and ask.
- "Too many requests": each connection has per-minute and per-day limits on requests, runs and builds. Wait a minute and try again; tell the person if it keeps happening.
- A test run is stopped if it goes over the cost limit it quoted or runs too long; `get_test_run` says why.
- `run_qa` has a daily limit per account.
- If a run fails, `diagnose_run` gives the likely cause (credits, app connection, permissions, rate limits, timeout, inputs, output format, approval, provider error) and whether trying again may help.

## Costs

Connecting an assistant, and finding or changing workflows and agents, uses no credits. Runs, test runs, retries, scheduled runs and share page runs use workspace credits, the same as runs started on the dashboard. `estimate_cost` gives the usual cost before a run.
