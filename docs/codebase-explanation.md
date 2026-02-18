# Shop Chat Agent - コードベース解説

## 概要

ShopifyストアフロントにAI駆動チャットウィジェットを埋め込むテンプレートアプリです。Model Context Protocol (MCP)を使ってShopify APIと統合し、製品検索、カート操作、注文追跡などを会話形式で実現します。

`★ Insight ─────────────────────────────────────`
このアプリの核心は、**MCP (Model Context Protocol)** を使ってClaudeとShopify APIを繋ぐアーキテクチャです。MCPにより、LLMが動的にShopifyの機能（製品検索、カート管理等）をツールとして呼び出せるようになっています。
`─────────────────────────────────────────────────`

---

## 1. MCPクライアント - Shopify API統合の中核

`app/mcp-client.js:8-28`では、2種類のMCPエンドポイントに接続します:

```javascript
class MCPClient {
  constructor(hostUrl, conversationId, shopId, customerMcpEndpoint) {
    this.tools = [];
    this.customerTools = [];      // 顧客向けツール（注文追跡、リターン等）
    this.storefrontTools = [];    // ストアフロントツール（製品検索、カート操作等）

    this.storefrontMcpEndpoint = `${hostUrl}/api/mcp`;
    const accountHostUrl = hostUrl.replace(/(\.myshopify\.com)$/, '.account$1');
    this.customerMcpEndpoint = customerMcpEndpoint || `${accountHostUrl}/customer/api/mcp`;
  }
```

**ツール検索の仕組み** (`app/mcp-client.js:85-112`):

```javascript
async connectToStorefrontServer() {
  const response = await this._makeJsonRpcRequest(
    this.storefrontMcpEndpoint,
    "tools/list",  // JSON-RPCメソッド
    {},
    headers
  );

  const toolsData = response.result && response.result.tools ? response.result.tools : [];
  const storefrontTools = this._formatToolsData(toolsData);

  this.storefrontTools = storefrontTools;
  this.tools = [...this.tools, ...storefrontTools];  // 全ツールリストに追加
}
```

**ツール呼び出し** (`app/mcp-client.js:122-130`):

```javascript
async callTool(toolName, toolArgs) {
  if (this.customerTools.some(tool => tool.name === toolName)) {
    return this.callCustomerTool(toolName, toolArgs);
  } else if (this.storefrontTools.some(tool => tool.name === toolName)) {
    return this.callStorefrontTool(toolName, toolArgs);
  } else {
    throw new Error(`Tool ${toolName} not found`);
  }
}
```

---

## 2. チャットエンドポイント - 会話のオーケストレーション

`app/routes/chat.jsx:65-104`でSSE (Server-Sent Events) ストリームを開始:

```javascript
async function handleChatRequest(request) {
  const body = await request.json();
  const userMessage = body.message;
  const conversationId = body.conversation_id || Date.now().toString();

  // SSEストリームを作成
  const responseStream = createSseStream(async (stream) => {
    await handleChatSession({
      request,
      userMessage,
      conversationId,
      promptType: body.prompt_type || AppConfig.api.defaultPromptType,
      stream,
    });
  });

  return new Response(responseStream, {
    headers: getSseHeaders(request),
  });
}
```

**メインループ** (`app/routes/chat.jsx:182-266`) - Claudeとの対話を継続:

