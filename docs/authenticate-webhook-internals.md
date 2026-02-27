# `authenticate.webhook()` 内部実装ガイド

このドキュメントは `@shopify/shopify-app-react-router` の `authenticate.webhook()` がどのように動作するかを、ソースコード (`opensrc/`) を参照しながら解説したものです。

- 参照ソース: `opensrc/repos/github.com/Shopify/shopify-app-js/`
- 公式ドキュメント: https://shopify.dev/docs/apps/build/webhooks

---

## 概要

```js
// app/routes/api.webhooks.jsx
const { shop, session, topic } = await authenticate.webhook(request);
```

この1行の内部では、以下の3つの処理が順に実行されます。

1. **HTTPバリデーション** — POSTメソッドの確認
2. **HMAC署名検証** — リクエストがShopifyから送られたものか確認
3. **セッション取得** — DBからshopのオフラインセッションを取得

---

## 実行フロー

```
POST /api/webhooks
    │
    ▼
authenticate.webhook(request)
    │
    ├─ [1] method !== POST?  →  throw Response(405)
    ├─ rawBody = await request.text()   ← JSON.parse せず生文字列で取得
    │
    ├─ [2] api.webhooks.validate(rawBody, request)
    │       │
    │       ├─ detectWebhookType(headers)
    │       │     shopify-hmac-sha256 あり → WebhookType.Events
    │       │     X-Shopify-Hmac-Sha256 あり → WebhookType.Webhooks
    │       │
    │       ├─ validateHmacFromRequest()
    │       │     rawBody 空? → fail(MissingBody)
    │       │     HMACヘッダーなし? → fail(MissingHmac)
    │       │     SHA256-HMAC(ClientSecret, rawBody) 計算
    │       │     safeCompare(受信HMAC, 計算HMAC)
    │       │       → 不一致: { valid: false, reason: InvalidHmac } → throw 401
    │       │
    │       └─ checkWebhookHeaders()
    │             必須ヘッダー不足 → throw 400
    │             全OK → { valid: true, domain, topic, webhookId, ... }
    │
    ├─ [3] ensureValidOfflineSession(params, check.domain)
    │       DBから offlineSessionId = "{shop}_offline" でセッション取得
    │       なければ undefined（アンインストール済みショップ等）
    │
    └─ return WebhookContext
```

---

## [1] HTTPバリデーション

```ts
// packages/apps/shopify-app-react-router/src/server/authenticate/webhooks/authenticate.ts

if (request.method !== 'POST') {
  throw new Response(undefined, { status: 405, statusText: 'Method not allowed' });
}

// request.json() ではなく request.text() を使う理由:
// JSON.parse → JSON.stringify すると空白・改行が変化し
// Shopify側が計算したHMACと一致しなくなるため
const rawBody = await request.text();
```

---

## [2] HMAC署名検証

### Webhookの種類の判定

```ts
// packages/apps/shopify-api/lib/webhooks/validate.ts

function detectWebhookType(headers: Headers): WebhookTypeValue {
  const eventsHmac = getHeader(headers, 'shopify-hmac-sha256');
  if (eventsHmac) return WebhookType.Events;     // 新型 Events Webhook

  const webhooksHmac = getHeader(headers, 'X-Shopify-Hmac-Sha256');
  if (webhooksHmac) return WebhookType.Webhooks; // 従来型 Webhook

  return WebhookType.Webhooks; // デフォルト
}
```

### HMACヘッダーの違い（2種類）

| | 従来型 Webhook | Events Webhook |
|---|---|---|
| HMACヘッダー | `X-Shopify-Hmac-Sha256` | `shopify-hmac-sha256` |
| Topicヘッダー | `X-Shopify-Topic` | `shopify-topic` |
| Domainヘッダー | `X-Shopify-Shop-Domain` | `shopify-shop-domain` |
| 固有フィールド | `webhookId`, `subTopic`, `name` | `handle`, `action`, `resourceId` |
| HMAC計算 | SHA256-HMAC(Secret, rawBody) | **同一** |

> **重要:** HMACの計算アルゴリズムは両者で全く同じ。異なるのはヘッダー名のみ。

### 実際のHTTPヘッダー

```
# 従来型 Webhook
X-Shopify-Hmac-Sha256: XWmrwMey6OsLMeiZKwP4FppHH3cmAiiJJAweH5Jo4bM=
X-Shopify-Topic: orders/create
X-Shopify-Shop-Domain: example.myshopify.com
X-Shopify-API-Version: 2026-01
X-Shopify-Webhook-Id: b54557e4-bdd9-4b37-8a5f-bf7d70bcd043

# Events Webhook
shopify-hmac-sha256: (base64 encoded HMAC)
shopify-topic: orders/create
shopify-shop-domain: example.myshopify.com
shopify-event-id: 98880550-7158-44d4-b7cd-2c97c8a091b5
shopify-handle: my-subscription-handle
shopify-action: create
shopify-resource-id: gid://shopify/Order/12345
```

### HMAC計算ロジック

```ts
// packages/apps/shopify-api/lib/utils/hmac-validator.ts

// HMACヘッダーをwebhookTypeに応じて動的に選択
const hmacHeaderName = WEBHOOK_HEADER_NAMES[webhookType].hmac;
const hmac = getHeader(request.headers, hmacHeaderName);

// アプリのClientSecretで rawBody を SHA256-HMAC計算
const localHmac = await createSHA256HMAC(config.apiSecretKey, rawBody, HashFormat.Base64);

// タイミング攻撃耐性のある比較（=== は使わない）
return safeCompare(hmac, localHmac);
```

