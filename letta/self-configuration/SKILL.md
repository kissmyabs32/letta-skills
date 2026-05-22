---
name: self-configuration
description: Helps Letta agents configure themselves or their current conversation with one API call. Use when changing an agent's own model, context window, system prompt, reasoning effort, model settings, or conversation-scoped model overrides.
license: MIT
---

# Letta self-configuration

Use the Letta API directly when an agent needs to change its own model, context window, system prompt, or reasoning effort.

## When to use this skill

Use this skill for runtime changes to an existing Letta agent or conversation. For provider setup, API keys, server configuration, or choosing a model for a new agent, use the broader Letta configuration/API skills instead.

## Choose the target

- **Agent default:** `PATCH /v1/agents/$AGENT_ID`. Persists across conversations. Use for model defaults and system prompt changes.
- **Current conversation:** `PATCH /v1/conversations/$CONVERSATION_ID`. Affects only this thread. Use for high-frequency changes like a context window slider or temporary model/reasoning changes.

Use `AGENT_ID` for yourself. Use `CONVERSATION_ID` for the current thread when it is available. In Letta Code it is usually injected into the runtime context; in custom apps, track the conversation ID returned by the Conversations API.

```bash
BASE_URL="${LETTA_BASE_URL:-https://api.letta.com}"
: "${LETTA_API_KEY:?Set LETTA_API_KEY}"
: "${AGENT_ID:?Set AGENT_ID}"
```

## Context window only

Use this for the most common operation. `context_window_limit` is top-level in the request. Do not put it inside `model_settings`. For agent responses, verify the effective value at `llm_config.context_window`; `context_window_limit` may be `null` on the agent object.

```bash
curl -sS -X PATCH "$BASE_URL/v1/agents/$AGENT_ID" \
  -H "Authorization: Bearer $LETTA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"context_window_limit": 64000}'
```

For a conversation-scoped context window:

```bash
: "${CONVERSATION_ID:?Set CONVERSATION_ID}"

curl -sS -X PATCH "$BASE_URL/v1/conversations/$CONVERSATION_ID" \
  -H "Authorization: Bearer $LETTA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"context_window_limit": 64000}'
```

## Agent-level model update

Use this for persistent defaults. Omit top-level fields you are not changing.

**Important:** `model_settings` is usually treated as a replacement object, not a deep merge. Read the current agent first and include any existing model settings you want to keep.

```bash
curl -sS -X PATCH "$BASE_URL/v1/agents/$AGENT_ID" \
  -H "Authorization: Bearer $LETTA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-5.2",
    "context_window_limit": 272000,
    "model_settings": {
      "provider_type": "openai",
      "parallel_tool_calls": true,
      "reasoning": { "reasoning_effort": "medium" },
      "max_output_tokens": 128000
    }
  }'
```

For agent responses, verify the effective context window at `llm_config.context_window`.

## Conversation-scoped model update

Use this for temporary changes to the current thread. It does not change the agent's default model or system prompt.
Conversation responses use `context_window_limit` directly.

```bash
: "${CONVERSATION_ID:?Set CONVERSATION_ID}"

curl -sS -X PATCH "$BASE_URL/v1/conversations/$CONVERSATION_ID" \
  -H "Authorization: Bearer $LETTA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-5.2",
    "context_window_limit": 64000,
    "model_settings": {
      "provider_type": "openai",
      "parallel_tool_calls": true,
      "reasoning": { "reasoning_effort": "low" }
    }
  }'
```

## Provider-specific model settings

Pick the shape that matches the provider. Do not send OpenAI reasoning fields to Anthropic models.

| Provider/model handle | Reasoning shape |
| --- | --- |
| `openai/...` | `model_settings.reasoning.reasoning_effort` |
| `chatgpt_oauth/...` or ChatGPT OAuth provider | `model_settings.reasoning.reasoning_effort` with `provider_type: "chatgpt_oauth"`; do not assume every OpenAI effort value is accepted |
| `anthropic/...` | `model_settings.effort`, optionally `model_settings.thinking` |
| `bedrock/...` Claude models | `provider_type: "bedrock"`; check the current API schema before sending reasoning fields |
| `google_ai/...` or `google_vertex/...` | `model_settings.thinking_config.thinking_budget`, optionally `include_thoughts` |

