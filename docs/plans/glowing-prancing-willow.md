# Multi-LLM Provider Architecture

## Context

Claude is hardcoded as LLM. Goal: support OpenAI + OpenAI-compatible providers (Cerebras, Groq, Together, etc.) via env var. Single "openai" adapter with configurable `baseURL` covers all OpenAI-compatible APIs.

**Library choice:** `openai` package (not Vercel AI SDK). Reasons:
- MCP tools already in Claude format → small conversion to OpenAI format
- Existing SSE streaming is custom — AI SDK targets Next.js patterns
- 1 new dep vs 3-4
- `baseURL` option covers all OpenAI-compatible providers natively

## Design: Factory Functions + Shared Interface

**Not class inheritance** — each provider is a factory function returning an object with the same shape. TS migration-friendly: define `interface LLMProvider` later, each factory returns `: LLMProvider`.

### Provider Interface Contract

```js
/**
 * @param {Object} config
 * @param {string} config.apiKey
 * @param {string} config.model - e.g. 'claude-sonnet-4-20250514', 'gpt-4o', 'llama-4-scout-17b-16e-instruct'
 * @param {number} config.maxTokens
 * @param {string} [config.baseURL] - OpenAI-compatible endpoint override
 */
createXxxProvider(config) → {
  streamConversation({ messages, systemPrompt, tools }, { onText, onToolUse, onMessage, onContentBlock })
    → Promise<{ stopReason: "end_turn"|"tool_use", message: { role, content } }>
  formatTools(mcpTools) → provider-formatted tools
  createToolResultMessage(toolCallId, content) → provider-formatted message
}
```

### Callback Contract (normalized across providers)

Both providers MUST call callbacks with identical shapes:

```js
// onText(delta: string) — text chunk
onText("Hello")

// onToolUse({ id, name, input }) — normalized tool call
// Claude: passthrough (already this shape)
// OpenAI: converted from { id, function: { name, arguments } }
onToolUse({ id: "call_xyz", name: "search_products", input: { query: "shoes" } })

// onMessage({ role, content }) — Claude-style content blocks
// content = [{ type: "text", text }, { type: "tool_use", id, name, input }]
onMessage({ role: "assistant", content: [...] })

// onContentBlock({ type, text? }) — individual block completion
onContentBlock({ type: "text", text: "full text here" })
```

Key: providers normalize internally → chat.jsx receives identical data regardless of provider.

### Return Value Contract

```js
{
  stopReason: "end_turn" | "tool_use",  // normalized from provider-specific values
  message: { role: "assistant", content: [/* Claude-style content blocks */] }
}
```

- Claude: `stop_reason` → `stopReason` (rename only)
- OpenAI: `finish_reason === "stop"` → `"end_turn"`, `"tool_calls"` → `"tool_use"`

## Files to Create

### 1. `app/services/providers/claude.server.js`
- Extract from current `claude.server.js`
- `createClaudeProvider({ apiKey, model, maxTokens })`
- `formatTools` = identity (MCP already uses Claude format)
- `createToolResultMessage` = `{ role: 'user', content: [{ type: "tool_result", tool_use_id, content }] }`
- `streamConversation`: wrap existing Anthropic SDK logic, normalize return to `{ stopReason, message }`

### 2. `app/services/providers/openai.server.js`
- `createOpenAIProvider({ apiKey, model, maxTokens, baseURL })`
- `baseURL` enables Cerebras/Groq/Together etc.
- `formatTools` = convert `input_schema` → `{ type: "function", function: { name, description, parameters } }`
- `createToolResultMessage` = `{ role: "tool", tool_call_id, content }`
- `streamConversation`:
  - Prepend system prompt as `{ role: "system" }` message
  - Use raw async iterable: `openai.chat.completions.create({ stream: true })`
  - **Tool call accumulation** (detailed below)
  - Normalize to Claude-style content blocks on finish
  - Call `onToolUse` with normalized `{ id, name, input }` shape

#### Tool Call Accumulation (OpenAI streaming)

OpenAI streams tool_calls as delta fragments indexed by position:

```js
const toolCalls = {};  // index → { id, function: { name, arguments } }

for await (const chunk of stream) {
  const delta = chunk.choices[0]?.delta;
  if (!delta) continue;  // guard: some providers send empty deltas

  if (delta.content) {
    textContent += delta.content;
    onText?.(delta.content);
  }

  if (delta.tool_calls) {
    for (const tc of delta.tool_calls) {
      const idx = tc.index;
      if (!toolCalls[idx]) {
        // First chunk: has id + function.name
        toolCalls[idx] = { id: tc.id, function: { name: tc.function?.name || "", arguments: "" } };
      }
      // Subsequent chunks: append arguments JSON fragments
      if (tc.function?.arguments) {
        toolCalls[idx].function.arguments += tc.function.arguments;
      }
    }
  }
}

// After loop: normalize to Claude-style content blocks
const content = [];
if (textContent) content.push({ type: "text", text: textContent });
for (const tc of Object.values(toolCalls)) {
  content.push({
    type: "tool_use",
    id: tc.id,
    name: tc.function.name,
    input: JSON.parse(tc.function.arguments),
  });
}
```