```javascript
let finalMessage = { role: "user", content: userMessage };

while (finalMessage.stop_reason !== "end_turn") {
  finalMessage = await claudeService.streamConversation(
    {
      messages: conversationHistory,
      promptType,
      tools: mcpClient.tools, // 利用可能なMCPツールを渡す
    },
    {
      onText: (textDelta) => {
        stream.sendMessage({ type: "chunk", chunk: textDelta });
      },

      onMessage: (message) => {
        conversationHistory.push({
          role: message.role,
          content: message.content,
        });
        saveMessage(
          conversationId,
          message.role,
          JSON.stringify(message.content),
        );
      },

      onToolUse: async (content) => {
        const toolName = content.name;
        const toolArgs = content.input;
        const toolUseId = content.id;

        stream.sendMessage({
          type: "tool_use",
          tool_use_message: `Calling tool: ${toolName} with arguments: ${JSON.stringify(toolArgs)}`,
        });

        // MCPツールを実行
        const toolUseResponse = await mcpClient.callTool(toolName, toolArgs);

        // 結果をメモリに追加
        if (toolUseResponse.error) {
          await toolService.handleToolError(/* ... */);
        } else {
          await toolService.handleToolSuccess(/* ... */);
        }

        // 新しいターンを開始（エージェントが結果を解釈）
        stream.sendMessage({ type: "new_message" });
      },
    },
  );
}
```

`★ Insight ─────────────────────────────────────`
この`while`ループが重要です。Claudeがツールを呼び出すたびに、その結果を会話履歴に追加し、Claudeに再度問い合わせます。これにより、複数ツールの連鎖実行（例: 製品検索→カートに追加→チェックアウトURL生成）が可能になります。
`─────────────────────────────────────────────────`

---

## 3. Claudeサービス - LLMとの統合

`app/services/claude.server.js:30-73`でストリーミングAPI呼び出し:

```javascript
const streamConversation = async (
  { messages, promptType = AppConfig.api.defaultPromptType, tools },
  streamHandlers,
) => {
  const systemInstruction = getSystemPrompt(promptType);

  // Claude Streaming API
  const stream = await anthropic.messages.stream({
    model: AppConfig.api.defaultModel,
    max_tokens: AppConfig.api.maxTokens,
    system: systemInstruction,
    messages,
    tools: tools && tools.length > 0 ? tools : undefined, // MCPツールを渡す
  });

  // イベントハンドラ設定
  if (streamHandlers.onText) {
    stream.on("text", streamHandlers.onText);
  }

  const finalMessage = await stream.finalMessage();

  // ツール使用要求を処理
  if (streamHandlers.onToolUse && finalMessage.content) {
    for (const content of finalMessage.content) {
      if (content.type === "tool_use") {
        await streamHandlers.onToolUse(content);
      }
    }
  }

  return finalMessage;
};
```

---

## 4. ツールサービス - MCPツール実行管理

`app/services/tool.server.js:42-49`で製品検索結果を処理:

```javascript
const handleToolSuccess = async (
  toolUseResponse,
  toolName,
  toolUseId,
  conversationHistory,
  productsToDisplay,
  conversationId,
) => {
  // 製品検索ツールの場合、結果を抽出
  if (toolName === AppConfig.tools.productSearchName) {
    productsToDisplay.push(...processProductSearchResult(toolUseResponse));
  }

  addToolResultToHistory(
    conversationHistory,
    toolUseId,
    toolUseResponse.content,
    conversationId,
  );
};
```

**認証エラーハンドリング** (`app/mcp-client.js:209-223`):

```javascript
try {
  const response = await this._makeJsonRpcRequest(/* ... */);
  return response.result || response;
} catch (error) {
  if (error.status === 401) {
    console.log("Unauthorized, generating authorization URL for customer");

    const authResponse = await generateAuthUrl(
      this.conversationId,
      this.shopId,
    );

    // 認証URLをフロントエンドに返す
    return {
      error: {
        type: "auth_required",
        data: `You need to authorize the app to access your customer data. [Click here to authorize](${authResponse.url})`,
      },
    };
  }
  throw error;
}
```

---

## 5. フロントエンドチャットUI

`extensions/chat-bubble/assets/chat.js:473-535`でSSEストリームを処理:

```javascript
streamResponse: async function(userMessage, conversationId, messagesContainer) {
  const response = await fetch(streamUrl, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'text/event-stream',
      'X-Shopify-Shop-Id': shopId
    },
    body: requestBody
  });

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  while (true) {
    const { value, done } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n\n');
    buffer = lines.pop() || '';

    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const data = JSON.parse(line.slice(6));
        this.handleStreamEvent(data, currentMessageElement, messagesContainer, userMessage,
          (newElement) => { currentMessageElement = newElement; });
      }
    }
  }
}
```

