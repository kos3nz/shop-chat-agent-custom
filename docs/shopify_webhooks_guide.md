# Shopify Webhook 実装ガイド

このドキュメントは、Shopify アプリにおける Webhook の仕組み、実装方法、およびベストプラクティスについてまとめたものです。

---

## 1. Webhook の基本概念

Webhook は、Shopify ストアで特定のイベント（商品作成、注文更新、アプリのアンインストールなど）が発生した際に、Shopify からアプリのサーバーへ HTTP POST リクエスト（通知）を送る仕組みです。これにより、アプリは定期的に API を叩いて状態を確認（ポーリング）する必要がなくなり、効率的な運用が可能になります。

| 方法                    | 設定場所            | 特徴                                                                                   |
| :---------------------- | :------------------ | :------------------------------------------------------------------------------------- |
| **App-specific (推奨)** | `shopify.app.toml`  | **静的な一律登録**。デプロイ時に CLI が同期。全ショップ共通の設定に最適。              |
| **Shop-specific**       | `shopify.server.js` | **動的な個別登録**。コード内で `registerWebhooks` を実行。ショップ毎の条件分岐が可能。 |

### なぜ `app.toml` では「プラン別の出し分け」ができないのか？

`shopify.app.toml` はアプリ全体の構成を決定する **静的なファイル** です。アプリのインストール時に Shopify 側がこのファイルを読み取って一律に Webhook を設定するため、「このショップはプレミアムプランだから登録する」といった **実行時のロジック（プログラム）を挟む余地がありません。**

### 公式（shopify.dev）の見解

- **原則**: 「App-specific（`toml`）」が推奨されます。管理が簡単で、再インストール時の同期漏れといったエラーを防げるためです。
- **例外**: 公式ドキュメントでは、**「Webhook サブスクリプションの設定が、アプリがインストールされているショップに依存する場合」** に限り、Shop-specific（コードによる登録）を使用すべきであると明記されています。

つまり、**「普通の Webhook は `toml` で、ショップごとに変えたい特殊なものだけ `shopifyApp` のコードで」** というハイブリッドな使い分けが最も理想的な設計です。

---

## 2. `registerWebhooks` 関数とプログラム登録

`registerWebhooks` は、**Shop-specific** な Webhook をプログラムから明示的に登録するための関数です。

### なぜ `Record<string, ...>` なのか？

`shopifyApp()` の引数オブジェクトで Webhook のキーが任意の文字列（`string`）を許容している理由は、**拡張性と柔軟性**にあります。

- **トピックの膨大さ**: Shopify には 100 以上のトピックがあり、頻繁に追加・更新されます。
- **ライブラリの非依存**: ライブラリ側で型を固定してしまうと、Shopify の新機能リリースに型定義が追いつかなくなるため、あえて緩やかな定義になっています。

---

## 3. どちらの方法を選ぶべきか？

### A. `shopify.app.toml` (App-specific)

現在の主流であり、Shopify が強く推奨している方法です。

- **メリット**: 実装が非常にシンプル。Shopify 側で購読状態を管理するため、再インストール時の同期漏れなどが起きにくい。
- **設定例**:
  ```toml
  [webhooks]
  api_version = "2025-04"
  [[webhooks.subscriptions]]
  topics = [ "products/create" ]
  uri = "/api/webhooks"
  ```

### B. `shopifyApp` + `registerWebhooks` (Shop-specific)

プログラムで柔軟に制御したい場合に使用します。

- **メリット**: ショップのメタデータや料金プランに応じて、購読するトピックを条件分岐させることができる。
- **コード例**:

  ```javascript
  // shopify.server.js
  const shopify = shopifyApp({
    webhooks: {
      INVENTORY_LEVELS_UPDATE: {
        deliveryMethod: DeliveryMethod.Http,
        callbackUrl: "/api/webhooks/inventory-levels-update",
      },
    },
    hooks: {
      afterAuth: async ({ session, admin }) => {
        // 条件に応じて登録を制御するロジック
        const shopPlan = await getShopPlan(session.shop);

        if (shopPlan === "pro") {
          // プロプランのストアのみ在庫更新通知を購読する
          await shopify.registerWebhooks({ session });
        }
      },
    },
  });
  ```

#### 動的登録が必要になる主なケース

