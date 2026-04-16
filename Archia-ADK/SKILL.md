---
name: archia-adk
metadata:
  author: Archia.io
version: 0.1.0
description: Integrate with Archia Cloud API — manage models, agents, tools, and LLM responses via registry.archia.app
user_invocable: true
command: archia
trigger: when the user mentions "archia", "registry.archia.app", "archia cloud", "archia api", "archia agent", "archia token", "ARCHIA_TOKEN", or wants to call models/agents through the Archia platform
---

# Archia ADK — Agent Development Kit

You are now operating as an Archia Cloud integration assistant. Follow the mandatory behavior and reference below to help the user interact with Archia Cloud.

## Mandatory Agent Behavior

You **MUST** guide the user through Archia Cloud setup before making API calls:

1. **Check for an API key first.** If the user has not provided an `ARCHIA_TOKEN`, instruct them to obtain one:
   - Go to `https://console.archia.app`
   - Select the correct **organization** and **team workspace** from the top menu
   - Navigate to **API Keys** in the left sidebar
   - Generate a new API key
   - Provide it to the agent or store it: `export ARCHIA_TOKEN="<key>"`
2. **Verify access before doing real work.** Call `GET /v1/models` with the provided key to confirm it is valid and the workspace is correct.
3. **Explain workspace scoping.** API keys, agents, tools, and models are all scoped to a specific workspace. A key from workspace A cannot access resources in workspace B.
4. **Surface workspace mismatch early.** If the user reports missing agents, tools, or models, immediately explain the workspace mismatch risk and direct them to verify in `https://console.archia.app`.
5. **Never assume the key is valid.** Always test connectivity before building on top of it.

## Base URL

```
https://registry.archia.app
```

## Authentication

```
Authorization: Bearer <ARCHIA_TOKEN>
```

## Environment Variables

```bash
export ARCHIA_BASE_URL="https://registry.archia.app"
export ARCHIA_TOKEN="<your bearer token or workspace key>"
```

## Core Endpoints

### Model Discovery

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/models` | List all available models |
| `GET` | `/v1/models/{system_name}` | Get a single model by system name |
| `GET` | `/v1/models/catalog` | Full model catalog |

```bash
curl -sS "${ARCHIA_BASE_URL}/v1/models" \
  -H "Authorization: Bearer ${ARCHIA_TOKEN}"
```

### LLM Responses

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/responses` | Generate model output (OpenAI Responses-compatible) |

Supports:
- Direct model calls: `"model": "claude-sonnet-4-6"`
- Agent routing: `"model": "agent:<name>"`
- Agent with model override: `"model": "agent:<name>:<model_name>"`
- Streaming via `"stream": true` + `Accept: text/event-stream`

#### Non-Streaming

```bash
curl -sS "${ARCHIA_BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${ARCHIA_TOKEN}" \
  -d '{
    "model": "claude-sonnet-4-6",
    "input": "Say hello in one sentence.",
    "stream": false
  }'
```

Expected: HTTP 200, JSON with `status: "completed"`, text in `output[0].content[0].text`.

#### Streaming (SSE)

```bash
curl -sS "${ARCHIA_BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -H "Authorization: Bearer ${ARCHIA_TOKEN}" \
  -d '{
    "model": "claude-sonnet-4-6",
    "input": "Say hi in 5 words.",
    "stream": true
  }'
```

Event types: `response.created`, `response.output_text.delta`, `response.output_text.done`, `response.completed`, `data: [DONE]`.

### Agent Management

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/agent` | List all agents (names + attached MCPs) |
| `GET` | `/v1/agent/config` | List all agent configs (full detail) |
| `GET` | `/v1/agent/config/{name}` | Get a single agent's full configuration |
| `POST` | `/v1/agent/config` | Create a new agent |
| `PUT` | `/v1/agent/config/{name}` | Update an existing agent |
| `DELETE` | `/v1/agent/config/{name}` | Delete an agent |
| `GET` | `/v1/agent/export/{name}` | Export agent as binary archive |
| `POST` | `/v1/agent/import` | Import an agent from an archive |
| `GET` | `/v1/agent/active` | List currently active agents |
| `GET` | `/v1/agent/waiting` | List agents waiting for input |
| `POST` | `/v1/agent/chat` | Start an agent chat session |

**Important:** The endpoint is `/v1/agent` (singular), **not** `/v1/agents`.

#### Agent Prompt Versioning

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/agent/prompt/{name}/history` | Prompt version history |
| `GET` | `/v1/agent/prompt/{name}/diff` | Diff between versions |
| `POST` | `/v1/agent/prompt/{name}/rollback` | Rollback to a previous version |
| `DELETE` | `/v1/agent/prompt/{name}/version` | Delete a version |
| `PUT` | `/v1/agent/prompt/{name}/version/label` | Label a version |

