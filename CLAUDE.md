# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start Shopify app dev server (tunneled)
npm run build        # Production build (react-router build)
npm run setup        # Prisma generate + migrate deploy
npm run lint         # ESLint with cache
npm run typecheck    # React Router typegen + tsc --noEmit
npm run deploy       # Deploy to Shopify
npm run start        # Serve production build
```

No test framework configured. Validation is via `lint` and `typecheck`.

## Architecture

Shopify template app embedding an AI chat widget on storefronts. Uses **Tool-Augmented Generation (TAG)** via MCP — Claude calls Shopify APIs as tools in a multi-turn agentic loop.

### Request Flow

```
User message → POST /chat → SSE stream created
  → MCPClient connects to 2 MCP endpoints (storefront + customer)
  → Load conversation history from DB
  → Agentic loop:
      Claude streams response → if tool_use → MCPClient.callTool() → result added to history → loop
  → SSE events sent to frontend: chunk, tool_use, product_results, auth_required, end_turn
```

### Key Files

- **`app/routes/chat.jsx`** — Chat orchestrator. SSE endpoint, agentic while-loop (`stop_reason !== "end_turn"`)
- **`app/mcp-client.js`** — JSON-RPC 2.0 client connecting to storefront (`/api/mcp`) and customer (`/customer/api/mcp`) MCP endpoints. Routes tool calls to correct endpoint.
- **`app/services/claude.server.js`** — Anthropic SDK wrapper, streaming conversation
- **`app/services/tool.server.js`** — Tool result formatting, error handling, product data extraction
- **`app/services/streaming.server.js`** — SSE stream creation/management
- **`app/services/config.server.js`** — Configuration management
- **`app/prompts/prompts.json`** — System prompts for Claude
- **`extensions/chat-bubble/`** — Shopify theme extension (Liquid block + JS/CSS chat UI)
  - `assets/chat.js` — Frontend SSE client, message rendering, UI state

### Database (Prisma + SQLite)

Schema at `prisma/schema.prisma`. Key models: `Session`, `CustomerToken`, `CodeVerifier` (PKCE OAuth), `Conversation`, `Message`, `CustomerAccountUrls`.

Run `npm run setup` after schema changes.

## Tech Stack

- **Framework:** React Router v7 (file-based routing in `app/routes/`)
- **AI:** @anthropic-ai/sdk (Claude) with MCP tool integration
- **Shopify:** @shopify/shopify-app-react-router, App Bridge React
- **DB:** Prisma + SQLite
- **Build:** Vite
- **Language:** JavaScript (JSX) with TypeScript checking enabled

## Conventions

- ES modules (`"type": "module"`)
- `.server.js` suffix = server-only code (React Router convention)
- Customer MCP endpoint derived from store URL: `hostUrl.replace(/(\.myshopify\.com)$/, '.account$1')`
- SSE event format: `event: {type}\ndata: {json}\n\n`
- Node >= 20.10