**イベント処理** (`extensions/chat-bubble/assets/chat.js:546-616`):

```javascript
handleStreamEvent: function(data, currentMessageElement, messagesContainer, userMessage, updateCurrentElement) {
  switch (data.type) {
    case 'chunk':
      // テキストチャンクをリアルタイム表示
      currentMessageElement.dataset.rawText += data.chunk;
      currentMessageElement.textContent = currentMessageElement.dataset.rawText;
      break;

    case 'tool_use':
      // ツール使用メッセージを表示
      ShopAIChat.Message.addToolUse(data.tool_use_message, messagesContainer);
      break;

    case 'auth_required':
      // 最後のメッセージを保存（認証後に再実行）
      sessionStorage.setItem('shopAiLastMessage', userMessage || '');
      break;

    case 'product_results':
      // 製品検索結果をカード表示
      ShopAIChat.UI.displayProductResults(data.products);
      break;

    case 'new_message':
      // 新しいメッセージ要素を作成（次のClaudeレスポンス用）
      const newMessageElement = document.createElement('div');
      newMessageElement.classList.add('shop-ai-message', 'assistant');
      messagesContainer.appendChild(newMessageElement);
      updateCurrentElement(newMessageElement);
      break;
  }
}
```

`★ Insight ─────────────────────────────────────`
フロントエンドはSSEストリームを解析し、イベントタイプに応じてUIを動的に更新します。`new_message`イベントが特に重要で、ツール呼び出し後の新しいClaudeレスポンスを受け取る準備をします。これにより、会話の連続性とツール実行の透明性が保たれます。
`─────────────────────────────────────────────────`

---

## アーキテクチャフロー

1. **ユーザー入力** → フロントエンドがPOST `/chat`
2. **MCP接続** → バックエンドがStorefront & Customer MCPサーバーに接続し利用可能ツールを取得
3. **Claude呼び出し** → 会話履歴とツールリストをClaudeに送信
4. **ツール実行** → Claudeがツールを要求したらMCPエンドポイント経由で実行
5. **結果返却** → ツール結果を会話履歴に追加し、Claudeに再度問い合わせ
6. **SSEストリーム** → 全過程をリアルタイムでフロントエンドにストリーミング

---

# RAG設計について

**いいえ、このコードベースにはRAG (Retrieval-Augmented Generation) の設計はされていません。**

## 現在の設計: Tool-Augmented Generation

このアプリは**MCP (Model Context Protocol) ベースのツール拡張**アプローチを採用しています:

### 1. データベース構造 - RAG要素なし

`prisma/schema.prisma:63-72`を見ると、メッセージストレージはシンプルです:

```prisma
model Message {
  id             String       @id @default(cuid())
  conversationId String
  conversation   Conversation @relation(fields: [conversationId], references: [id], onDelete: Cascade)
  role           String       // "user" or "assistant"
  content        String       // 単純なテキストストレージ
  createdAt      DateTime     @default(now())

  @@index([conversationId])
}
```

**RAGに必要だが欠けているもの:**

- ベクトル埋め込み (embeddings) カラム
- ドキュメントチャンクテーブル
- メタデータやスコアリング用フィールド
- ベクトル類似度検索のインデックス

### 2. 情報取得の仕組み - MCPツール経由

`README.md:20-24`で言及される検索機能:

```
- `can you search for snowboards` > will use the `search_shop_catalog` MCP tool.
- `what languages is your store available in?` > will use the `search_shop_policies_and_faqs` MCP tool.
```

これらは**Shopify側のMCPサーバー**が提供するツールで、アプリ内で実装されていません。

### 3. コンテキスト注入 - 会話履歴のみ

