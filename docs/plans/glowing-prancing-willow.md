# Multi-LLM Provider Architecture

## Context

Claude is hardcoded as LLM. Goal: support OpenAI + OpenAI-compatible providers (Cerebras, Groq, Together, etc.) via env var. Single "openai" adapter with configurable `baseURL` covers all OpenAI-compatible APIs.

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

Key: `streamConversation` normalizes output to Claude-style content blocks (`{ type: "text"|"tool_use", ... }`) so chat.jsx and tool.server.js stay unchanged.

## Files to Create

### 1. `app/services/providers/claude.server.js`
- Extract from current `claude.server.js`
- `createClaudeProvider({ apiKey, model, maxTokens })`
- `formatTools` = identity (MCP already uses Claude format)
- `createToolResultMessage` = `{ role: 'user', content: [{ type: "tool_result", tool_use_id, content }] }`

### 2. `app/services/providers/openai.server.js`
- `createOpenAIProvider({ apiKey, model, maxTokens, baseURL })`
- `baseURL` enables Cerebras/Groq/Together etc.
- `formatTools` = convert `input_schema` → `{ type: "function", function: { name, description, parameters } }`
- `createToolResultMessage` = `{ role: "tool", tool_call_id, content }`
- `streamConversation`:
  - Prepend system prompt as `{ role: "system" }` message
  - Use `openai.chat.completions.create({ stream: true })` async iterable
  - Accumulate text deltas + tool_calls from chunks
  - Normalize to Claude-style content blocks on finish
  - Map `finish_reason === "stop"` → `"end_turn"`, `"tool_calls"` → `"tool_use"`

### 3. `app/services/providers/index.server.js`
- `createLLMProvider(overrides?)` factory — accepts optional config overrides
- Defaults from env vars + `AppConfig`, overrides take precedence
- Env vars: `LLM_PROVIDER`, `CLAUDE_API_KEY`, `OPENAI_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL`

```js
const PROVIDERS = { claude: createClaudeProvider, openai: createOpenAIProvider };

/**
 * @param {Object} [overrides] - Optional config to override env/defaults
 * @param {string} [overrides.provider] - 'claude' | 'openai'
 * @param {string} [overrides.model] - Model name override
 * @param {number} [overrides.maxTokens] - Token limit override
 * @param {string} [overrides.baseURL] - API base URL override
 */
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

## Files to Modify

### 4. `app/services/config.server.js`
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

### 5. `app/routes/chat.jsx`
- Replace `createClaudeService` → `createLLMProvider` from providers/index
- Change loop: `finalMessage.stop_reason` → `result.stopReason` (normalized)
- Pass `llmProvider.formatTools(mcpClient.tools)` for tools
- Move `getSystemPrompt` to shared util or inline (it just reads prompts.json)

Changes are small — swap service creation, use normalized return value:
```js
const llmProvider = createLLMProvider();
let result = { stopReason: null };
while (result.stopReason !== "end_turn") {
  result = await llmProvider.streamConversation(
    { messages: conversationHistory, systemPrompt, tools: llmProvider.formatTools(mcpClient.tools) },
    { onText, onToolUse, onMessage, onContentBlock }
  );
}
```

### 6. `app/services/tool.server.js`
- `createToolService(llmProvider)` — accept provider
- `addToolResultToHistory` uses `llmProvider.createToolResultMessage(toolCallId, content)` instead of hardcoded Claude format

### 7. `app/services/streaming.server.js`
- Genericize error strings: "Claude API" → "AI API", "response from Claude" → "AI response"

### 8. Delete `app/services/claude.server.js`
- All logic moved to `providers/claude.server.js`

### 9. `package.json`
- Add `openai` dependency

### 10. `.env`
- Add: `LLM_PROVIDER`, `OPENAI_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL`

## DB Conversation History

Store in Claude-style content blocks (current behavior). When loading from DB for OpenAI provider, convert message format. Add `formatHistoryMessage(dbMessage)` to provider interface if needed, or handle in chat.jsx with simple mapping.

Note: switching providers between restarts = old conversations may be incompatible. Acceptable for now.

## `getSystemPrompt` utility

Move out of claude.server.js into `app/services/prompt.server.js` (simple: reads prompts.json, returns string). Both providers use it.

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
10. `npm install openai` + create `providers/openai.server.js`
11. Test with `LLM_PROVIDER=openai`

## Verification

1. Set `LLM_PROVIDER=claude` (or unset) → chat works as before
2. Set `LLM_PROVIDER=openai` + `OPENAI_API_KEY` → chat works with OpenAI
3. Set `LLM_PROVIDER=openai` + `LLM_BASE_URL=https://api.cerebras.ai/v1` → works with Cerebras
4. Tool calling works with both providers
5. `npm run lint` + `npm run typecheck` pass
