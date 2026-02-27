# カスタム Express サーバー

`@react-router/serve` を使わず、自前の Express サーバーをセットアップする実装例です。`@react-router/serve` が内部で行っている処理（compression・static・morgan 等）を再現しつつ、カスタムサーバーならではの拡張（`getLoadContext`・認証 middleware 等）も含みます。

---

## なぜカスタムサーバーが必要か

`@react-router/serve` では不可能で、カスタムサーバーで初めてできることがあります：

| 用途 | 説明 |
|------|------|
| `getLoadContext` | DB クライアントや認証情報を全 loader/action に注入 |
| カスタム middleware | 認証・レート制限・CORS 等を React Router の前に挟む |
| 複数アプリの共存 | 同一 Express 上で複数サービスを動かす |
| ヘルスチェックエンドポイント | `/health` 等のカスタムルートを追加 |

---

## 実装例

```typescript
// server.ts
import "react-router";

import fs from "node:fs";
import path from "node:path";
import url from "node:url";

import { createRequestHandler } from "@react-router/express";
import compression from "compression";
import express from "express";
import getPort from "get-port";
import morgan from "morgan";
import sourceMapSupport from "source-map-support";

// ソースマップサポート（エラースタックトレースを元ファイルの行番号で表示）
sourceMapSupport.install({
  retrieveSourceMap(source) {
    // file:// プロトコルの URL のみ対象
    if (!source.startsWith("file://")) return null;
    const filePath = url.fileURLToPath(source);
    const mapPath = `${filePath}.map`;
    if (!fs.existsSync(mapPath)) return null;
    return { url: source, map: fs.readFileSync(mapPath, "utf8") };
  },
});

// ---------------------------------------------------------------
// 開発 / 本番の分岐
// ---------------------------------------------------------------
const isDev = process.env.NODE_ENV !== "production";

async function createApp() {
  const app = express();

  // X-Powered-By: Express ヘッダーを除去（サーバー情報の隠蔽）
  app.disable("x-powered-by");

  // ① レスポンス圧縮（gzip / brotli）
  app.use(compression());

  if (isDev) {
    // -------------------------------------------------------
    // 開発モード：Vite をミドルウェアとして組み込む
    // HMR・React Fast Refresh が使えるようになる
    // -------------------------------------------------------
    const vite = await import("vite");
    const viteDevServer = await vite.createServer({
      server: { middlewareMode: true },
    });

    // Vite の HMR WebSocket・静的ファイル配信を委譲
    app.use(viteDevServer.middlewares);

    // ⑤ ログ（dev = メソッド・パス・ステータスが色付きで表示）
    app.use(morgan("dev"));

    // ⑥ 終端ハンドラ（開発時は仮想モジュールを毎回 ssrLoadModule）
    //    毎リクエスト呼ぶことで HMR 後の最新モジュールが使われる
    app.all(
      "*",
      createRequestHandler({
        build: () =>
          viteDevServer.ssrLoadModule("virtual:react-router/server-build"),
        getLoadContext: buildLoadContext,
      }),
    );
  } else {
    // -------------------------------------------------------
    // 本番モード：ビルド済み成果物を使用
    // -------------------------------------------------------

    // build/server/index.js をモジュールとして読み込む
    // （サーバープロセスではなく、routes + entry の集合体）
    const buildPath = path.resolve("build/server/index.js");
    const buildModule = await import(url.pathToFileURL(buildPath).href);

    const publicPath: string = buildModule.assets?.prefix ?? "/";
    const assetsBuildDir: string =
      buildModule.assetsBuildDirectory ?? "build/client";

    // ② ハッシュ付きアセット（例: app.a1b2c3.js）
    //    immutable + 1年キャッシュ = ファイル名が変わらない限り再検証不要（Cache Busting）
    app.use(
      path.posix.join(publicPath, "assets"),
      express.static(path.join(assetsBuildDir, "assets"), {
        immutable: true,
        maxAge: "1y",
      }),
    );

    // ③ その他のビルド成果物（HTML manifest 等）
    app.use(publicPath, express.static(assetsBuildDir));

    // ④ public/ ディレクトリ（favicon.ico, robots.txt 等）
    app.use(express.static("public", { maxAge: "1h" }));

    // ⑤ ログ（tiny = "METHOD URL STATUS SIZE - TIME"）
    app.use(morgan("tiny"));

    // ⑥ 終端ハンドラ（全 HTTP メソッド × 全パス）
    //    next() は呼ばず、React Router が SSR してレスポンス送信
    app.all(
      "*",
      createRequestHandler({
        build: buildModule,
        mode: process.env.NODE_ENV,
        getLoadContext: buildLoadContext,
      }),
    );
  }

  return app;
}

// ---------------------------------------------------------------
// AppLoadContext：全 loader / action の context 引数に渡るオブジェクト
// カスタムサーバーの最大の利点 — @react-router/serve では使えない
// ---------------------------------------------------------------
function buildLoadContext(
  req: express.Request,
  _res: express.Response,
): Record<string, unknown> {
  return {
    // 例: DB クライアント、認証情報、フィーチャーフラグ等を注入できる
    // db: prisma,
    // user: req.user,
  };
}

// ---------------------------------------------------------------
// サーバー起動
// ---------------------------------------------------------------
async function run() {
  const port =
    Number(process.env.PORT) ||
    (await getPort({ port: 3000 })); // 3000 が使用中なら 3001, 3002... と自動探索

  const host = process.env.HOST;
  const app = await createApp();

  const server = host
    ? app.listen(port, host, onListen)
    : app.listen(port, onListen);

  function onListen() {
    console.log(`[server] http://localhost:${port}`);
  }

  // グレースフルシャットダウン
  // SIGTERM: コンテナオーケストレータ（Kubernetes 等）が送るシグナル
  // SIGINT:  Ctrl+C
  for (const signal of ["SIGTERM", "SIGINT"] as const) {
    process.once(signal, () => {
      console.log(`[server] ${signal} received, shutting down...`);
      server.close((err) => {
        if (err) console.error(err);
        process.exit(err ? 1 : 0);
      });
    });
  }
}

