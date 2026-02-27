# Shopify App 認証・セッション管理 詳細技術リファレンス

このドキュメントは、Shopify 埋め込みアプリにおける認証フロー、トークンの種類、およびセキュリティ管理の仕組みを詳細にまとめたものです。

---

## 1. Shopify における「セッション」の定義

Shopify アプリ開発では、混同しやすい2種類のセッションが存在します。

| セッションの名称             | 主体         | 実体             | 特徴                                                         |
| :--------------------------- | :----------- | :--------------- | :----------------------------------------------------------- |
| **Admin ログインセッション** | マーチャント | ブラウザクッキー | 管理画面そのものに入るためのもの。                           |
| **アプリ Session Token**     | アプリ       | 署名付き JWT     | App Bridge が管理画面から取得する身分証明書。1分で使い捨て。 |

### Session Token (JWT) の取得フロー

1.  Admin画面の中でアプリが開かれる。
2.  `App Bridge` (JavaScript) が `shopify.idToken()` 等を実行し、Shopify サーバーから JWT を取得。
3.  アプリはこの JWT を HTTP ヘッダー（`Authorization: Bearer <JWT>`）に含めてバックエンドへ送信する。

---

## 2. Access Token の種類と Token Exchange

Session Token 自体には API を操作する権限がありません。サーバー側で **Access Token** に交換（Token Exchange）する必要があります。

### トークンの2つのモード

1.  **Online Access Token**:
    - **紐付け**: 特定の管理画面ユーザー（店長またはスタッフ）。
    - **期限**: ログアウト時、または最大24時間。
    - **用途**: フロントエンドの操作に対するリアルタイムなレスポンス。
    - **特性**: 操作している**ユーザーの権限を継承**する。
2.  **Offline Access Token**:
    - **紐付け**: ショップ全体（管理画面ユーザーがいなくても有効）。
    - **期限**:
      - **従来型**: 無期限。
      - **2025年12月以降の仕様**: Access Token (60分) + Refresh Token (90日間)。
    - **用途**: Webhook 処理、定期実行バッチ、バックグラウンド処理。
    - **特性**: **アプリが許可された全権限**を行使できる。

---

## 3. `authenticate.admin(request)` の内部プロセス

プロジェクトの `loader` 等で呼ばれるこのメソッドは、以下の4ステップを完全に自動化（カプセル化）しています。

```javascript
// app/routes/app.jsx 等
export const loader = async ({ request }) => {
  const { admin, session } = await authenticate.admin(request);
  // ...
};
```

1.  **JWT 検証**: `request` ヘッダーの署名を `SHOPIFY_API_SECRET` で検証。
2.  **セッション検索**: 受取った Shop ドメインをキーに、`PrismaSessionStorage` (DB) から Access Token を検索。
3.  **Token Exchange**: DB にない、または有効期限切れの場合、Shopify へトークン交換をリクエストし、新しいトークンを DB に保存。
4.  **クライアント生成**: 有効なトークンを内部に保持した `admin.graphql` 等を返し、開発者が生のトークンを扱わずに済むようにする。

---

## 4. なぜ DB（Session Storage）への保存が必要か

JWT (Session Token) は「短期的な証明」に過ぎないため、以下の理由で Access Token の永続化が必要です。

- **Webhook / Background Jobs**: ブラウザが開いていない（JWT が送られてこない）状況で API を叩くには、DB に保存された **Offline Token** が不可欠。
- **パフォーマンス**: 全リクエストでトークン交換を行うと遅延が発生するため、DB にキャッシュして再利用する。
- **権限の維持**: `shopify.unauthenticated.admin(shopDomain)` を実行することで、ショップドメインをキーに DB からオフライン権限を復元できる。

---

## 5. セキュリティリスクと暗号化 (Encryption at Rest)

### デフォルトの状態

Shopify CLI が生成したデフォルトの状態では、DB 内の Access Token は**平文（暗号化なし）**で保存されています。

