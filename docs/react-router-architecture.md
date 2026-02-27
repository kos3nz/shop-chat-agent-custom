# React Router v7 アーキテクチャ解説

このドキュメントは、このプロジェクトで使われている React Router v7 の仕組みと、関連するパッケージ・ファイルの役割を体系的にまとめたものです。

---

## 目次

1. [サーバーの受け口：なぜ `server.ts` がないのか](#1-サーバーの受け口なぜ`server.ts`がないのか)
2. [各ファイルの役割](#2-各ファイルの役割)
3. [パッケージの役割と依存関係](#3-パッケージの役割と依存関係)
4. [Cloudflare Workers への移行について](#4-cloudflare-workers-への移行について)
5. [root.jsx がどう差し込まれるか](#5-rootjsx-がどう差し込まれるか)
6. [entry.client.jsx がない理由](#6-entryclientjsx-がない理由)
7. [useLoaderData とクライアントへのデータ渡し](#7-useloaderdata-とクライアントへのデータ渡し)
8. [context とは何か](#8-context-とは何か)
9. [App Bridge の役割](#9-app-bridge-の役割)
10. [Hydrogen（Cloudflare Workers）での server.ts](#10-hydrogencloudflare-workersでのserverts)

---

## 1. サーバーの受け口：なぜ `server.ts` がないのか

### Hydrogen との違い

Hydrogen プロジェクトにはルートに `server.ts` があります。これは Hydrogen が **Cloudflare Workers（Oxygen）** で動くためで、Workers の `fetch` イベントハンドラ形式が必要です：

```ts
// Hydrogen の server.ts（Cloudflare Workers 形式）
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext) {
    // リクエストの受け口
  },
};
```

このプロジェクトは **Node.js** で動くため、`@react-router/serve` が内部で Express サーバーを立ち上げてその役割を担います。`server.ts` は「存在するが、ライブラリ側に隠蔽されている」状態です。

### `@react-router/serve` によるサーバー起動

```json
// package.json
"start": "react-router-serve ./build/server/index.js"
```

`react-router-serve` は React Router チームが提供する **Express ベースの組み込みサーバー CLI** です。内部で `compression` / `express.static` / `morgan` の3つのミドルウェアを持ちます。

```
HTTP リクエスト
    ↓
react-router-serve（Express プロセス）
    ├─ compression / express.static / morgan middleware（通過型）
    └─ createRequestHandler({ build }) ← 終端ハンドラ
           ↓ build/server/index.js を「モジュール」として参照（サーバーではない）
       React Router がルーティング・SSR を処理
```

> **注意：** `build/server/index.js` は Express とは別のサーバーではなく、`createRequestHandler` に渡される **ESM モジュール**（ルート定義・entry 関数の集合体）です。リクエストごとに Express ミドルウェアから呼び出されます。

カスタムサーバーが必要になった場合は `@react-router/serve` を捨てて `@react-router/express` を使い、自前の `server.ts` を書く形に移行できます。

---

## 2. 各ファイルの役割

### `app/routes.js` — ルート設定

```js
import { flatRoutes } from "@react-router/fs-routes";
export default flatRoutes();
```

`app/routes/` ディレクトリのファイル構造を**自動的にスキャン**してルートマップを生成します。ファイル名の命名規則（`.` でネスト、`$param` で動的セグメント）を URL パターンに変換します。`vite.config.js` 内の `reactRouter()` プラグインがこのファイルを参照します。

### `app/root.jsx` — ルートレイアウト

```jsx
export default function App() {
  return (
    <html>
      <head>
        <Meta />
        <Links />
      </head>
      <body>
        <Outlet /> {/* 各ルートコンポーネントがここに入る */}
        <Scripts />
      </body>
    </html>
  );
}
```

全ルートを包む**最上位レイアウト**。`<Outlet />` が各ページのコンポーネントを差し込む場所です。Next.js の `_document.js` + `_app.js` に近い概念です。

### `app/entry.server.jsx` — SSR エントリーポイント

React Router フレームワークがサーバー側でレンダリングするときに呼ぶ**SSR のエントリー関数**です。

```jsx
export default async function handleRequest(
  request,
  responseStatusCode,
  responseHeaders,
  reactRouterContext,
) {
  addDocumentResponseHeaders(request, responseHeaders); // Shopify 必須ヘッダー注入
  const callbackName = isbot(userAgent) ? "onAllReady" : "onShellReady";

  renderToPipeableStream(
    <ServerRouter context={reactRouterContext} url={request.url} />,
    {
      [callbackName]: () => {
        /* HTML ストリームを Response として返す */
      },
    },
  );
}
```

重要な処理が2つあります：

- **`addDocumentResponseHeaders`** — Shopify が要求する CSP（Content Security Policy）などのヘッダーを注入
- **`isbot` によるボット判定** — クローラーには `onAllReady`（全コンポーネント準備完了後）、通常ユーザーには `onShellReady`（シェル HTML 準備後すぐ）を使い分けてストリーミング

このファイルが明示的に書かれている理由は `addDocumentResponseHeaders` のカスタマイズが必要なためです。書かなければ React Router のデフォルト実装が使われます。

### `app/shopify.server.js` — Shopify SDK のシングルトン

```js
import "@shopify/shopify-app-react-router/adapters/node"; // ← 最重要

const shopify = shopifyApp({ ... });
export const authenticate = shopify.authenticate;
```

**`adapters/node` のインポートが最重要**です。Node.js 環境には `fetch` や `crypto` などの Web 標準 API が完全には揃っていないため、このアダプターがポリフィルを提供します。

### リクエストフロー全体

```
HTTP リクエスト
    ↓
react-router-serve（Express）
    ↓
build/server/index.js
    ├─ ルートマッチング（routes.js の定義に従う）
    ├─ loader / action 実行（例: authenticate.admin(request)）
    │       ↑
    │   shopify.server.js が認証・セッション管理
    ↓
entry.server.jsx の handleRequest()
    ├─ addDocumentResponseHeaders() で Shopify ヘッダー追加
    └─ renderToPipeableStream() で root.jsx → Outlet → 各ルートを SSR
    ↓
HTML ストリームとして Response を返す
```

---

## 3. パッケージの役割と依存関係

### `@react-router/node` — ユーティリティライブラリ

**アプリコードからインポートして使う**ライブラリです（サーバーではありません）。

```js
// エクスポート一覧
createReadableStreamFromReadable; // Node.js Readable → Web ReadableStream
createRequestListener; // Node.js HTTP サーバーのリスナー生成
createFileSessionStorage; // ファイルベースのセッションストレージ
readableStreamToString;
writeAsyncIterableToWritable;
writeReadableStreamToWritable;
```

**役割：Node.js のストリーム（`Readable`/`Writable`）と Web Streams API（`ReadableStream`）を橋渡しする変換ユーティリティ集。**

React Router 本体は Web Fetch API（Web 標準）を前提に設計されていますが、Node.js はそれと異なる独自のストリームを持つため、このブリッジが必要です。

このプロジェクトでは `entry.server.jsx` でインポートされます：

```js
import { createReadableStreamFromReadable } from "@react-router/node";
```

### `@react-router/express` — Express 変換アダプター

エクスポートは `createRequestHandler` **1つだけ**で、やっていることはシンプルです：

```js
function createRequestHandler({ build, getLoadContext, mode }) {
  // ① react-router 本体のハンドラーを生成（Web Fetch API ベース）
  let handleRequest = createRequestHandler(build, mode);

  // ② Express ミドルウェア関数 (req, res, next) => {} を返す
  return async (req, res, next) => {
    // ③ Express の req → Web Request に変換
    let request = createRemixRequest(req, res);
    // ④ React Router（Web 標準）でリクエスト処理
    let response = await handleRequest(request, loadContext);
    // ⑤ Web Response → Express の res に変換して送信
    await sendRemixResponse(res, response);
  };
}
```

変換の際に `@react-router/node` のストリームユーティリティを使います：

```js
// リクエストボディ: Node.js Readable → Web ReadableStream
body: createReadableStreamFromReadable(req);

// レスポンスボディ: Web ReadableStream → Node.js Writable (res)
await writeReadableStreamToWritable(nodeResponse.body, res);
```

### `@react-router/serve` — CLI サーバーバイナリ

`#!/usr/bin/env node` から始まる**プロセスとして起動するCLIバイナリ**です。実体は147行のシンプルな Express ラッパーです。

#### middleware の登録順序（実際のコード通り）

```javascript
app.disable("x-powered-by");                  // X-Powered-By ヘッダー除去（セキュリティ）
app.use(compression());                        // ① レスポンス圧縮
app.use("/assets", static(assets, {            // ② ハッシュ付き静的ファイル
  immutable: true, maxAge: "1y"               //    → 1年間キャッシュ固定
}));
app.use(publicPath, static(buildDir));         // ③ その他ビルド成果物
app.use(static("public", { maxAge: "1h" }));  // ④ public/ ディレクトリ
app.use(morgan("tiny"));                       // ⑤ HTTP ログ
app.all("*", createRequestHandler({ build })); // ⑥ React Router 終端ハンドラ
```

各リクエストは上から順に評価され、先にキャッチされた middleware で終端します：

```
GET /assets/app.a1b2c3.js → ② でキャッチ（終端、1年キャッシュ）
GET /favicon.ico           → ③ でキャッチ（終端）
GET /logo.png              → ④ でキャッチ（終端、1時間キャッシュ）
GET /products              → ①〜④ スルー → ⑥ React Router で SSR（終端）
```

#### 各 middleware の仕組み

**① `compression`：レスポンス圧縮**

```
リクエスト (Accept-Encoding: gzip) を受け取る
  ↓
res.write / res.end を Node.js zlib.createGzip() の Transform Stream で上書き
  ↓
レスポンス (Content-Encoding: gzip) として透過的に圧縮して送信
```

クライアントが `Accept-Encoding` を送っていなければ素通りします。

**② `express.static`（ハッシュ付きアセット）：Cache Busting**

Vite がビルドするファイル（`app.a1b2c3.js` など）はコンテンツハッシュ付きのファイル名になります。内容が変わればファイル名も変わるため、`immutable: true` + `maxAge: "1y"` で 1年間キャッシュしても安全です。ファイルが存在しなければ `next()` を呼んで次へ渡します。

**⑤ `morgan("tiny")`：非同期ログ**

`morgan` は `res.end` をモンキーパッチし、レスポンスが実際に送信されたタイミングでログを出力します。⑥ より**前**に登録されていても、実際の出力は ⑥ の実行完了後になります。

```
POST /chat 200 1234 - 89.123 ms
```

**⑥ `app.all("*", createRequestHandler(...))`：終端ハンドラ**

`app.all("*")` は全 HTTP メソッド × 全パスにマッチします。`createRequestHandler` はレスポンスを送信して終端し、`next()` を呼びません（エラー時のみ `next(error)`）。Express の `app.use()` で登録していますが、実態は**終端ハンドラ**です。

#### その他の実装詳細

**ポート解決：**

```javascript
let port = parseNumber(process.env.PORT)
  ?? await getPort({ port: 3000 }); // 3000 が使用中なら 3001, 3002... と探す
```

**グレースフルシャットダウン：**

```javascript
["SIGTERM", "SIGINT"].forEach((signal) => {
  process.once(signal, () => server?.close(console.error));
});
```

**RSC ビルドの自動検出：** ビルドモジュールが `default.fetch` 関数を持つ場合（Cloudflare Workers 形式）は `@react-router/express` ではなく `@mjackson/node-fetch-server` の `createRequestListener` で処理します。

### 依存関係の全体像

```
react-router-serve（CLI バイナリ）
  └─ 内部で @react-router/express を使用
       └─ Express req/res ↔ Web Request/Response の変換
       └─ @react-router/node のストリームユーティリティを使用
            └─ Node.js Readable/Writable ↔ Web ReadableStream の変換
                 └─ react-router 本体（Web Fetch API のみ知っている）
```

---

## 4. Cloudflare Workers への移行について

### Node アダプターは Cloudflare では動かない

`@react-router/node` は Node.js 専用です。Cloudflare Workers は V8 isolates のみで動き、`stream`、`fs`、`crypto` などの Node.js API がないため使えません。

|                          | Node.js          | Cloudflare Workers             |
| ------------------------ | ---------------- | ------------------------------ |
| ランタイム               | V8 + Node.js API | V8 isolates のみ               |
| `stream`, `fs`, `crypto` | あり             | なし                           |
| 実行モデル               | 長期プロセス     | リクエストごとの短命な isolate |

### Cloudflare では全部不要

```
Node.js 環境                        Cloudflare Workers 環境
──────────────────────────────────  ──────────────────────────────────
@react-router/serve    （CLI）       不要
@react-router/express  （変換層）    不要
@react-router/node     （Stream変換）不要

react-router 本体                   react-router 本体
                                    @react-router/cloudflare
```

Cloudflare Workers は Web Fetch API がネイティブのため変換が一切不要です：

```ts
// Cloudflare の server.ts
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext) {
    const handler = createRequestHandler(
      () => import("virtual:react-router/server-build"),
    );
    return handler(request, { cloudflare: { env, ctx } });
  },
};
```

### このプロジェクト固有の障壁

React Router アダプターの変更だけでは済みません：

| 現状                                     | Cloudflare Workers での問題                    |
| ---------------------------------------- | ---------------------------------------------- |
| Prisma + SQLite                          | ファイルシステムなし → D1 や外部 DB に変更必要 |
| `@react-router/node`                     | `@react-router/cloudflare` に変更必要          |
| `shopify-app-react-router/adapters/node` | Cloudflare 向けアダプターが Shopify 非公式     |
| `PassThrough` ストリーム                 | Web Streams API に書き換え必要                 |

Shopify の推奨ホスティングは Google Cloud Run、Fly.io、Render などの **Node.js 環境**です。

### Hono との組み合わせ

公式の `@react-router/hono` は存在しませんが、Hono は Web Fetch API をネイティブで使うため `createRequestHandler` を直接組み合わせられます：

```ts
import { Hono } from "hono";
import { createRequestHandler } from "react-router";

const app = new Hono();
const handler = createRequestHandler(
  () => import("virtual:react-router/server-build"),
);

app.all("*", (c) => handler(c.req.raw));
export default app;
```

---

## 5. root.jsx がどう差し込まれるか

`entry.server.jsx` に `root.jsx` の `App` コンポーネントは明示的にインポートされていません。これは `reactRouter()` Vite プラグインと仮想モジュールの仕組みによるものです。

### ビルド時の仕組み

```
reactRouter() Vite プラグインの仕事
  1. app/root.jsx を自動発見
  2. app/routes.js の flatRoutes() でルートファイルを全スキャン
  3. これらをまとめた仮想モジュールを生成

virtual:react-router/server-build（Vite の仮想モジュール）
  ├── entry: { module: entry.server.jsx の handleRequest }
  ├── routes: {
  │     root: { id: "root", module: root.jsx の App など }
  │     "routes/app": { id: "routes/app", module: ... }
  │     ...すべてのルート
  │   }
  └── manifest: { ルートの URL → ID のマッピング }
```

`npm run build` で生成される `build/server/index.js` がこの仮想モジュールのコンパイル結果です。

### `virtual:react-router/server-build` とは何か

`virtual:` プレフィックスは Vite の**仮想モジュール**の慣例です。ディスク上に実ファイルとして存在せず、`reactRouter()` Vite プラグインが `resolveId` フックで捕捉し、メモリ上でコードを合成して `import` できるようにしたモジュールです。

**開発時と本番時の対称性：**

```
開発時                                      本番時
──────────────────────────────────────      ──────────────────────────────────────
viteDevServer.ssrLoadModule(                npm run build
  "virtual:react-router/server-build"         ↓
)                                           build/server/index.js（コンパイル済み）
  ↑ Vite がメモリで合成、HMR 付き              ↑ 同じ内容がファイルとして出力される
```

`build/server/index.js` は「仮想モジュールをビルドしたもの」であり、両者は同じものの開発/本番形態です。どちらも `{ routes, entry, assets }` を export する通常の ESM モジュールで、サーバープロセスではありません。

### Vite と React Router の責務分離

`virtual:react-router/server-build` は **「地図（マニフェスト）」** です。この地図を介して Vite と React Router は完全に分離されています：

```
【Vite の責務】                          【React Router の責務】
──────────────────────────────────       ──────────────────────────────────
app/root.jsx                             virtual:react-router/server-build
app/routes/products.jsx   → スキャン →   ├─ routes: {
app/routes/cart.jsx          バンドル     │   root:              ┐
app/entry.server.jsx                     │   "routes/products": │ URL → モジュール
                                         │   "routes/cart":     ┘ の「地図」
「コードをどうまとめるか」を知っている    │ }
                                         ├─ entry: { handleRequest }
                                         └─ assets: { ... }
                                              ↓
                                         createRequestHandler に渡す
                                              ↓
                                         リクエスト /products が来た
                                           → routes["routes/products"] を参照
                                           → loader() 実行
                                           → SSR → HTML 返却

                                         「リクエストをどう処理するか」を知っている
```

| 担当 | 役割 |
|------|------|
| **Vite** | ファイルをスキャン・変換・バンドルして「地図」を作る |
| **virtual module** | その「地図」の実体（routes + entry + assets の集合） |
| **createRequestHandler** | 地図を受け取り、URL に応じて該当ルートを呼ぶ |

React Router 本体は「Web Fetch API しか知らない」純粋なルーターです。Node.js か Cloudflare Workers かを気にしません。`createRequestHandler` に渡す `build`（= virtual module の実体）が変わるだけで、同じ React Router がどのランタイムでも動きます。

### リクエスト時の仕組み

```
HTTP リクエスト到着
  ↓
React Router コアが以下を構築:
  reactRouterContext = {
    manifest: { ... },
    routeModules: {
      root: {
        default: App,           // ← root.jsx の App コンポーネント
      },
      "routes/app": { ... },
    },
    staticHandlerContext: {
      matches: [...],           // マッチしたルート
      loaderData: { ... },      // 各 loader の戻り値
    }
  }
  ↓
entry.server.jsx の handleRequest(request, ..., reactRouterContext) を呼ぶ
  ↓
<ServerRouter context={reactRouterContext} url={request.url} />
  ↓
ServerRouter が context.routeModules["root"].default = App を取り出して
コンポーネントツリーを構築・レンダリング
```

`root.jsx` は `reactRouterContext.routeModules` の中にすでに含まれているため、明示的な import は不要です。むしろ書いてはいけない（二重バンドルになる）。

---

## 6. entry.client.jsx がない理由

`entry.client.jsx` は React Router v7 のフレームワークモードでは**省略可能**です。省略した場合、Vite プラグインがデフォルト実装を自動で注入します。

### デフォルトの entry.client（自動注入されるもの）

```tsx
import { startTransition, StrictMode } from "react";
import { hydrateRoot } from "react-dom/client";
import { HydratedRouter } from "react-router/dom";

startTransition(() => {
  hydrateRoot(
    document,
    <StrictMode>
      <HydratedRouter /> {/* SSR された HTML に React を接着 */}
    </StrictMode>,
  );
});
```

つまり**このプロジェクトはSSRのみではなく、SSR + hydration の通常のReact Routerアプリ**です。

`entry.client.jsx` を書かないのは「デフォルトの hydration 動作で十分だから」。`entry.server.jsx` は明示的に書いている — Shopify 必須ヘッダーのカスタマイズが必要なためです。

`react-router reveal entry.client` コマンドでデフォルトをファイルとして書き出すこともできます。

### hydration 前後の違い

```
hydration 前（HTML 受信直後）        hydration 後（JS 実行済み）
──────────────────────────────     ──────────────────────────────
✅ HTML は表示されている              ✅ React イベントが動く
✅ useLoaderData の値は               ✅ useState, useEffect が動く
   HTML に焼き込まれている             ✅ App Bridge が postMessage 開始
✅ apiKey は HTML に含まれている       ✅ Shopify Admin との通信が可能
❌ App Bridge は未初期化              ✅ ナビゲーション等の UI が使える
❌ ボタンクリック等は無反応
```

---

## 7. useLoaderData とクライアントへのデータ渡し

### loader の戻り値はクライアントに渡る

React Router の loader データは、サーバーで実行されたあとクライアントに自動的に渡されます：

```
サーバー（loader 実行）
  process.env.SHOPIFY_API_KEY → "abc123"
  return { apiKey: "abc123" }
          ↓
  HTML に JSON として埋め込まれる（serverHandoffString）
  <script>window.__reactRouterContext = { loaderData: { "routes/app": { apiKey: "abc123" } } }</script>
          ↓
クライアント（HydratedRouter が読み取り）
  useLoaderData() → { apiKey: "abc123" }
```

### useLoaderData はサーバー・クライアント両方で動く

`useLoaderData()` は SSR 時にも動作します。サーバーレンダリング時、React Router はサーバー側コンテキストから値を供給するため、hydration を待たずに HTML に値が焼き込まれます。

### セキュリティ：何を渡してよいか

```js
// ❌ 絶対にやってはいけない（クライアントに漏れる）
export const loader = async () => {
  return {
    apiSecret: process.env.SHOPIFY_API_SECRET, // 秘密鍵
    accessToken: session.accessToken, // アクセストークン
  };
};

// ✅ 安全（公開情報のみ）
export const loader = async () => {
  return {
    apiKey: process.env.SHOPIFY_API_KEY, // Client ID（公開情報）
  };
};
```

`SHOPIFY_API_KEY` は OAuth の Client ID であり、本質的に公開情報です（URL パラメーターにも含まれる）。`SHOPIFY_API_SECRET` は絶対にクライアントに渡してはいけません。

---

## 8. context とは何か

「context」は登場する場所によって全く異なるものを指します。

### ① EntryContext（entry.server.jsx の reactRouterContext）

React Router フレームワークが**内部的に組み立てる**オブジェクト。開発者が直接触るものではありません：

```ts
type EntryContext = {
  manifest: AssetsManifest;      // CSS/JS アセットのマッピング
  routeModules: RouteModules;    // { [routeId]: { default, loader, action, ... } }
  staticHandlerContext: {
    matches: RouteMatch[];       // 現在の URL にマッチしたルート一覧
    loaderData: Record<...>;     // 各 loader の戻り値
    errors: Record<...> | null;
  };
  serverHandoffString: string;   // クライアントへの引き継ぎデータ（JSON）
}
```

`<ServerRouter context={reactRouterContext}>` に渡すことで SSR 時のコンポーネントツリーが構築されます。

### ② AppLoadContext（loader/action の context 引数）

```js
export async function loader({ request, params, context }) {
  //                                              ↑ AppLoadContext
}
```

**サーバー（HTTP サーバー）から React Router へデータを橋渡し**するための仕組みです。`getLoadContext` で組み立てます：

```js
// Express の場合
app.use(
  createRequestHandler({
    getLoadContext(req, res) {
      return { db: myDatabase, user: req.user };
    },
  }),
);

// Cloudflare Workers の場合
handler(request, { cloudflare: { env, ctx } });

// Hydrogen の場合（最も多機能）
getLoadContext: () => ({
  storefront, // Storefront API クライアント
  customerAccount, // Customer Account API クライアント
  session,
  env,
});
```

### このプロジェクトが context を使わない理由

```js
export const loader = async ({ request }) => {
  //                    ↑ context を使っていない
  await authenticate.admin(request); // shopify.server.js が直接担当
};
```

`shopify.server.js` の `authenticate` がリクエストから直接セッションを取得するため、`context` 経由でサーバーオブジェクトを渡す必要がありません。

### ③ Cloudflare Workers の ExecutionContext（ctx）

Cloudflare 固有。`ctx.waitUntil()` でレスポンス後も非同期処理を継続できます。

---

## 9. App Bridge の役割

### 埋め込みアプリは iframe で動く

```
Shopify Admin（親ウィンドウ）
┌─────────────────────────────────────────┐
│  Shopify のナビゲーション・ヘッダー等    │
│  ┌─────────────────────────────────┐   │
│  │                                 │   │
│  │  あなたのアプリ（iframe）         │   │
│  │  ← ここで React Router が動く    │   │
│  │                                 │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

`iframe` と親ウィンドウは別々の JavaScript 実行環境のため、直接関数を呼んだり DOM を触ったりできません。

### App Bridge = postMessage のラッパー

> "App Bridge components don't render as part of the app's component hierarchy.
> They're **React-like wrappers around JavaScript messages** that communicate with the Shopify admin.
> **The Shopify admin does the UI rendering.**"（Shopify 公式ドキュメント）

```
あなたのアプリ（iframe）                  Shopify Admin（親ウィンドウ）
──────────────────────────────────        ──────────────────────────────────
App Bridge が                             postMessage を受け取って
window.parent.postMessage(msg) ────────→  Shopify Admin が実際に UI を描画
```

### App Bridge ができること

**ナビゲーションメニュー：**

```jsx
<s-app-nav>
  <s-link href="/app">Home</s-link> {/* Shopify Admin 左サイドバーに表示 */}
</s-app-nav>
```

**タイトルバー・アクション：**

```jsx
<ui-title-bar title="My App">
  <button variant="primary">Save</button> {/* Admin トップバーに表示 */}
</ui-title-bar>
```

**セッショントークンの取得（認証の要）：**

```js
const token = await shopify.idToken(); // JWT 形式トークン
// → バックエンドへのリクエスト認証に使う
// Authorization: Bearer <token>
```

現代のブラウザはサードパーティ Cookie（iframe 内の Cookie）をデフォルトでブロックするため、埋め込みアプリではセッショントークンが認証の代替手段となります。

**Direct API Access：**

```js
// App Bridge 初期化後、Shopify Admin GraphQL を直接呼べる
const res = await fetch("shopify:admin/api/2025-04/graphql.json", {
  method: "POST",
  body: JSON.stringify({ query: `{ shop { name } }` }),
});
// App Bridge が自動で認証ヘッダーを付与
```

### apiKey が必要な理由

```
AppProvider embedded apiKey="abc123" が初期化
  ↓
window.parent.postMessage({
  type: "APP_BRIDGE_INIT",
  apiKey: "abc123",   // どのアプリかを識別（Client ID = 公開情報）
}, "*")
  ↓
Shopify Admin が検証 → 通信チャネルを確立
```

### JS なしでは動かない理由

```
SSR で生成される HTML
  └─ <s-app-nav> の DOM は存在する
  └─ HTML 構造は表示される

JS（App Bridge）ロード後
  └─ window.parent.postMessage() が実行される
  └─ Shopify Admin がナビゲーションを描画する
  └─ shopify.idToken() が使えるようになる
  └─ セッショントークンによる認証が機能する
```

HTML に `<s-app-nav>` が存在しても、JS なしでは Shopify Admin に「ここにナビゲーションを表示してほしい」というメッセージが送られないため、親ウィンドウ側は何も表示しません。

---

## 10. Hydrogen（Cloudflare Workers）での server.ts

Hydrogen プロジェクトの `server.ts` には `compression`・`express.static`・`morgan` が存在しません。これは Node.js と Cloudflare Workers のインフラアーキテクチャの根本的な違いによるものです。

### Node.js vs Cloudflare Workers の役割分担

```
Node.js（react-router-serve）              Cloudflare Workers（Hydrogen/Oxygen）
──────────────────────────────────────     ──────────────────────────────────────
すべてのリクエスト → Express（1プロセス）   HTTP リクエスト
  ├─ 静的ファイル: express.static          → Cloudflare エッジネットワーク
  │   がディスクから読んで送信                 ├─ /assets/*.js, *.css
  ├─ 圧縮: compression middleware が            │   → Shopify CDN へ（Worker を通らない）
  │   手動で gzip 変換                         ├─ 圧縮: Cloudflare が自動でエッジ実施
  ├─ ログ: morgan が手動でアクセスログ          ├─ ログ: Cloudflare Dashboard に組み込み
  └─ 動的ルート: createRequestHandler          └─ 動的ルート（/products, /cart 等）
                                                      ↓
                                               Cloudflare Worker
                                               → server.ts の fetch() 実行
```

**Node.js サーバーは「何でも屋」**（静的配信・圧縮・ログを自前で実装）。
**Workers は「SSR 専用」**（静的ファイルは Worker に到達しないため `express.static` の出番がない）。

### 静的アセットはどこへ行くのか

`shopify hydrogen deploy` を実行すると：

```
ビルド成果物
  ├─ dist/worker/index.js  → Cloudflare Worker にアップロード（SSR 担当）
  └─ dist/client/          → Shopify CDN にアップロード（静的ファイル担当）
       ├─ assets/app.a1b2c3.js      ← shopify.cdn.com から配信
       └─ assets/style.d4e5f6.css
```

ブラウザが `<script src="/assets/app.a1b2c3.js">` をリクエストしても、それは Cloudflare Worker ではなく Shopify CDN が直接返します。Worker はそのリクエストを一切見ません。

### `virtual:react-router/server-build` の扱いも違う

```
Node.js（react-router-serve）              Cloudflare Workers（Hydrogen）
──────────────────────────────────────     ──────────────────────────────────────
// ランタイムに動的 import                 // ビルド時に Worker バンドルへ織り込む
import(pathToFileURL(buildPath))           import('virtual:react-router/server-build')
  ↑ ディスク上の別ファイルを                 ↑ Vite が全ルートモジュールを
    実行時に読み込む                           Worker JS に静的バンドル

Worker と build は別ファイル              Worker と routes は1ファイル
（実行時に接続）                           （ビルド時に接続済み）
```

Cloudflare Workers にはファイルシステムがないため、全ルートモジュールは Worker バンドルにビルド時に織り込まれます。

### カスタムサーバー（Express）への移行

`@react-router/serve` の代わりに自前の Express サーバーを書く場合は [`docs/custom-express-server.md`](./custom-express-server.md) を参照してください。`getLoadContext` を使って DB クライアントや認証情報を全 loader/action に注入できます。

---

## まとめ：全体像

```
ビルド時
  reactRouter() Vite プラグイン
    → root.jsx + 全ルートを仮想モジュールに束ねる
    → build/server/index.js を生成

実行時（Node.js）
  react-router-serve（@react-router/express 内包）
    → Express req/res ↔ Web Request/Response を変換（@react-router/node 使用）
    → React Router 本体（Web 標準）でルーティング・SSR
    → entry.server.jsx で Shopify ヘッダー付与 + HTML ストリーム生成
    → HTML に serverHandoffString（loader データ等）を埋め込む

ブラウザ
  HydratedRouter が hydrateRoot() で SSR 済み HTML に React を接着
    → App Bridge（JS）が Shopify Admin と postMessage 通信開始
    → useLoaderData() でサーバーの loader データが使える
```