`app/routes/chat.jsx:162-177`では、単純な会話履歴を取得:

```javascript
// Fetch all messages from the database for this conversation
const dbMessages = await getConversationHistory(conversationId);

// Format messages for Claude API
conversationHistory = dbMessages.map((dbMessage) => {
  let content;
  try {
    content = JSON.parse(dbMessage.content);
  } catch (e) {
    content = dbMessage.content;
  }
  return {
    role: dbMessage.role,
    content,
  };
});
```

**RAGなら行うべき処理:**

- ユーザークエリのベクトル埋め込み生成
- ドキュメントストアからセマンティック検索
- 関連チャンクをシステムプロンプトに動的注入
- スコアリングとリランキング

`★ Insight ─────────────────────────────────────`
このアプリは「Knowledge retrieval in LLM」ではなく「API tool execution」で情報を取得します。RAGが**静的ドキュメントからのセマンティック検索**なのに対し、MCPツールは**動的APIからのリアルタイム情報取得**です。製品在庫、カート状態、注文ステータスなど、常に変化するデータに適しています。
`─────────────────────────────────────────────────`

### 4. システムプロンプト - 静的テンプレート

`app/prompts/prompts.json:4`のプロンプトは完全に静的:

```json
{
  "content": "You are a helpful store assistant for an e-commerce shop. Answer the customer's questions in a friendly, helpful way about products, shipping, returns, or anything else about the store.\n\nFormatting guidelines:\n..."
}
```

RAGなら、ここに検索結果を動的挿入します（例: `{retrieved_context}`プレースホルダー）。

---

## RAG vs Tool-Augmented の比較

| 側面                 | RAG                             | このアプリ (Tool-Augmented) |
| -------------------- | ------------------------------- | --------------------------- |
| **情報源**           | 静的ドキュメントDB              | ShopifyのリアルタイムAPI    |
| **検索手法**         | ベクトル類似度検索              | MCPツール呼び出し           |
| **コンテキスト注入** | システムプロンプトに動的挿入    | ツール結果を会話履歴に追加  |
| **適用場面**         | ポリシー、FAQ、ドキュメント     | 製品在庫、カート、注文状態  |
| **実装複雑度**       | 高（embeddings, vector DB必要） | 中（MCPサーバー統合のみ）   |

**もしRAGを追加するなら:**

- ストアポリシー、FAQをベクトルDBに保存
- ユーザークエリから関連情報を検索
- システムプロンプトに注入して精度向上

現状は、動的データに特化した**Tool-Augmented Generation**で、ShopifyエコシステムとのAPI統合が強みです。

---

# Claude Embeddings APIとAIエージェントのコード

## 1. Claude Embeddings APIについて

**いいえ、Anthropic (Claude) はembeddings APIを提供していません。**

2026年現在も、Anthropicは独自のembeddingsモデルを提供しておらず、以下のサードパーティプロバイダーの利用を推奨しています：

### 推奨プロバイダー