### パブリックアプリでの推奨対策

1.  **暗号化の追加**: DB 流出時に備え、保存直前に `encrypt`、取得直後に `decrypt` を行うラッパーを実装する。
2.  **キー管理**: 暗号化キーは DB とは別の `.env` や KMS (Key Management Service) で管理する。

### 実装イメージ（暗号化ラッパー）

```javascript
// 概念的な実装例
const secureStorage = {
  ...prismaStorage,
  storeSession: async (session) => {
    session.accessToken = encrypt(session.accessToken, process.env.SECRET_KEY);
    return prismaStorage.storeSession(session);
  },
  loadSession: async (id) => {
    const session = await prismaStorage.loadSession(id);
    if (session)
      session.accessToken = decrypt(
        session.accessToken,
        process.env.SECRET_KEY,
      );
    return session;
  },
};
```

---

## 6. なぜ Online と Offline を使い分けるのか（設計思想）

一見すると「常に Offline Token を使えば楽ではないか」と思えますが、実務上は以下の理由で厳格に使い分けられます。

### A. スタッフ権限の尊重 (RBAC: Role Based Access Control)

Shopify ストアには多くのスタッフがおり、個別に「顧客は見れるが注文は見れない」といった権限が設定されています。

- **Online Token の場合**: 操作しているスタッフの権限が反映されます。アプリが「顧客編集権限」を持っていても、操作者がその権限を持たなければ、API 呼び出し時に **403 Forbidden** エラーが返ります。
- **`authenticate.admin` の挙動**: このメソッド自体のパスは**通ります**（認証には成功するため）。しかし、その後の `admin.graphql` や `admin.rest` による API リクエストが、ユーザーの権限不足によって失敗します。
- **メリット**: アプリ側で複雑な権限チェックを実装せずとも、Shopify の標準権限に相乗りできます。

### B. 特殊なケース：アプリ権限 (Offline) でのバイパス

もしシステムの都合上、スタッフ個人の権限に関わらず強制的に処理を完了させる必要がある場合は、意図的に **Offline Token** を取得するコードを書くことができます。

- **実装例**: `const { admin } = await shopify.unauthenticated.admin(shopDomain);`
- **注意点**: これはショップオーナーがスタッフに課した制限を無視することになるため、ログ記録などのシステム的な必要性がある場合に限定すべき「禁じ手」に近い手法です。

### C. セキュリティと追跡性

- **最小特権の原則**: ユーザーがいる時は、そのユーザーの権限内（Online）で動くのが安全です。
- **監査ログ**: Shopify 側のログに「誰が」操作したかが正確に残ります（Offline だと常に「アプリ」が実行したことになります）。
- **不正利用防止**: Online Token は短命なため、万が一漏洩しても被害が限定的です。一方、Offline Token は強力なため、Webhook などの保護されたエンドポイント以外で安易に使用すべきではありません。

---

## 8. 埋め込み (Embedded) vs 非埋め込み (Standalone)

`authenticate.admin` は両方のモードをサポートしていますが、**認証のソース**が異なります。

| アプリの種類         | 認証メカニズム          | 設定 (`shopify.server.js`)         |
| :------------------- | :---------------------- | :--------------------------------- |
| **埋め込みアプリ**   | **Session Token (JWT)** | `isEmbeddedApp: true` (デフォルト) |
| **非埋め込みアプリ** | **ブラウザクッキー**    | `isEmbeddedApp: false`             |

### 非埋め込みアプリの実装方法と認証フロー

非埋め込み（スタンドアロン）版では、セッション管理が **ブラウザクッキー** に依存します。実装とフローは以下の通りです。

#### A. サーバー設定 (`app/shopify.server.js`)

`isEmbeddedApp: false` を指定すると、ライブラリは JWT の代わりにクッキーを探すロジックに自動で切り替わります。

