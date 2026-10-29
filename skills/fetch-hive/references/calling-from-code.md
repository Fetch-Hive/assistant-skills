# Calling Fetch Hive from the person's code

Base URL: `https://api.fetchhive.com/v1`. Every request sends the workspace API key as `Authorization: Bearer <key>`, read from the `FETCH_HIVE_API_KEY` environment variable.

## Before you write code

- Get the real names from `get_integration_code` (MCP) or from **Get Code** on the dashboard: a workflow's **deployment** and **variant**, or an agent's ID. Don't invent them.
- A workflow must be deployed. If it isn't, deploy it (`deploy_workflow`, with the person's yes) or tell them to deploy it in the editor.
- Inputs are the workflow's **Start** step inputs, with matching types. Array inputs are real JSON arrays.

## Keys and where the code runs

- Call Fetch Hive from server code only: an API route, server action, backend job or worker. Never from browser code, a mobile app bundle or anything a visitor can read.
- Read the key from `FETCH_HIVE_API_KEY`. Add `FETCH_HIVE_API_KEY=` (empty) to `.env.example` and tell the person to create a key in **Settings → API Keys** and set it in their environment and hosting provider. Don't ask them to paste it to you.
- Make sure `.env` files with real values are in `.gitignore`.

## Workflows

`POST /v1/workflow/invoke`

| Field | Required | Notes |
| --- | --- | --- |
| `deployment` | yes | Deployment name |
| `variant` | yes | Variant name |
| `inputs` | no | Start step inputs |
| `async` | no | `{ "enabled": true, "callback_url": "https://..." }` for callback delivery |
| `metadata` | no | Flat object of strings, numbers, booleans or null, for log filtering. Not workflow input |
| `tool_approval` | no | `"policy"` (default) or `"auto_approve"` when the key is allowed to auto-approve |

Without `async`, the call waits for the workflow and returns `{ request_id, run_status, output }`. If it runs past the wait window it is cancelled and returns `504` with `run_status: "failed"`. Use callback delivery for long workflows, and always for workflows with a Human in the Loop step (a direct call to those returns a validation error).

### TypeScript (`@fetch-hive/sdk`, Node 18+)

```bash
npm install @fetch-hive/sdk
```

```typescript
import { FetchHive } from '@fetch-hive/sdk';

const client = new FetchHive(); // reads FETCH_HIVE_API_KEY

const run = await client.invokeWorkflow({
  deployment: 'DEPLOYMENT_NAME',
  variant: 'VARIANT_NAME',
  inputs: { topic: 'State of enterprise AI in 2026' },
});

if (run.status !== 'completed') throw new Error(`Workflow ${run.request_id} ended ${run.status}`);
console.log(run.output);
```

### Next.js route handler

```typescript
// app/api/report/route.ts
import { FetchHive } from '@fetch-hive/sdk';

const client = new FetchHive();

export async function POST(request: Request) {
  const { topic } = await request.json();
  if (typeof topic !== 'string' || !topic.trim()) {
    return Response.json({ error: 'topic is required' }, { status: 400 });
  }

  const run = await client.invokeWorkflow({
    deployment: 'DEPLOYMENT_NAME',
    variant: 'VARIANT_NAME',
    inputs: { topic },
  });

  if (run.status !== 'completed') {
    return Response.json({ error: 'The report could not be generated', request_id: run.request_id }, { status: 502 });
  }
  return Response.json({ report: run.output });
}
```

Check the person's own auth and rate limits on this route: every call runs the workflow on their credits.

### Python (`fetch-hive-sdk`, Python 3.9+)

```bash
pip install fetch-hive-sdk
```

```python
from fetch_hive_sdk import FetchHive

client = FetchHive()  # reads FETCH_HIVE_API_KEY

run = client.invoke_workflow(
    deployment="DEPLOYMENT_NAME",
    variant="VARIANT_NAME",
    inputs={"topic": "State of enterprise AI in 2026"},
)
if run["run_status"] != "completed":
    raise RuntimeError(f"Workflow {run['request_id']} ended {run['run_status']}")
print(run["output"])
```