1.  **料金プランによる制限（機能の出し分け）**: 全ショップ一律ではなく、高機能版（プロプラン）を契約したストアのみに特定の通知を配信したい場合。
2.  **ユーザーによる機能のON/OFF**: アプリの設定画面で、ユーザーが「通知を受け取る」のチェックを入れた時のみ、動的に購読を開始（または解除）したい場合。
3.  **特定のリソースを持つストアのみ**: 特定のメタオブジェクトやメタフィールドを利用している高度な構成のストアのみ、その更新を監視したい場合。

### A と B の使い分けと「併用」について

**結論として、どちらか一方の方法で登録すれば十分です。**

- `shopifyApp`（プログラム側）で Webhook を定義・登録している場合、`shopify.app.toml` の `[webhooks]` セクションに同じトピックを書く必要はありません。
- プログラム側での登録 (`registerWebhooks`) が完了すれば、Shopify 側のデータベースに購読情報が保存され、Webhook は有効になります。

### 🚨 最も重要な注意点：API スコープ

どちらの方法を採用する場合でも、そのトピックを受信するのに必要な **API スコープ（`read_products` など）は必ず `shopify.app.toml` の `access_scopes` に記載されている必要があります。**
スコープが不足していると、登録自体は成功しても、実際のイベント発生時にデータが送信されません。

---

## 4. `shopifyApp` の呼び出しと実行タイミング

アプリがいつ Webhook を意識し、いつ登録されるのかのライフサイクルを理解することが重要です。

### 1. `shopifyApp` 自体の初期化

- **タイミング**: **サーバー（アプリプロセス）の起動時**。
- **内容**: 設定値のバリデーションや、モジュールのエクスポート準備が行われます。リクエストのたびに `shopifyApp` が一から作り直されるわけではありません。

### 2. 登録 (Registration)

- **`toml` を使う場合**: **`shopify app deploy` 実行時**。この時、Shopify のプラットフォーム側に設定が保存されます。各ショップがアプリをインストールした際、Shopify は自動でその設定を適用します。
- **プログラム (Shop-specific) を使う場合**: **`afterAuth` フックの実行時**。OAuth 認証が完了し、アプリが対象ショップへのアクセス権（トークン）を得た直後に、アプリから Shopify へ登録リクエストが送られます。

---

## 5. Mandatory Webhooks (必須ウェブフック / コンプライアンス)

Shopify App Store でアプリを公開するためには、**「Mandatory（必須）ウェブフック」**への対応が義務付けられています。これらは個人情報保護（GDPR等）の観点で非常に重要です。

### 必須トピック一覧

1.  **`customers/data_request`**: 顧客が自分のデータの開示を要求した時。
2.  **`customers/redact`**: 顧客が自分のデータの削除を要求した時。
3.  **`shop/redact`**: ショップがアプリを削除し、一定期間（48時間）経過した後にショップデータを削除する要求。

**注意:** これらの必須ウェブフックは、コード内での登録ではなく、**パートナーダッシュボード**からエンドポイント URL を設定する必要があります。

---

## 6. `APP_UNINSTALLED` と `shop/redact` の違い

どちらもアプリの削除に関連しますが、目的とタイミングが異なります。

| 特徴                     | `APP_UNINSTALLED`                                | `shop/redact`                                                |
| :----------------------- | :----------------------------------------------- | :----------------------------------------------------------- |
| **タイミング**           | アプリ削除の**直後**                             | アプリ削除の**48時間後**                                     |
| **主な目的**             | アプリ側のクリーンアップ（セッションの削除など） | 法的なデータ完全消去（GDPR対応）                             |
| **再インストールの考慮** | 即座にトークンが無効化されるため、対応必須。     | 48時間の猶予があるため、誤削除後の再インストールに対応可能。 |

---

## 7. 実装のベストプラクティス

### 認証（HMAC 検証）の実装

Shopify からの Webhook は常に署名（HMAC）が含まれています。偽装リクエストを防ぐため、必ず `authenticate.webhook(request)` を使用して検証を行ってください。検証に失敗した場合は `401 Unauthorized` を返す必要があります。

### 冪等性（Idempotency）の確保

Webhook は稀に同じ内容が複数回送られてくることがあります。同じリクエストを 2 回処理しても問題が起きないよう（冪等性）、リクエスト ID やタイムスタンプを確認するロジックを検討してください。

