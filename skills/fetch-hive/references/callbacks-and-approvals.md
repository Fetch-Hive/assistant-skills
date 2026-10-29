# Callbacks, webhook triggers and tool approvals

## Three separate things

- **API invoke** starts a run with the workspace API key (`POST /v1/workflow/invoke`, `POST /v1/agent/invoke`).
- **Webhook trigger** lets another service start a deployed workflow or an agent without an API key, using the deployment's or agent's webhook secret in `X-Fetch-Hive-Webhook-Secret`.
- **Callback delivery** returns at once and later POSTs a signed event with the result to your `callback_url`.

The same secret is used both ways: a trigger sends it to prove it may start the run, and Fetch Hive signs callbacks with it. Keep it in an environment variable such as `FETCH_HIVE_WEBHOOK_SECRET`. The person copies it from **Get Code** (Signing secret) on the workflow deployment or agent; never ask for it in chat.

## Callback delivery for a workflow

```typescript
const run = await client.invokeWorkflow({
  deployment: 'DEPLOYMENT_NAME',
  variant: 'VARIANT_NAME',
  inputs: { topic },
  async: { enabled: true, callback_url: `${process.env.APP_URL}/api/fetch-hive/callback` },
});
// run.status is "running" or "queued"; store run.request_id against your own record
```

Python: `client.invoke_workflow(..., async_mode=True, callback_url="https://...")`.

Use callback delivery when the workflow may run longer than a normal request, and always when it has a Human in the Loop step. Save the returned `request_id` with your own record so the callback can find it.

## Callback events

Every callback has the same envelope:

```json
{ "event_type": "workflow.completed", "request_id": "req_...", "data": { "output": "..." } }
```

- Workflows: `workflow.completed` (`data.output`), `workflow.failed` (`data.error`), `workflow.cancelled` (`data.error`, `data.reason: "cancelled"`). All three are final.
- Agents: `agent.completed` (`data.response`, and `data.output_json` when a `response_format` was sent), `agent.failed` (`data.error`), `agent.awaiting_approval` (`pending_tool_approvals`, `review_url`). After an approval, a later `agent.completed` carries `resumed_from_run_id`.

Make the handler idempotent: a delivery can be retried, so look the record up by `request_id` and ignore an event you've already applied. Answer quickly with a 2xx and do slow work after.

## Verifying the signature

Each callback has `X-Fetch-Hive-Signature`, `X-Fetch-Hive-Timestamp` and `X-Fetch-Hive-Webhook-Id`. The signature is the hex HMAC SHA256 of `{timestamp}.{raw request body}` with the webhook secret. Verify against the raw body exactly as received (not re-serialised JSON), reject timestamps more than 5 minutes old, and compare in constant time. Treat anything that fails as untrusted.

### Next.js route handler

```typescript
// app/api/fetch-hive/callback/route.ts
import crypto from 'node:crypto';

export async function POST(request: Request) {
  const secret = process.env.FETCH_HIVE_WEBHOOK_SECRET;
  const signature = request.headers.get('x-fetch-hive-signature') ?? '';
  const timestamp = request.headers.get('x-fetch-hive-timestamp') ?? '';
  const raw = await request.text();

  if (!secret || !signature || !timestamp) return new Response('unauthorised', { status: 401 });
  if (Math.abs(Date.now() / 1000 - Number(timestamp)) > 300) return new Response('stale', { status: 401 });

  const expected = crypto.createHmac('sha256', secret).update(`${timestamp}.${raw}`).digest('hex');
  const valid =
    expected.length === signature.length &&
    crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
  if (!valid) return new Response('bad signature', { status: 401 });

  const event = JSON.parse(raw) as { event_type: string; request_id: string; data: Record<string, unknown> };
  switch (event.event_type) {
    case 'workflow.completed':
      // save event.data.output against the record with event.request_id
      break;
    case 'workflow.failed':
    case 'workflow.cancelled':
      // mark the record failed with event.data.error
      break;
  }
  return new Response(null, { status: 204 });
}
```

### Python (FastAPI)

```python
import hashlib, hmac, json, os, time
from fastapi import FastAPI, HTTPException, Request

app = FastAPI()

@app.post("/fetch-hive/callback")
async def fetch_hive_callback(request: Request):
    secret = os.environ["FETCH_HIVE_WEBHOOK_SECRET"]
    signature = request.headers.get("x-fetch-hive-signature", "")
    timestamp = request.headers.get("x-fetch-hive-timestamp", "")
    raw = await request.body()

    if not signature or not timestamp or abs(time.time() - int(timestamp)) > 300:
        raise HTTPException(status_code=401)
    expected = hmac.new(secret.encode(), f"{timestamp}.".encode() + raw, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, signature):
        raise HTTPException(status_code=401)

    event = json.loads(raw)
    # handle event["event_type"], keyed by event["request_id"]
    return {"ok": True}
```

## Webhook triggers

Workflow: `POST https://api.fetchhive.com/v1/workflow/webhooks/{endpoint_id}/variants/{variant_id}`. Agent: `POST https://api.fetchhive.com/v1/agent/webhooks/{agent_id}`. Get the exact URL from `get_integration_code` or **Get Code** → **Webhook trigger**.

```bash
curl "$FETCH_HIVE_TRIGGER_URL" \
  -H "X-Fetch-Hive-Webhook-Secret: $FETCH_HIVE_WEBHOOK_SECRET" \
  -H 'Content-Type: application/json' \
  -d '{"inputs":{"topic":"State of enterprise AI in 2026"},"async":{"callback_url":"https://example.com/callback"}}'
```

Triggers always need `async.callback_url` and always deliver the result by callback. For a workflow, a body without `inputs` is treated as the inputs. Optional `user` and `metadata` sit beside `inputs`. An agent trigger needs `message` and can take `thread_id` and `response_format`. If the sending service can't set headers, `?secret=` works as a fallback; prefer the header.

## Tool approvals in the person's app

Agent actions that write to apps, and destructive actions, can pause the run until someone approves them.

- A non-streaming agent response then has `status: "awaiting_approval"` and `pending_tool_approvals` (each with `approval_id`, `tool_name`, `summary`, `arguments`, `expires_at` and `review_url`). A streaming call sends a `tool_approval` event and the same fields on the final `usage` event.
- Show the person in your app what is waiting. Decide with the API key: `POST /v1/public/workspaces/{workspace_id}/tool_action_approvals/{id}/decide` with `decision` `allow_once` or `deny` (SDK: `decideToolActionApproval` / `decide_tool_action_approval`), or send them to the `review_url`.
- After the decision the run continues on the server by itself. Don't re-invoke the thread; poll or wait for the callback (`agent.completed` with `resumed_from_run_id`).
- "Always allow" can only be set on the dashboard. A second decision returns `409`; an expired approval returns `410`.
- A trusted server can skip approvals for one call with `tool_approval: "auto_approve"`, only if a workspace owner or admin allowed that API key to auto-approve (**Settings → API Keys**). Destructive actions still ask, and Deny rules still block. Don't turn this on for the person; tell them where it is if they want it.