The Python SDK's default timeout is 120 seconds (`FetchHive(timeout=...)`). Prefer callback delivery to a longer timeout.

### curl

```bash
curl https://api.fetchhive.com/v1/workflow/invoke \
  -H "Authorization: Bearer $FETCH_HIVE_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"deployment":"DEPLOYMENT_NAME","variant":"VARIANT_NAME","inputs":{"topic":"State of enterprise AI in 2026"}}'
```

Other languages: Ruby and PHP SDKs are in the docs under Workflows → Run with Ruby SDK / Run with PHP SDK.

## Agents

`POST /v1/agent/invoke`

| Field | Required | Notes |
| --- | --- | --- |
| `agent` | yes | Agent ID |
| `message` | yes | The message |
| `streaming` | no | `true` for server-sent events |
| `thread_id` | no | Any string. Fetch Hive keeps the conversation and resumes it on the next call with the same value |
| `messages` | no | Earlier turns you manage yourself, `{ role, content }`. Not stored |
| `attachments` | no | Up to five `https://` file URLs (CSV, XLSX, PDF, DOCX, text, images) |
| `response_format` | no | A JSON schema for this call's answer (below) |
| `metadata`, `tool_approval` | no | As for workflows |
| `async` | no | `{ "callback_url": "https://..." }` for background work |

Use a stable `thread_id` per conversation, such as your user or ticket ID, rather than sending the whole history each time.

To get JSON back instead of text, send `response_format` on the call (it applies to that call only):

```json
{
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "competitors",
      "strict": true,
      "schema": {
        "type": "object",
        "properties": { "competitors": { "type": "array", "items": { "type": "string" } } },
        "required": ["competitors"],
        "additionalProperties": false
      }
    }
  }
}
```

### TypeScript

```typescript
import { FetchHive } from '@fetch-hive/sdk';

const client = new FetchHive();

const reply = await client.invokeAgent({
  agent: 'AGENT_ID',
  message: 'Summarise this week’s support tickets',
  thread_id: `support-${userId}`,
});

if (reply.status === 'awaiting_approval') {
  // see callbacks-and-approvals.md
}
console.log(reply.response);
```

Streaming to the browser from a server route:

```typescript
// app/api/chat/route.ts
import { FetchHive } from '@fetch-hive/sdk';

const client = new FetchHive();

export async function POST(request: Request) {
  const { message, threadId } = await request.json();
  const encoder = new TextEncoder();

  const stream = new ReadableStream({
    async start(controller) {
      try {
        for await (const chunk of client.invokeAgentStream({ agent: 'AGENT_ID', message, thread_id: threadId })) {
          if (chunk.type === 'response' && chunk.response) controller.enqueue(encoder.encode(chunk.response));
          if (chunk.type === 'error') controller.enqueue(encoder.encode('\n[The agent hit an error. Please try again.]'));
        }
      } finally {
        controller.close();
      }
    },
  });

  return new Response(stream, { headers: { 'Content-Type': 'text/plain; charset=utf-8' } });
}
```

Stream events, in order when present: `summary`, `reasoning`, `response`, `tool`, `tool_approval`, then a final `usage` (or `error`).

### Python

```python
from fetch_hive_sdk import FetchHive

client = FetchHive()

reply = client.invoke_agent(agent="AGENT_ID", message="Summarise this week's support tickets", thread_id="support-42")
print(reply["response"])

for chunk in client.invoke_agent_stream(agent="AGENT_ID", message="And last week?", thread_id="support-42"):
    if chunk.get("type") == "response":
        print(chunk.get("response", ""), end="", flush=True)
```

### curl

```bash
curl https://api.fetchhive.com/v1/agent/invoke \
  -H "Authorization: Bearer $FETCH_HIVE_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"agent":"AGENT_ID","message":"Summarise this week’s support tickets","streaming":false}'
```

## Tests to add

Mock the SDK client or the HTTP call in the person's tests; don't call Fetch Hive from a test suite, since every call is a paid run. Test the inputs you send, the handling of `failed` and `awaiting_approval`, and the missing-key case.