### 迅速なレスポンス

Shopify は Webhook 送信後、迅速な HTTP 200 OK の返却を期待しています。重い処理（外部 API 連携や大量の計算）が必要な場合は、レスポンスを返した後にバックグラウンドジョブ（Queue 等）で実行するのが定石です。

#### 実装イメージ（バックグラウンドジョブの利用）

```javascript
export const action = async ({ request }) => {
  const { topic, payload } = await authenticate.webhook(request);

  // 重い処理が確実に完了するように外部のキュー（RedisのBullMQ, SQS, Cloudflare Queues など）に登録。この呼び出し自体は即座に終わる。
  await webhookQueue.add({ topic, payload });

  // Shopifyにはすぐに 200 を返す
  return new Response(null, { status: 200 });
};
```

**⚠️ 注意: サーバーレス環境での「待たない」実行**
`await` を付けずに非同期関数を呼び出す（fire-and-forget）手法は、Cloudflare Workers や Lambda 等では**推奨されません**。レスポンスが返った瞬間に実行環境がシャットダウンされ、バックグラウンドで動いていたはずの処理が強制終了される可能性があるためです。確実な処理には必ず永続的なキューシステム（Cloudflare Queues, SQS, Redis等）を介してください。

**デメリット：HTTP で非同期にする場合の悩み**

- **キューの管理**: Redis などの外部インフラを別途用意・管理する必要が出てきます。
- **リトライの複雑さ**: Shopify はあなたのサーバーから 200 が返ってきた時点で「配信成功」とみなします。もしその後、あなたのバックグラウンドジョブ側でエラーが起きても、Shopify は再送してくれません（自前でリトライロジックを組む必要があります）。

### データ不一致の補完（Reconciliation）

Webhook の配送は 100% 保証されているわけではありません（ネットワーク障害等）。重要なデータについては、定期的なポーリングや同期ジョブを別途実装し、Webhook による更新を補完することが推奨されます。

---

## 8. 配信方法 (DeliveryMethod) の比較と設定

Shopify は Webhook の配信先として 3 つの方式をサポートしています。

| メソッド          | 配信先          | 特徴                                         | 適したケース         |
| :---------------- | :-------------- | :------------------------------------------- | :------------------- |
| **`Http`**        | HTTPS URL       | シンプルで最も一般的。                       | 低〜中頻度の通知     |
| **`EventBridge`** | AWS EventBridge | AWS インフラへ直接送信。サーバー負荷が低い。 | AWS利用/大規模アプリ |
| **`PubSub`**      | Google Pub/Sub  | GCP インフラへ直接送信。スケールに強い。     | GCP利用/大規模アプリ |

### `shopify.app.toml` での設定形式

設定ファイルでは、`uri` プロパティの形式を変えることで配信方法を指定します。

- **HTTP**: `uri = "/webhooks/path"` (通常のパス)
- **EventBridge**: `uri = "arn:aws:events:..."` (AWSのARN)
- **PubSub**: `uri = "pubsub://project-id:topic-name"` (GCPプロジェクトとトピック名)

### サーバーレス環境（Cloudflare Workers等）での注意点

Cloudflare Workers はリクエストの集中には非常に強いですが、後続のデータベース接続数などを圧迫する可能性があります。大量の通知が予想される場合は、直接 DB を操作せず、**Queue（Cloudflare Queues 等）** を介した非同期処理を検討してください。

### EventBridge を利用した AWS Lambda での処理例

EventBridge を使用すると、Shopify 側の「5秒以内のレスポンス」という制約から解放され、AWS 側で確実にリトライやキューイングを行うことができます。

```javascript
// AWS Lambda での受信例
export const handler = async (event) => {
  const topic = event["detail-type"]; // 例: "orders/create"
  const shop = event.detail.metadata.shop_domain;
  const payload = event.detail.payload; // Webhookの本体データ

  console.log(`EventBridge Webhook: ${topic} for ${shop}`);

  // 重い処理やDB操作を安全に実行可能
  await processData(payload);

  return { statusCode: 200 };
};
```

### なぜクラウドバスでは「5秒制限」がなくなるのか？