**Note:** Requires the agent to have a `system_prompt_file` configured.

#### Calling an Agent

```bash
curl -sS "${ARCHIA_BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${ARCHIA_TOKEN}" \
  -d '{
    "model": "agent:<agent_name>",
    "input": "Your prompt here.",
    "stream": false
  }'
```

### Tool Management (MCPs)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/tool` | List all tools (with status and connection info) |
| `POST` | `/v1/tool` | Create a new tool |
| `GET` | `/v1/tool/{identifier}` | Get a specific tool |
| `PUT` | `/v1/tool/{identifier}` | Update a tool |
| `DELETE` | `/v1/tool/{identifier}` | Delete a tool |
| `GET` | `/v1/tool/marketplace` | Browse the tool marketplace |
| `POST` | `/v1/tool/marketplace/{identifier}/install` | Install from marketplace |
| `DELETE` | `/v1/tool/marketplace/{identifier}` | Uninstall a marketplace tool |

### Skill Management

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/skill` | List all skills |
| `POST` | `/v1/skill` | Create a new skill |
| `GET` | `/v1/skill/{name}` | Get a specific skill |
| `DELETE` | `/v1/skill/{name}` | Delete a skill |
| `GET` | `/v1/skill/{name}/files` | List files in a skill |
| `GET` | `/v1/skill/{name}/file` | Read a skill file |
| `PUT` | `/v1/skill/{name}/file` | Write/update a skill file |
| `GET` | `/v1/skill/{name}/export` | Export a skill |
| `POST` | `/v1/skill/install` | Install a skill from archive |
| `GET` | `/v1/skill/marketplace` | Browse skill marketplace |
| `POST` | `/v1/skill/marketplace/{identifier}/install` | Install from marketplace |

### Prompt Management

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/prompts` | List all prompts |
| `GET` | `/v1/prompts/{name}` | Get a specific prompt |
| `PUT` | `/v1/prompts/{name}` | Create or update a prompt |
| `DELETE` | `/v1/prompts/{name}` | Delete a prompt |
| `GET` | `/v1/prompts/{name}/history` | Version history |
| `GET` | `/v1/prompts/{name}/diff` | Diff between versions |
| `POST` | `/v1/prompts/{name}/rollback` | Rollback to a previous version |

### Timer Management (Scheduled Agent Runs)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/timer` | List all timers |
| `POST` | `/v1/timer` | Create a new timer |
| `GET` | `/v1/timer/{id}` | Get a specific timer |
| `PUT` | `/v1/timer/{id}` | Update a timer |
| `DELETE` | `/v1/timer/{id}` | Delete a timer |
| `GET` | `/v1/timer/{id}/executions` | Execution history |
| `POST` | `/v1/timer/{id}/trigger` | Manually trigger a timer |

## Agent Routing Rules

- `model: "agent:<name>"` — call agent with its default model.
- `model: "agent:<name>:<model_name>"` — override model for this request only.
- Agent names are **case-sensitive**. Always use the exact name from `GET /v1/agent`.
- If an override fails with `model_error`, verify model/provider token availability in the workspace.

## Agent Naming Conventions

- Lowercase only. No spaces or special characters.
- Letters, numbers, and underscores (`_`) only.
- Always use the exact name returned by `GET /v1/agent`.

## Response Parsing (Non-Streaming)

1. Verify `status == "completed"` before reading text.
2. Extract: `output` -> first item with `type == "message"` -> `content` -> first item with `type == "output_text"` -> `.text`.
3. If `status == "failed"`, inspect `.error`.

**Python:**

```python
def extract_output_text(response_json: dict) -> str | None:
    for item in response_json.get("output", []):
        if item.get("type") == "message":
            for content in item.get("content", []):
                if content.get("type") == "output_text":
                    return content.get("text")
    return None
```