OpenAI:

```json
{
  "provider_type": "openai",
  "parallel_tool_calls": true,
  "reasoning": { "reasoning_effort": "medium" },
  "max_output_tokens": 128000
}
```

Anthropic:

```json
{
  "provider_type": "anthropic",
  "parallel_tool_calls": true,
  "effort": "medium",
  "thinking": { "type": "enabled", "budget_tokens": 12000 },
  "max_output_tokens": 128000
}
```

Google AI:

```json
{
  "provider_type": "google_ai",
  "parallel_tool_calls": true,
  "thinking_config": { "thinking_budget": 12000, "include_thoughts": false }
}
```

Reasoning/effort values are provider-dependent. Common OpenAI values are `none`, `minimal`, `low`, `medium`, `high`, `xhigh`. Anthropic effort commonly uses `low`, `medium`, `high`, `xhigh`, or `max` on supported models.

## System prompt replacement

Only use `system` when the user explicitly asks to change the persistent system prompt. It is a full replacement, not an append.

First inspect the current prompt:

```bash
curl -sS "$BASE_URL/v1/agents/$AGENT_ID" \
  -H "Authorization: Bearer $LETTA_API_KEY" | jq -r '.system'
```

Then send the complete replacement prompt:

```bash
curl -sS -X PATCH "$BASE_URL/v1/agents/$AGENT_ID" \
  -H "Authorization: Bearer $LETTA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"system": "<FULL replacement system prompt. Preserve important existing instructions.>"}'
```

## TypeScript SDK equivalent

```typescript
import { Letta } from "@letta-ai/letta-client";

const client = new Letta({ token: process.env.LETTA_API_KEY! });

await client.agents.update(process.env.AGENT_ID!, {
  model: "openai/gpt-5.2",
  contextWindowLimit: 64000,
  modelSettings: {
    providerType: "openai",
    parallelToolCalls: true,
    reasoning: { reasoningEffort: "medium" },
  },
});
```

## Python SDK equivalent

```python
import os
from letta_client import Letta

client = Letta(token=os.environ["LETTA_API_KEY"])
client.agents.update(
    agent_id=os.environ["AGENT_ID"],
    model="openai/gpt-5.2",
    context_window_limit=64000,
    model_settings={
        "provider_type": "openai",
        "parallel_tool_calls": True,
        "reasoning": {"reasoning_effort": "medium"},
    },
)
```

For a conversation-scoped override:

```python
client.conversations.update(
    conversation_id=os.environ["CONVERSATION_ID"],
    context_window_limit=64000,
)
```

## TypeScript fetch equivalent

```typescript
const baseUrl = process.env.LETTA_BASE_URL ?? "https://api.letta.com";
const agentId = process.env.AGENT_ID!;
const apiKey = process.env.LETTA_API_KEY!;

await fetch(`${baseUrl}/v1/agents/${agentId}`, {
  method: "PATCH",
  headers: {
    Authorization: `Bearer ${apiKey}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    model: "openai/gpt-5.2",
    context_window_limit: 272000,
    model_settings: {
      provider_type: "openai",
      parallel_tool_calls: true,
      reasoning: { reasoning_effort: "medium" },
    },
  }),
});
```

## Verify

```bash
curl -sS "$BASE_URL/v1/agents/$AGENT_ID" \
  -H "Authorization: Bearer $LETTA_API_KEY" | \
  jq '{id, model, context_window_limit, llm_config_context_window: .llm_config.context_window, model_settings, system_chars: (.system | length)}'
```

## Guardrails

- Ask the user before changing your own persistent defaults unless they explicitly requested it.
- Prefer conversation-scoped updates for experiments.
- Keep context windows only as large as needed. Bigger windows increase latency and cost.
- If an update returns `400`, first check model handle validity, provider type, and whether reasoning settings are in the provider's expected shape.