- **[Voyage AI](https://docs.voyage.ai/)** (公式パートナー) - `voyage-3.5`モデル (1024次元)
- **[OpenAI](https://platform.openai.com/docs/guides/embeddings)** - `text-embedding-3-small/large`
- **[Cohere](https://docs.cohere.com/docs/embeddings)** - `embed-english-v3.0`

このプロジェクトで実装する場合、例：

```javascript
import { VoyageAIClient } from "voyageai";

const voyageClient = new VoyageAIClient({ apiKey: process.env.VOYAGE_API_KEY });

async function generateEmbeddings(texts) {
  const response = await voyageClient.embed({
    input: texts,
    model: "voyage-3.5",
  });
  return response.embeddings; // [[0.1, 0.2, ...], ...]
}
```

---

## 2. AIエージェントのコード解説

このプロジェクトのAIエージェントは、複数のファイルに分散した**協調システム**として実装されています。

### 主要コンポーネント

#### A. エージェントの頭脳: `app/services/claude.server.js`

`app/services/claude.server.js:14-89` - Claude APIとの通信を管理:

```javascript
export function createClaudeService(apiKey = process.env.CLAUDE_API_KEY) {
  const anthropic = new Anthropic({ apiKey });

  /**
   * エージェントのメイン推論ループ
   */
  const streamConversation = async (
    {
      messages, // 会話履歴（エージェントのメモリ）
      promptType, // システムプロンプト（エージェントの役割定義）
      tools, // 利用可能なツール（エージェントの能力）
    },
    streamHandlers,
  ) => {
    const systemInstruction = getSystemPrompt(promptType);

    // Claude APIにストリーミングリクエスト
    const stream = await anthropic.messages.stream({
      model: AppConfig.api.defaultModel, // claude-sonnet-4
      max_tokens: AppConfig.api.maxTokens,
      system: systemInstruction, // エージェントの性格・役割
      messages, // 会話コンテキスト
      tools: tools && tools.length > 0 ? tools : undefined, // ツール定義
    });

    // イベントハンドラでリアルタイム応答を処理
    if (streamHandlers.onText) {
      stream.on("text", streamHandlers.onText); // テキスト生成
    }

    const finalMessage = await stream.finalMessage();

    // ツール使用要求を検出・実行
    if (streamHandlers.onToolUse && finalMessage.content) {
      for (const content of finalMessage.content) {
        if (content.type === "tool_use") {
          await streamHandlers.onToolUse(content); // ツール実行トリガー
        }
      }
    }

    return finalMessage;
  };

  return { streamConversation, getSystemPrompt };
}
```

`★ Insight ─────────────────────────────────────`
このサービスは**エージェントの推論エンジン**です。`messages`配列が短期記憶、`systemInstruction`が性格、`tools`が行動能力を定義します。Claudeは「次に何をすべきか」を決定し、必要なら`tool_use`ブロックを返してアクションを要求します。
`─────────────────────────────────────────────────`

---

#### B. エージェントのオーケストレーター: `app/routes/chat.jsx`

`app/routes/chat.jsx:115-282` - エージェントの実行ループを管理:

```javascript
async function handleChatSession({
  request,
  userMessage,
  conversationId,
  promptType,
  stream,
}) {
  const claudeService = createClaudeService();
  const toolService = createToolService();

  // エージェントの「ツールベルト」を準備
  const mcpClient = new MCPClient(
    shopDomain,
    conversationId,
    shopId,
    mcpApiUrl,
  );

  // MCPサーバーに接続してツールリストを取得
  let storefrontMcpTools = await mcpClient.connectToStorefrontServer();
  let customerMcpTools = await mcpClient.connectToCustomerServer();

  // エージェントのメモリ（会話履歴）を取得
  const dbMessages = await getConversationHistory(conversationId);
  conversationHistory = dbMessages.map(/* フォーマット */);

  let finalMessage = { role: "user", content: userMessage };

  // ========== エージェントの実行ループ ==========
  while (finalMessage.stop_reason !== "end_turn") {
    finalMessage = await claudeService.streamConversation(
      {
        messages: conversationHistory,
        promptType,
        tools: mcpClient.tools, // エージェントが使える全ツール
      },
      {
        // ① テキスト生成時のコールバック
        onText: (textDelta) => {
          stream.sendMessage({ type: "chunk", chunk: textDelta });
        },

        // ② メッセージ完了時のコールバック（メモリに保存）
        onMessage: (message) => {
          conversationHistory.push({
            role: message.role,
            content: message.content,
          });
          saveMessage(
            conversationId,
            message.role,
            JSON.stringify(message.content),
          );
        },

        // ③ ツール使用要求時のコールバック（アクション実行）
        onToolUse: async (content) => {
          const toolName = content.name;
          const toolArgs = content.input;
          const toolUseId = content.id;

          stream.sendMessage({
            type: "tool_use",
            tool_use_message: `Calling tool: ${toolName} with arguments: ${JSON.stringify(toolArgs)}`,
          });

          // MCPツールを実行
          const toolUseResponse = await mcpClient.callTool(toolName, toolArgs);

          // 結果をメモリに追加
          if (toolUseResponse.error) {
            await toolService.handleToolError(/* ... */);
          } else {
            await toolService.handleToolSuccess(/* ... */);
          }

          // 新しいターンを開始（エージェントが結果を解釈）
          stream.sendMessage({ type: "new_message" });
        },
      },
    );
  }

  // エージェントのタスク完了
  stream.sendMessage({ type: "end_turn" });
}
```

`★ Insight ─────────────────────────────────────`
この`while`ループが**エージェントの思考→行動→観察サイクル**を実装しています：

1. **思考** (`streamConversation`) - Claudeが次のアクションを決定
2. **行動** (`onToolUse`) - MCPツールを実行
3. **観察** (`handleToolSuccess`) - 結果を会話履歴に追加
4. **再思考** - ループが続き、Claudeが結果を解釈して次のステップを決定

このパターンは**ReAct (Reasoning + Acting)** フレームワークの実装です。
`─────────────────────────────────────────────────`

---

#### C. エージェントの能力インターフェース: `app/mcp-client.js`

`app/mcp-client.js:8-288` - エージェントがShopify APIにアクセスする手段:

```javascript
class MCPClient {
  constructor(hostUrl, conversationId, shopId, customerMcpEndpoint) {
    this.tools = []; // エージェントが使える全ツールのリスト
    this.customerTools = []; // 顧客関連ツール
    this.storefrontTools = []; // ストアフロント関連ツール

    // MCPエンドポイント
    this.storefrontMcpEndpoint = `${hostUrl}/api/mcp`;
    this.customerMcpEndpoint = customerMcpEndpoint;
  }

  /**
   * エージェントの「ツールディスカバリー」
   */
  async connectToStorefrontServer() {
    const response = await this._makeJsonRpcRequest(
      this.storefrontMcpEndpoint,
      "tools/list", // 利用可能なツールを問い合わせ
      {},
      headers,
    );

    const toolsData = response.result.tools;
    const storefrontTools = this._formatToolsData(toolsData);

    this.storefrontTools = storefrontTools;
    this.tools = [...this.tools, ...storefrontTools];

    return storefrontTools;
  }

  /**
   * エージェントの「ツール実行」
   */
  async callTool(toolName, toolArgs) {
    // どのMCPサーバーにツールがあるか判定
    if (this.customerTools.some((tool) => tool.name === toolName)) {
      return this.callCustomerTool(toolName, toolArgs);
    } else if (this.storefrontTools.some((tool) => tool.name === toolName)) {
      return this.callStorefrontTool(toolName, toolArgs);
    } else {
      throw new Error(`Tool ${toolName} not found`);
    }
  }

  /**
   * 認証が必要なツールの処理
   */
  async callCustomerTool(toolName, toolArgs) {
    try {
      const response = await this._makeJsonRpcRequest(
        this.customerMcpEndpoint,
        "tools/call",
        { name: toolName, arguments: toolArgs },
        headers,
      );

      return response.result;
    } catch (error) {
      if (error.status === 401) {
        // エージェントが認証URLを生成してユーザーに提示
        const authResponse = await generateAuthUrl(
          this.conversationId,
          this.shopId,
        );

        return {
          error: {
            type: "auth_required",
            data: `You need to authorize the app... [Click here](${authResponse.url})`,
          },
        };
      }
      throw error;
    }
  }
}
```

---

#### D. エージェントのツール処理: `app/services/tool.server.js`

`app/services/tool.server.js:42-149` - ツール実行結果の処理:

```javascript
export function createToolService() {
  /**
   * ツール実行成功時の処理
   */
  const handleToolSuccess = async (
    toolUseResponse,
    toolName,
    toolUseId,
    conversationHistory,
    productsToDisplay,
    conversationId,
  ) => {
    // 製品検索ツールの特別処理
    if (toolName === AppConfig.tools.productSearchName) {
      productsToDisplay.push(...processProductSearchResult(toolUseResponse));
    }

    // ツール結果をエージェントのメモリに追加
    addToolResultToHistory(
      conversationHistory,
      toolUseId,
      toolUseResponse.content,
      conversationId,
    );
  };

  /**
   * ツール結果を会話履歴に追加（エージェントのメモリ更新）
   */
  const addToolResultToHistory = async (
    conversationHistory,
    toolUseId,
    content,
    conversationId,
  ) => {
    const toolResultMessage = {
      role: "user", // Claudeへの「観察結果」として返す
      content: [
        {
          type: "tool_result",
          tool_use_id: toolUseId,
          content: content,
        },
      ],
    };

    // インメモリ履歴に追加
    conversationHistory.push(toolResultMessage);

    // 永続化
    if (conversationId) {
      await saveMessage(
        conversationId,
        "user",
        JSON.stringify(toolResultMessage.content),
      );
    }
  };

  return { handleToolError, handleToolSuccess, addToolResultToHistory };
}
```

---

## エージェントアーキテクチャ全体図

```
┌─────────────────────────────────────────────────────────────┐
│                    AIエージェント実行フロー                    │
└─────────────────────────────────────────────────────────────┘

1. ユーザー入力
   ↓
2. chat.jsx (オーケストレーター)
   ├─ MCP Client初期化 → ツールディスカバリー
   ├─ 会話履歴取得 (DB) → エージェントのメモリロード
   └─ 実行ループ開始
      ↓
3. claude.server.js (推論エンジン)
   ├─ システムプロンプト + 会話履歴 + ツールリスト → Claude API
   ├─ Claude が次のアクションを決定
   └─ tool_use ブロックを返す OR テキスト応答
      ↓
4. mcp-client.js (ツール実行)
   ├─ JSON-RPC でMCPサーバーに接続
   ├─ ツール実行 (例: search_shop_catalog)
   └─ 結果を返す
      ↓
5. tool.server.js (結果処理)
   ├─ ツール結果を会話履歴に追加
   └─ DB に保存 (永続化)
      ↓
6. ループ継続 (step 3に戻る)
   └─ Claude が結果を解釈 → 次のアクション決定
      ↓
7. stop_reason === "end_turn"
   └─ エージェント完了
```

`★ Insight ─────────────────────────────────────`

このエージェントは**マルチターン推論**を実装しています。単純なLLM呼び出しと異なり、Claudeは複数回の思考サイクルを経て、ツールを連鎖的に使用します（例: 製品検索 → カート追加 → チェックアウトURL生成）。`while`ループと会話履歴への結果追加が、この自律的な振る舞いを可能にしています。

- TAG over RAG: This codebase uses Tool-Augmented Generation, not Retrieval-Augmented Generation. Claude dynamically calls Shopify APIs via MCP at inference
  time rather than pre-fetching docs into a vector store. This is a key architectural distinction worth highlighting.
- Dual MCP endpoints: The storefront vs customer endpoint split means tool routing (callTool in mcp-client.js) is a critical piece — it inspects which tool
  list a name belongs to before dispatching. This pattern would be easy to break if tools are added without updating both connection methods.
- No tests: The repo has no test framework, so lint + typecheck are the only automated quality gates. Future Claude instances should know this to avoid
  trying to run nonexistent test commands.

`─────────────────────────────────────────────────`

The file covers:

1. All dev commands (dev, build, setup, lint, typecheck, deploy)
2. Architecture — the TAG agentic loop, SSE streaming flow, and MCP dual-endpoint pattern
3. Key files with their roles, so future instances can jump straight to the right file
4. Database schema overview and migration workflow
5. Conventions — .server.js suffix, ES modules, SSE event format, Node version requirement

---

## 参考リンク

- [Embeddings - Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/embeddings)