run();
```

---

## middleware の登録順序と各リクエストの流れ

```
GET /assets/app.a1b2c3.js → ② でキャッチ（終端、1年キャッシュ・immutable）
GET /favicon.ico           → ③ でキャッチ（終端）
GET /logo.png              → ④ でキャッチ（終端、1時間キャッシュ）
GET /products              → ①〜④ スルー → ⑥ React Router で SSR（終端）
```

---

## カスタムサーバーならではの拡張例

```typescript
// 認証 middleware を React Router の前に挟む
app.use("/admin/*", requireAuth);

// レート制限
import rateLimit from "express-rate-limit";
app.use("/api/*", rateLimit({ windowMs: 60_000, max: 100 }));

// ヘルスチェックエンドポイント（コンテナ環境で必要）
app.get("/health", (_req, res) => res.json({ status: "ok" }));
```

```typescript
// getLoadContext で DB を全 loader に注入
function buildLoadContext(req, _res) {
  return {
    db: prisma,    // loader({ context }) → context.db.user.findMany()
    user: req.user,
    env: process.env,
  };
}

// loader 側（context 経由で受け取る）
export async function loader({ request, context }) {
  const { db, user } = context;
  const orders = await db.order.findMany({ where: { userId: user.id } });
  return { orders };
}
```

---

## `@react-router/serve` との対応表

| `@react-router/serve` の処理 | カスタムサーバーでの対応 |
|------------------------------|--------------------------|
| `app.disable("x-powered-by")` | そのまま同じ |
| `compression()` | そのまま同じ |
| `express.static(...assets, { immutable, maxAge: "1y" })` | そのまま同じ |
| `express.static(assetsBuildDir)` | そのまま同じ |
| `express.static("public", { maxAge: "1h" })` | そのまま同じ |
| `morgan("tiny")` | 開発時は `"dev"`、本番時は `"tiny"` に変更 |
| `createRequestHandler({ build })` | `getLoadContext` を追加して拡張 |
| `get-port` でポート自動探索 | そのまま同じ |
| `SIGTERM`/`SIGINT` グレースフルシャットダウン | `server.close()` で接続を待ってから終了するよう改善 |
| `source-map-support` | そのまま同じ |
| RSC ビルド自動検出 | 省略（必要なら追加） |