HTTP 配信の場合、Shopify は直接アプリサーバーにリクエストを送り、そのレスポンスを 5 秒間待ちます。一方で、EventBridge や Pub/Sub の場合、Shopify は **AWS や GCP のインフラ（イベントバス/トピック）** に対してリクエストを投げます。クラウドインフラは即座に受理応答（Ack）を返すため、Shopify にとっての配送は一瞬で完了します。その後の Lambda 等での処理時間は、Shopify 側のタイムアウト制限に一切影響しません。

### セキュリティ検証（HMAC）について

- **HTTP**: 誰でもエンドポイントを叩けるため、`X-Shopify-Hmac-SHA256` による検証が**必須**です。
- **EventBridge / PubSub**: Shopify と各クラウドプロバイダーの間でパートナー連携の設定（IAM等）が済んでいるため、通信経路自体が保護されています。そのため、**受け手側での HMAC 検証は原則不要**です（公式ドキュメントでも不要と明記されています）。

---

## 付録：最新の動向 (2025年時点)

- **GraphQL Admin API への完全移行**: 2024年10月1日より REST Admin API は「レガシー」となり、2025年4月1日以降の新規公開アプリは GraphQL Admin API の利用が**必須**となります。
  - **shopifyApp の内部挙動**: `shopify.registerWebhooks` や `shopify.app.toml` による登録は、内部的に GraphQL の `webhookSubscriptionCreate` ミューテーションを実行しています。ライブラリを使用している限り、開発者は意識せずとも最新の推奨事項に従っていることになります。

  - **GraphQL 移行のメリット**: REST 版では不可能だった「メタオブジェクトのサブトピック指定」や「配信フィールドのフィルタリング」など、より高度で効率的な Webhook 運用が可能になります。
    1. **`include_fields` (ペイロードの軽量化)**: 受信データから必要なフィールドのみを抽出して受信可能。

    ```toml
    [[webhooks.subscriptions]]
    topics = ["products/update"]
    uri = "/api/webhooks/products-update"
    # 必要なフィールドだけを配列で指定
    include_fields = ["id", "updated_at", "variants.price", "variants.id"]
    ```

    2. **`filter` (イベントの絞り込み)**: Shopify 側で条件判定を行い、合致する場合のみ Webhook を送信させる。

    ```toml
    [[webhooks.subscriptions]]
    topics = ["products/update"]
    uri = "/api/webhooks/music-products"
    # ステータスが active かつ、商品タイプが Music か Movies のものだけ受け取る
    filter = "status:active AND (product_type:Music OR product_type:Movies)"
    ```

    3. **`metafieldNamespaces` (メタフィールドの同梱)**: 通常は含まれないメタフィールドを Webhook データに含めることが可能。

    ```javascript
    const shopify = shopifyApp({
      webhooks: {
        PRODUCTS_UPDATE: {
          deliveryMethod: DeliveryMethod.Http,
          callbackUrl: "/api/webhooks/products",
          // 指定した名前空間のメタフィールドを Webhook データに含める
          metafieldNamespaces: ["my_app_custom_data"],
        },
      },
    });
    ```

    4. **`filter` (Metaobject のタイプ指定)**: 以前は `sub_topic` と呼ばれていましたが、2024-07 以降は `filter` フィールドに統合されました。メタオブジェクトの購読には必須の設定です。[required-filters-for-metaobjects-webhooks](https://shopify.dev/docs/apps/build/webhooks/customize/filters#required-filters-for-metaobjects-webhooks)

    ```toml
    [[webhooks.subscriptions]]
    topics = ["metaobjects/update"]
    uri = "/api/webhooks/metaobjects"
    # type: {定義ハンドル} の形式で指定
    filter = "type:product_review"
    ```

- **配信方法の多様化**: HTTP だけではなく、**Amazon EventBridge** や **Google Cloud Pub/Sub** への直接配信もサポートされており、大規模なアプリではこれらを利用することでサーバー負荷を軽減できます。

---

---

## 9. アプリ外部サービスへの Webhook 送信

Shopify の Webhook 通知先はアプリサーバーに限らず、Google Apps Script (GAS) などの外部サービスを直接指定することも可能です。

### 登録方法の比較

| 方法                       | 難易度   | コード            | 適したケース                  |
| :------------------------- | :------- | :---------------- | :---------------------------- |
| **Shopify Flow**           | 最も簡単 | 不要              | 試作・Shopify Plus プランのみ |
| **`shopify.app.toml`**     | 簡単     | 最小限            | カスタムアプリ + 本番運用     |
| **GraphQL Admin API 直接** | 中程度   | 不要（curl のみ） | 一時的・テスト用              |

### 方法 1: Shopify Flow（コード不要）

```
orders/create イベント → Flow: "Send HTTP Request" → GAS WebApp URL
```

- 管理画面の操作だけで完結

### 方法 2: `shopify.app.toml` に外部 URL を直接指定

`uri` にアプリサーバー以外のURLを指定できます。

```toml
[[webhooks.subscriptions]]
topics = ["orders/create"]
uri = "https://script.google.com/macros/s/xxx/exec"
```

- `shopify app deploy` で登録完了
- GAS 側は `doPost(e)` を実装して Web アプリとして公開するだけ
- Shopify の HMAC 署名検証を GAS 側で実装することを推奨

### 方法 3: GraphQL Admin API で直接登録（一度だけ）

Admin API アクセストークンがあれば、アプリコードなしで登録できます。

```bash
curl -X POST https://your-store.myshopify.com/admin/api/2025-01/graphql.json \
  -H "X-Shopify-Access-Token: <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation { webhookSubscriptionCreate(topic: ORDERS_CREATE, webhookSubscription: { callbackUrl: \"https://script.google.com/macros/s/xxx/exec\", format: JSON }) { webhookSubscription { id } userErrors { message } } }"
  }'
```

- 登録は**一度だけ**で OK。Shopify のDBに保存され、以降はイベント発生ごとに自動 POST される
- 削除は `webhookSubscriptionDelete(id: "gid://shopify/WebhookSubscription/xxx")` で可能

---

## 10. App-scoped と Shop-scoped の管理上の違い

### `webhookSubscriptions` クエリが返すもの

公式ドキュメントに明記されています：

> **Note: Returns only shop-scoped subscriptions, not app-scoped subscriptions configured in TOML files.**
>
> 参照: [WebhookSubscription - GraphQL Admin](https://shopify.dev/docs/api/admin-graphql/latest/objects/WebhookSubscription)

```graphql
# このクエリは toml で登録した webhook を返さない
{
  webhookSubscriptions(first: 10) {
    edges {
      node {
        id
        topic
        endpoint {
          __typename
          ... on WebhookHttpEndpoint {
            callbackUrl
          }
        }
      }
    }
  }
}
```

### スコープの概念

|                        | **App-scoped**（toml 管理）              | **Shop-scoped**（API 管理）   |
| :--------------------- | :--------------------------------------- | :---------------------------- |
| 登録方法               | `shopify.app.toml` + deploy              | GraphQL Admin API             |
| 適用範囲               | アプリがインストールされた**全ショップ** | **特定のショップ**のみ        |
| Subscription ID        | なし（"config-managed"と表示）           | あり                          |
| `webhookSubscriptions` | 返ってこない                             | 返ってくる                    |
| 確認方法               | app dashboard > Versions > Configuration | `webhookSubscriptions` クエリ |

「Shop-scoped」という命名は「どのショップに適用されるか」というスコープの話であり、アクセストークン（アプリ）の識別とは別の概念です。API 経由で登録した webhook は、そのアクセストークンが発行されたショップ×アプリに紐付きます。

### toml で登録した Webhook の確認方法

GraphQL API では確認できないため、以下の方法を使います：

1. **`shopify.app.toml` ファイル自体**（ソースオブトゥルース）
2. **Partner Dashboard** → Apps → [アプリ] → Configuration → Webhooks
3. **App Dashboard** → Versions → Configuration → Subscriptions

---

_参照ドキュメント:_

- [Shopify Dev: Webhooks overview](https://shopify.dev/docs/apps/build/webhooks)
- [Shopify Dev: Privacy law compliance](https://shopify.dev/docs/apps/build/compliance/privacy-law-compliance)
- [Shopify Dev: About managing webhook subscriptions](https://shopify.dev/docs/apps/build/webhooks/subscribe)
- [Shopify Dev: Subscribe using GraphQL Admin API](https://shopify.dev/docs/apps/build/webhooks/subscribe/subscribe-using-api)