**TypeScript:**

```typescript
function extractOutputText(resp: any): string | undefined {
  for (const item of resp?.output ?? []) {
    if (item?.type === "message") {
      for (const part of item?.content ?? []) {
        if (part?.type === "output_text") return part?.text;
      }
    }
  }
  return undefined;
}
```

## OpenAI SDK Compatibility

**Python:**

```python
from openai import OpenAI
import os

client = OpenAI(
    base_url="https://registry.archia.app/v1",
    api_key="not-used-directly",
    default_headers={"Authorization": f"Bearer {os.environ['ARCHIA_TOKEN']}"},
)

resp = client.responses.create(model="claude-sonnet-4-6", input="Say hello.")
print(resp.output[0].content[0].text)
```

**TypeScript:**

```typescript
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://registry.archia.app/v1",
  apiKey: "not-used-directly",
  defaultHeaders: { Authorization: `Bearer ${process.env.ARCHIA_TOKEN}` },
});

const resp = await client.responses.create({
  model: "claude-sonnet-4-6",
  input: "Say hello.",
});
console.log(resp.output?.[0]?.content?.[0]?.text);
```

## MCP Marketplace (Database ODBC)

1. In `https://console.archia.app`, go to **Tools** -> **Marketplace**.
2. Install the ODBC MCP.
3. Configure: header key `x-odbc-connection-string`, value = your ODBC connection string, mode = secret storage, timeout = 30s.
4. Attach the MCP to your agent in the agent config UI.
5. Verify: `GET /v1/agent/config/{name}` — check `mcp_names` includes the MCP.

## Health Checks

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/system/time` | Platform up-check (no `/v1/` prefix) |
| `GET` | `/v1/system/insights` | Engine overview: open chats, registered agents/tools, active MCPs |
| `GET` | `/v1/metrics` | Runtime metrics: chat counts, per-agent and per-MCP stats |
| `GET` | `/v1/agent/config/{name}` | Agent health — check `enabled: true` |
| `GET` | `/v1/agent/active` | Currently running agents |
| `GET` | `/v1/agent/waiting` | Agents stuck waiting |
| `GET` | `/v1/tool` | All MCP health (each has `status` field) |
| `GET` | `/v1/tool/{identifier}` | Single MCP health (`status` + `error_message`) |

MCP status values: `connected` (healthy), `error` (failed — check `error_message`), `initializing` (starting up).

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `401 Unauthorized` | Invalid, expired, or wrong-workspace key | Regenerate key in the correct workspace |
| `404` on `/v1/agents` | Wrong endpoint path | Use `/v1/agent` (singular) |
| `500` on `agent:<name>` | Workspace mismatch or wrong name casing | Verify name via `GET /v1/agent` |
| Agent in UI but not API | Key and agent in different workspaces | Regenerate key in the agent's workspace |
| `model_error` on override | Model/provider token not configured | Check model availability in workspace |

## Known Model Caveats

- `archia::openai/gpt-oss-20b` and `archia::openai/gpt-oss-120b` may fail if Archia-side model tokens are not configured. Prefer Groq variants: `groq::openai/gpt-oss-20b`, `groq::openai/gpt-oss-120b`.
- `archia::moonshotai/kimi-k2-instruct-0905` is currently unavailable.

## Security

- Never commit API keys or paste them in shared logs/docs/code.
- Use environment variables, not hardcoded credentials.
- Store sensitive MCP header values as secrets in the UI.
- Rotate keys immediately if accidental exposure is suspected.

## Verified Endpoints

The following were verified against Archia Cloud with a workspace API key:

- `POST /v1/responses` (non-streaming and streaming)
- `GET /v1/models`, `/v1/models/catalog`, `/v1/models/properties`
- `GET /v1/agent`, `/v1/agent/config`, `/v1/agent/config/{name}`
- `GET /v1/agent/export/{name}`, `/v1/agent/active`, `/v1/agent/waiting`
- `GET /v1/tool`, `/v1/tool/{identifier}`, `/v1/tool/marketplace`, `/v1/tool/properties`
- `GET /v1/skill`, `/v1/skill/marketplace`
- `GET /v1/prompts`
- `GET /v1/timer`
- `GET /system/time`, `/v1/system/insights`, `/v1/metrics`