> `safeCompare()` は `===` と異なり、文字列の長さに関わらず常に一定時間で比較する。
> 通常比較は最初の不一致で即リターンするため、差分を測定することで秘密鍵を推測する
> タイミング攻撃に対して脆弱になる。

---

## [3] セッション取得

```ts
// packages/apps/shopify-app-react-router/src/server/helpers/create-or-load-offline-session.ts

async function createOrLoadOfflineSession({ api, config }, shop) {
  // 公開アプリ: DBからオフラインセッションをロード
  const offlineSessionId = api.session.getOfflineId(shop); // "offline_xxx.myshopify.com"
  const session = await config.sessionStorage.loadSession(offlineSessionId);
  return session; // アンインストール済みなら undefined
}
```

---

## 返り値の型

```ts
// packages/apps/shopify-app-react-router/src/server/authenticate/webhooks/types.ts

// Union型で session の存在チェックをコンパイラが強制
type WebhookContext<Topics> =
  | WebhookContextWithoutSession<Topics>  // { session: undefined, admin: undefined }
  | WebhookContextWithSession<Topics>;    // { session: Session,   admin: AdminApiContext }
```

### 従来型 Webhook の返り値

```ts
{
  webhookType: 'webhooks',
  shop: 'example.myshopify.com',  // X-Shopify-Shop-Domain
  topic: 'APP_UNINSTALLED',       // 正規化済み (orders/create → ORDERS_CREATE)
  webhookId: 'abc-123',           // X-Shopify-Webhook-Id
  apiVersion: '2026-01',
  payload: { /* JSON.parse済み */ },
  subTopic?: string,
  name?: string,
  session: Session | undefined,
  admin: AdminApiContext | undefined,
}
```

### Events Webhook の返り値

```ts
{
  webhookType: 'events',
  shop: 'example.myshopify.com',
  topic: 'ORDERS_CREATE',
  webhookId: 'event-id-xxx',      // eventId のエイリアス（将来削除予定）
  eventId: 'event-id-xxx',
  handle: 'my-subscription',
  action: 'create',               // 'create' | 'update' | 'delete'
  resourceId: 'gid://shopify/Order/12345',
  payload: { /* JSON.parse済み */ },
  session: Session | undefined,
  admin: AdminApiContext | undefined,
}
```

---

## 実装パターン

### 基本パターン（このプロジェクト）

```js
// app/routes/api.webhooks.jsx
export const action = async ({ request }) => {
  const { shop, session, topic } = await authenticate.webhook(request);

  switch (topic) {
    case 'APP_UNINSTALLED':
      // session は undefined になりうる（アンインストール後の遅延配信）
      if (session) {
        await db.session.deleteMany({ where: { shop } });
      }
      break;
    default:
      throw new Response('Unhandled webhook topic', { status: 404 });
  }

  return new Response();
};
```

### webhookType を使うパターン

```js
export const action = async ({ request }) => {
  const context = await authenticate.webhook(request);

  if (context.webhookType === 'events') {
    // Events Webhook固有のフィールドが使える
    const { action, resourceId, handle } = context;
    console.log(`${action} on ${resourceId} via ${handle}`);
  } else {
    // 従来型
    const { webhookId, subTopic } = context;
  }

  return new Response();
};
```

---

## よくある落とし穴

### 1. `session` を確認せずに使う

```js
// ❌ NG: アプリアンインストール後の遅延配信でクラッシュ
const { session } = await authenticate.webhook(request);
await db.session.deleteMany({ where: { shop: session.shop } }); // session が undefined!

// ✅ OK
if (session) {
  await db.session.deleteMany({ where: { shop: session.shop } });
}
```

### 2. `request.text()` の二重読み取り

```js
// ❌ NG: authenticate.webhook() が内部で request.text() を呼んでいるため
//       その後で再度読もうとしても空になる
const { payload } = await authenticate.webhook(request);
const body = await request.text(); // "" (空)

// ✅ OK: payload を使う（authenticate.webhook が JSON.parse 済み）
const { payload } = await authenticate.webhook(request);
console.log(payload.id);
```

### 3. ClientSecret ローテーション後のHMAC不一致

> 公式ドキュメントより:
> ClientSecret をローテーションした場合、新しい Secret でのHMAC生成が反映されるまで**最大1時間**かかる。
> この間、HMACバリデーションが失敗する可能性がある。

---

## 関連ファイル

| ファイル | 役割 |
|---|---|
| `app/routes/api.webhooks.jsx` | Webhookエンドポイント |
| `opensrc/.../webhooks/authenticate.ts` | `authenticate.webhook()` 本体 |
| `opensrc/.../webhooks/validate.ts` | HMAC検証 + ヘッダーチェック |
| `opensrc/.../webhooks/types.ts` | `WEBHOOK_HEADER_NAMES`, 型定義 |
| `opensrc/.../utils/hmac-validator.ts` | SHA256-HMAC計算, `safeCompare` |
| `opensrc/.../lib/types.ts` | `ShopifyHeader`, `ShopifyEventsHeader` 定数 |
| `opensrc/.../helpers/create-or-load-offline-session.ts` | DBセッション取得 |
