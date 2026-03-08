# Node.js HTTP API vs Web Fetch API

Node.js と Web Fetch API は、HTTP リクエスト/レスポンスを扱う**2つの異なる設計体系**です。このドキュメントでは両者の違いと、現代フレームワークがなぜ Web Fetch API を選ぶのかを解説します。

---

## 目次

1. [全体像：2つの世界](#1-全体像2つの世界)
2. [Response の違い](#2-response-の違い)
3. [Request の違い](#3-request-の違い)
4. [ストリームの違い](#4-ストリームの違い)
5. [Node.js 18+ での共存](#5-nodejs-18-での共存)
6. [なぜ現代フレームワークは Web Fetch API を選ぶのか](#6-なぜ現代フレームワークは-web-fetch-api-を選ぶのか)
7. [このプロジェクトでの変換フロー](#7-このプロジェクトでの変換フロー)

---

## 1. 全体像：2つの世界

```
Node.js HTTP API（2009年〜）              Web Fetch API（2015年〜）
──────────────────────────────────       ──────────────────────────────────
http.IncomingMessage (req)               Request
http.ServerResponse (res)                Response
stream.Readable / Writable              ReadableStream / WritableStream
コールバック / イベント型                 Promise 型
Node.js 専用                             ブラウザ / Deno / Workers / Node.js 18+
```

Node.js HTTP API は Node.js の誕生とともに設計されました。Web Fetch API はブラウザの `fetch()` から始まり、サーバーサイドでも標準となりつつある新しい仕様です。

---

## 2. Response の違い

### Node.js: `http.ServerResponse`

`stream.Writable` を継承した**命令型**のオブジェクトです。プロパティをセットし、データを逐次書き込みます。

```js
// Node.js / Express
app.get('/', (req, res) => {
  res.statusCode = 200;                      // ステータスをセット
  res.setHeader('Content-Type', 'text/html'); // ヘッダーをセット
  res.write('<html>');                        // チャンクを送信（接続は開いたまま）
  res.write('<body>Hello</body>');            // 次のチャンクを送信
  res.end('</html>');                         // 最後のチャンクを送信して接続を閉じる
});
```

**特徴：**
- `res.write()` / `res.end()` は `stream.Writable` のメソッド
- 送信開始後にステータスコードやヘッダーを変更できない
- Express の `res.send()` / `res.json()` は内部で `write()` + `end()` を呼ぶ便利メソッド

```
継承チェーン:
stream.Writable
  └── http.ServerResponse    ← Node.js 標準
        └── Express の res   ← send(), json() 等を追加
```

### Web Fetch API: `Response`

コンストラクタで一括指定する**宣言型**のオブジェクトです。

```js
// Web Fetch API
const response = new Response('<html>Hello</html>', {
  status: 200,
  headers: { 'Content-Type': 'text/html' },
});
// ↑ 構築した時点で完成。あとは返すだけ
```

**特徴：**
- イミュータブル（作成後にステータスやヘッダーを変更しない前提）
- `body` には文字列、`ReadableStream`、`Blob`、`ArrayBuffer` 等を渡せる
- ストリーミングする場合は `ReadableStream` を body に渡す

```js
// ストリーミング Response
const stream = new ReadableStream({
  start(controller) {
    controller.enqueue(new TextEncoder().encode('<html>'));
    controller.enqueue(new TextEncoder().encode('Hello'));
    controller.enqueue(new TextEncoder().encode('</html>'));
    controller.close();
  },
});

new Response(stream, {
  headers: { 'Content-Type': 'text/html' },
});
```

### 比較表

| | Node.js `http.ServerResponse` | Web Fetch API `Response` |
|---|---|---|
| 設計 | 命令型（逐次書き込み） | 宣言型（一括指定） |
| 生成 | HTTP サーバーが自動生成 | `new Response()` で手動生成 |
| ストリーム | `stream.Writable` を継承 | `ReadableStream` を body に渡す |
| ステータス設定 | `res.statusCode = 200` | `new Response(body, { status: 200 })` |
| ヘッダー設定 | `res.setHeader(key, val)` | `new Response(body, { headers: {...} })` |
| データ送信 | `res.write()` + `res.end()` | body に渡すだけ（消費は読み手が行う） |

---

## 3. Request の違い

### Node.js: `http.IncomingMessage`

```js
app.post('/api', (req, res) => {
  // req は stream.Readable を継承
  console.log(req.method);              // 'POST'
  console.log(req.url);                 // '/api'
  console.log(req.headers['content-type']); // 通常のオブジェクト

  // ボディの読み取りはストリームイベントで行う
  let body = '';
  req.on('data', (chunk) => { body += chunk; });
  req.on('end', () => { console.log(body); });
});
```

### Web Fetch API: `Request`

```js
// サーバー側で受け取る場合（Cloudflare Workers 等）
export default {
  async fetch(request) {
    console.log(request.method);              // 'POST'
    console.log(request.url);                 // 'https://example.com/api'（完全な URL）
    console.log(request.headers.get('content-type')); // Headers オブジェクト

    // ボディの読み取りは await で行う
    const body = await request.json();        // or .text(), .formData(), .blob()
  }
};
```

### 比較表

| | Node.js `http.IncomingMessage` | Web Fetch API `Request` |
|---|---|---|
| URL | パス部分のみ（`/api`） | 完全な URL（`https://...`） |
| ヘッダー | 通常のオブジェクト | `Headers` オブジェクト（`.get()`, `.set()`） |
| ボディ読み取り | イベント駆動（`data` / `end`） | Promise（`await req.json()`） |
| ストリーム | `stream.Readable` を継承 | `body` プロパティが `ReadableStream` |

---

## 4. ストリームの違い

### Node.js Streams（2009年〜）

コールバック / イベント型の設計です。

```js
import { Readable, Writable, Transform, PassThrough } from 'stream';

// Readable: データの発生源
const readable = Readable.from(['chunk1', 'chunk2']);

// Writable: データの消費先
readable.pipe(process.stdout);  // pipe でつなぐ

// PassThrough: Writable + Readable を持つ素通しパイプ
const passthrough = new PassThrough();
readable.pipe(passthrough);     // 書き込み口
passthrough.pipe(process.stdout); // 読み出し口
```

```
Node.js Stream の階層:
stream.Readable   ← 読み出し専用（データ発生源）
stream.Writable   ← 書き込み専用（データ消費先）
stream.Duplex     ← Readable + Writable（双方向）
  └── stream.Transform  ← 入力を変換して出力
        └── stream.PassThrough ← 変換なしで素通し
```

### Web Streams API（2015年〜）

Promise 型の設計です。

```js
// ReadableStream: データの発生源
const readable = new ReadableStream({
  start(controller) {
    controller.enqueue('chunk1');
    controller.enqueue('chunk2');
    controller.close();
  },
});

// 消費: Reader で読む
const reader = readable.getReader();
const { value, done } = await reader.read(); // Promise ベース

// WritableStream: データの消費先
const writable = new WritableStream({
  write(chunk) {
    console.log(chunk);
  },
});

// pipe でつなぐ
await readable.pipeTo(writable);

// TransformStream: 入力を変換して出力（Duplex に相当）
const transform = new TransformStream({
  transform(chunk, controller) {
    controller.enqueue(chunk.toUpperCase());
  },
});
```

### 比較表

| | Node.js Streams | Web Streams API |
|---|---|---|
| 読み出し | `stream.Readable` | `ReadableStream` |
| 書き込み | `stream.Writable` | `WritableStream` |
| 変換 | `stream.Transform` | `TransformStream` |
| 素通し | `stream.PassThrough` | `TransformStream`（変換なし） |
| 双方向 | `stream.Duplex` | なし（TransformStream で代替） |
| 接続方法 | `.pipe(destination)` | `.pipeTo(writable)` / `.pipeThrough(transform)` |
| 非同期モデル | イベント（`data`, `end`, `error`） | Promise（`await reader.read()`） |
| バックプレッシャー | `highWaterMark` + `pause()`/`resume()` | 組み込み（pull ベース） |
| 歴史 | 2009年〜（Node.js v0.x） | 2015年〜（WHATWG 仕様） |

### Node.js PassThrough が必要な理由

このプロジェクトの `entry.server.jsx` では、React の `renderToPipeableStream`（Node.js Stream 出力）と Web Fetch API の `Response`（Web ReadableStream を期待）を繋ぐために `PassThrough` が必要です。

```
renderToPipeableStream
    │ pipe(body)        ← Node.js Writable として書き込む
    ▼
PassThrough (body)      ← Writable + Readable を一体化
    │
    ▼ createReadableStreamFromReadable(body)
Web ReadableStream      ← Node.js Readable → Web ReadableStream に変換
    │
    ▼
new Response(stream)    ← Web Fetch API の Response に渡す
```

詳細は [react-router-architecture.md セクション 11](./react-router-architecture.md#11-entryserverjsx-の-ssr-ストリーミング詳解) を参照。

---

## 5. Node.js 18+ での共存

Node.js 18 以降、Web Fetch API（`fetch`, `Request`, `Response`, `Headers`）がグローバルに追加されました。ただし従来の Node.js HTTP API も**そのまま残っています**。

```
Node.js 18+ のグローバル空間
├── http.IncomingMessage / http.ServerResponse  ← 従来の Node.js HTTP API
├── Request / Response / Headers                ← Web Fetch API（18+ で追加）
├── stream.Readable / Writable                  ← 従来の Node.js Streams
├── ReadableStream / WritableStream             ← Web Streams API（18+ で追加）
└── 両者は互換性なし（変換が必要）
```

### なぜ互換性がないのか

設計思想が根本的に異なるためです。

| 観点 | Node.js HTTP API | Web Fetch API |
|------|------------------|---------------|
| Response の生成 | サーバーが自動生成。開発者は `write()` で書き込む | 開発者が `new Response()` で手動生成 |
| ストリーム規格 | Node.js 独自の `stream` モジュール | WHATWG Web Streams 仕様 |
| 非同期モデル | コールバック / イベント | Promise / async-await |
| 設計時期 | 2009年（Web 標準が未成熟） | 2015年〜（Web 標準として策定） |

Node.js が Web Fetch API を追加したのは Deno や Cloudflare Workers との互換性を高めるためですが、Express のような既存エコシステムは `http.ServerResponse` に依存しているため、2つの API が並存する状態になっています。

### 変換ユーティリティ

`@react-router/node` が提供する変換関数：

```js
// Node.js Readable → Web ReadableStream
import { createReadableStreamFromReadable } from '@react-router/node';

// Web ReadableStream → Node.js Writable に書き込む
import { writeReadableStreamToWritable } from '@react-router/node';
```

---

## 6. なぜ現代フレームワークは Web Fetch API を選ぶのか

### ランタイム非依存性

Web Fetch API を内部インターフェースにすることで、アダプターを差し替えるだけで複数環境に対応できます。

```
                    Web Fetch API（共通インターフェース）
                    Request → handler → Response
                         ↑              ↓
┌────────────────────────┼──────────────┼────────────────────────┐
│ Node.js                │              │                        │
│  Express req/res ──→ 変換 ──→    ←── 変換 ──→ Express res    │
│  (@react-router/node)                                          │
├────────────────────────┼──────────────┼────────────────────────┤
│ Cloudflare Workers     │              │                        │
│  fetch(request) ──→ そのまま ──→ ←── そのまま ──→ return     │
│  (変換不要)                                                     │
├────────────────────────┼──────────────┼────────────────────────┤
│ Deno                   │              │                        │
│  Deno.serve(request) → そのまま → ←── そのまま ──→ return     │
│  (変換不要)                                                     │
└────────────────────────┴──────────────┴────────────────────────┘
```

### 採用しているフレームワーク

| フレームワーク | 内部インターフェース | Node.js アダプター |
|---|---|---|
| React Router v7 / Remix | Web Fetch API | `@react-router/node` |
| SvelteKit | Web Fetch API | `@sveltejs/adapter-node` |
| Hono | Web Fetch API | `@hono/node-server` |
| Astro | Web Fetch API | `@astrojs/node` |
| Next.js (App Router) | Web Fetch API ベース | 組み込み |

### Web Standard First の利点

```
1. ポータビリティ     — 同じコードが Node.js, Deno, Cloudflare Workers で動く
2. 学習の再利用       — ブラウザの fetch / Response / ReadableStream と同じ API
3. テストの容易さ     — new Request() / new Response() で単体テスト可能
4. エコシステム収束   — ランタイムごとの分断が解消されていく
```

---

## 7. このプロジェクトでの変換フロー

このプロジェクトは Node.js (Express) 上で React Router v7 を動かしているため、以下の変換が発生します。

### リクエスト側（Express req → Web Request）

`@react-router/express` の `createRequestHandler` が担当：

```
Express の req (http.IncomingMessage)
    ↓ @react-router/express 内部
    │  url: req.protocol + '://' + req.hostname + req.url  → 完全 URL に変換
    │  method: req.method
    │  headers: new Headers(req.headers)
    │  body: createReadableStreamFromReadable(req)  → Node.js Readable → Web ReadableStream
    ↓
Web Request オブジェクト
    ↓
handleRequest(request, ...) に渡される
```

### レスポンス側（Web Response → Express res）

```
handleRequest() が返す Web Response
    ↓ @react-router/express 内部
    │  res.statusCode = response.status
    │  response.headers.forEach → res.setHeader()
    │  writeReadableStreamToWritable(response.body, res)
    │    → Web ReadableStream → Node.js Writable (res) に書き込む
    ↓
Express の res (http.ServerResponse) でクライアントに送信
```

### SSR レンダリング内部（entry.server.jsx）

```
renderToPipeableStream()
    │ pipe(body)                  ← React が Node.js PassThrough に HTML を書く
    ↓
PassThrough (body)
    │ createReadableStreamFromReadable(body)
    ↓
Web ReadableStream (stream)
    │ new Response(stream, { ... })
    ↓
Web Response                      ← React Router に返す
```

### 全体の変換回数

```
リクエスト:  Express req  ──[変換①]──→  Web Request
SSR:        React HTML   ──[変換②]──→  Web ReadableStream → Web Response
レスポンス: Web Response  ──[変換③]──→  Express res

合計 3 回の変換が発生（Cloudflare Workers では 0 回）
```

この変換コストは、Web 標準に統一することでランタイム非依存のコードを実現するためのトレードオフです。