### 3. `app/services/providers/index.server.js`
- `createLLMProvider(overrides?)` factory — accepts optional config overrides
- Defaults from env vars + `AppConfig`, overrides take precedence
- Env vars: `LLM_PROVIDER`, `CLAUDE_API_KEY`, `OPENAI_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL`

```js
const PROVIDERS = { claude: createClaudeProvider, openai: createOpenAIProvider };

export function createLLMProvider(overrides = {}) {
  const providerName = overrides.provider || AppConfig.api.provider;
  const config = {
    apiKey: ...,   // provider-specific env var
    model: overrides.model || AppConfig.api.model,
    maxTokens: overrides.maxTokens || AppConfig.api.maxTokens,
    baseURL: overrides.baseURL || AppConfig.api.baseURL,
  };
  return PROVIDERS[providerName](config);
}
```

### 4. `app/services/prompt.server.js`
- Extract `getSystemPrompt` from `claude.server.js`
- Simple: reads prompts.json, returns string. Both providers use it.

## Files to Modify

### 5. `app/services/config.server.js`
- Replace `defaultModel` with provider-aware config:
```js
api: {
  provider: process.env.LLM_PROVIDER || 'claude',
  model: process.env.LLM_MODEL || 'claude-sonnet-4-20250514',
  maxTokens: 2000,
  baseURL: process.env.LLM_BASE_URL || undefined,
  defaultPromptType: 'standardAssistant',
}
```
- Genericize error messages: "Claude API" → "AI API"

### 6. `app/routes/chat.jsx`
- Replace `createClaudeService` → `createLLMProvider` from providers/index
- Change loop: `finalMessage.stop_reason` → `result.stopReason` (normalized)
- **Cache formatted tools outside loop** (tools don't change mid-conversation):

```js
const llmProvider = createLLMProvider();
const systemPrompt = getSystemPrompt(promptType);
const formattedTools = llmProvider.formatTools(mcpClient.tools);  // once

let result = { stopReason: null };
while (result.stopReason !== "end_turn") {
  result = await llmProvider.streamConversation(
    { messages: conversationHistory, systemPrompt, tools: formattedTools },
    { onText, onToolUse, onMessage, onContentBlock }
  );
}
```

- `onToolUse` callback stays identical — providers deliver normalized `{ id, name, input }`
- `onMessage` callback stays identical — providers deliver Claude-style content blocks

### 7. `app/services/tool.server.js`
- `createToolService(llmProvider)` — accept provider
- `addToolResultToHistory` uses `llmProvider.createToolResultMessage(toolCallId, content)` instead of hardcoded Claude format
- Everything else unchanged

### 8. `app/services/streaming.server.js`
- Genericize error strings only:
  - "Authentication failed with Claude API" → "Authentication failed with AI API"
  - "Failed to get response from Claude" → "Failed to get AI response"
- Error handling logic (status-based branching) works for both SDKs — both expose `.status`

### 9. Delete `app/services/claude.server.js`
- All logic moved to `providers/claude.server.js`

### 10. `package.json`
- Add `openai` dependency

### 11. `.env`
- Add: `LLM_PROVIDER`, `OPENAI_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL`

## DB Conversation History

Store in Claude-style content blocks (current behavior). When loading from DB for OpenAI provider, convert message format in `streamConversation` internally.

### Provider field on Conversation

Add `provider` column to track which provider created a conversation:

```prisma
model Conversation {
  id        String  @id @default(uuid())
  provider  String  @default("claude")
  // ...existing fields
}
```

On conversation load: if `conv.provider !== currentProvider`, log warning. For now, allow continuation (formats are normalized). Later: reject or convert.

## OpenAI-Compatible Provider Notes

| Provider | baseURL | Gotchas |
|----------|---------|---------|
| Cerebras | `https://api.cerebras.ai/v1` | `system` role elevated to developer-level — prompts may behave differently |
| Groq | `https://api.groq.com/openai/v1` | Structured outputs model-dependent; tool calling quality varies by model |
| Together | `https://api.together.xyz/v1` | Tool calling support is model-specific |

Common: guard `if (!delta) continue;` in streaming loop — some providers send empty delta chunks.

## Implementation Order

1. Create `providers/` dir + `claude.server.js` (extract existing logic)
2. Create `providers/index.server.js` (factory)
3. Create `app/services/prompt.server.js` (extract getSystemPrompt)
4. Update `config.server.js`
5. Update `chat.jsx` to use provider interface
6. Update `tool.server.js` to accept provider
7. Update `streaming.server.js` error strings
8. Delete old `claude.server.js`
9. Verify Claude still works (no behavior change)
10. Add `provider` field to Conversation model + `npm run setup`
11. `npm install openai` + create `providers/openai.server.js`
12. Test with `LLM_PROVIDER=openai`

## Verification

1. Set `LLM_PROVIDER=claude` (or unset) → chat works as before
2. Set `LLM_PROVIDER=openai` + `OPENAI_API_KEY` → chat works with OpenAI
3. Set `LLM_PROVIDER=openai` + `LLM_BASE_URL=https://api.cerebras.ai/v1` → works with Cerebras
4. Tool calling works with both providers
5. `npm run lint` + `npm run typecheck` pass