```javascript
import { shopifyApp } from "@shopify/shopify-app-remix/server";
import { PrismaSessionStorage } from "@shopify/shopify-app-session-storage-prisma";
import prisma from "./db.server";

const shopify = shopifyApp({
  apiKey: process.env.SHOPIFY_API_KEY,
  apiSecretKey: process.env.SHOPIFY_API_SECRET || "",
  apiVersion: "2024-01",
  scopes: process.env.SCOPES?.split(","),
  appUrl: process.env.SHOPIFY_APP_URL || "",

  // 非埋め込みアプリの設定
  isEmbeddedApp: false,

  sessionStorage: new PrismaSessionStorage(prisma),
});

export default shopify;
export const authenticate = shopify.authenticate;
```

#### B. ログインページの実装 (`app/routes/auth.login.jsx`)

非埋め込みアプリでは、ユーザーが直接URLを叩いた際にどのストアを認証すべきか判断できないため、ストア名を入力させるフォームが必要です。

```javascript
import { useState } from "react";
import { json } from "@remix-run/node";
import { Form, useActionData } from "@remix-run/react";
import shopify from "../../shopify.server";

export const loader = async ({ request }) => {
  return await shopify.login(request);
};

export const action = async ({ request }) => {
  // ユーザーがショップドメインを入力して送信すると、OAuthフローが開始される
  return await shopify.login(request);
};

export default function Login() {
  const actionData = useActionData();
  const [shop, setShop] = useState("");

  return (
    <div style={{ padding: "40px", fontFamily: "sans-serif" }}>
      <h1>App Login</h1>
      <p>認証を開始するにはショップドメインを入力してください</p>

      <Form method="post" style={{ marginTop: "20px" }}>
        <input
          type="text"
          name="shop"
          placeholder="example.myshopify.com"
          value={shop}
          onChange={(e) => setShop(e.target.value)}
          style={{ padding: "8px", width: "300px", marginRight: "10px" }}
        />
        <button type="submit" style={{ padding: "8px 16px" }}>
          Login
        </button>

        {actionData?.errors?.shop && (
          <p style={{ color: "red", marginTop: "10px" }}>
            {actionData.errors.shop === "MISSING_SHOP"
              ? "ドメインを入力してください"
              : "無効なドメインです"}
          </p>
        )}
      </Form>
    </div>
  );
}
```

#### C. 認証リダイレクトのタイムライン

1.  **初期アクセス**: `https://app.com/` にアクセス -> `authenticate.admin` がクッキーを確認し、なければログインページへ。
2.  **OAuth開始**: ユーザーがショップドメインを入力して送信 -> `/auth` を経由して Shopify の承認画面へリダイレクト。
3.  **コールバック**: 承認後、Shopify が **`auth/callback`** （テンプレートでは `auth/$.tsx`）にリクエストを戻す。
4.  **クッキー発行**: `auth/callback` ルート内で `authenticate.admin` が呼ばれ、アクセストークンの交換・保存に成功すると、サーバーがブラウザに **セッションクッキーを `Set-Cookie`** する。
5.  **完了**: クッキーを持った状態で、アプリの初期ページ（`/`など）へ最終リダイレクトされる。

### なぜ非埋め込みでは Session Token が使えないのか？

Session Token (JWT) は、Shopify Admin の親ウィンドウと通信する `App Bridge` によって発行されます。非埋め込みアプリは Shopify の外で独立して動くため、App Bridge が親ウィンドウ（Shopify）を見つけることができず、トークンの発行・取得ができないからです。

---

## 9. プロジェクトにおける制御ファイル

- **`app/shopify.server.js`**: `shopifyApp` 関数により、APIキー、スコープ、`sessionStorage` (Prisma) が一括設定されている。
- **`prisma/schema.prisma`**: `Session` モデルがアクセストークンの保存先。
- **`.env`**: `SHOPIFY_API_SECRET` が Cookie 署名や暗号化の根源的な鍵として機能する。

---

作成日: 2026年2月23日
Antigravity 詳細レポート
