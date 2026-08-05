# Purchase and Stripe API

Status: Approved

## 1. 方針

Stripe Checkoutを使用する。

Stripeは決済を担当し、商品権限はD1のT_USER_PRODUCTで管理する。

決済完了画面への遷移だけで権限を付与しない。

権限付与は署名検証済みWebhookを正本とする。

## 2. POST /api/purchases/checkout

### 目的

ログイン中ユーザーの商品購入用Checkout Sessionを作成する。

### 認証

必須。

### Request

```json
{
  "productCode": "SUN_AND_MOON"
}
```

AUTH_USER_ID、金額、Stripe Price IDをブラウザから受け取らない。

### サーバー処理

1. JWT検証
2. M_USER有効確認
3. PRODUCT_CODE確認
4. 既存の有効権限確認
5. サーバー側設定からStripe Price ID取得
6. Checkout Session作成
7. metadataを設定
8. Checkout URL返却

### metadata

```text
auth_user_id
product_code
```

必要に応じて内部追跡用IDを追加できる。

個人情報や秘密情報はmetadataへ入れない。

### Response

```json
{
  "result": "OK",
  "data": {
    "checkoutUrl": "https://checkout.stripe.com/..."
  }
}
```

## 3. POST /api/stripe/webhook

### 認証

利用者JWTは不要。

Stripe-Signatureによる署名検証を必須とする。

### 署名検証

- 未加工のrequest bodyを使用
- Stripe-Signatureヘッダーを使用
- Webhook SecretはCloudflare Secretに保存
- 検証失敗時は400
- JSON解析より前に署名検証する構成を優先

### 初期対象イベント

```text
checkout.session.completed
checkout.session.async_payment_succeeded
checkout.session.async_payment_failed
checkout.session.expired
```

即時決済のみを許可する場合でも、将来の決済方法追加で破綻しないようイベントを分離する。

### checkout.session.completed

支払状態を確認し、権限付与可能な完了であることを確認する。

使用値:

- Checkout Session ID
- metadata.auth_user_id
- metadata.product_code
- amount_total
- payment_status
- created
- payment_intent

### DB更新

以下をD1 batchで一括実行する。

```text
T_PURCHASE INSERT
T_USER_PRODUCT INSERTまたはUPDATE
```

T_PURCHASE:

```text
PURCHASE_SOURCE = 0
EXTERNAL_PURCHASE_ID = Checkout Session ID
PAYMENT_STATUS = 1
```

T_USER_PRODUCT:

```text
STATUS = 1
START_DATE = 決済日時
END_DATE = 9999-12-31T23:59:59+09:00
GRANT_TYPE = 0
PURCHASE_ID = 作成したPURCHASE_ID
DEL_FLG = 0
```

### 冪等性

`PURCHASE_SOURCE + EXTERNAL_PURCHASE_ID`の一意制約で二重登録を防ぐ。

同じWebhookが再送された場合は、処理済みとして200を返す。

### 不整合

以下の場合は権限付与せず内部エラーとして記録する。

- AUTH_USER_IDが存在しない
- PRODUCT_CODEが存在しない
- metadata不足
- 金額・Price設定の不一致
- 支払完了条件を満たさない

## 4. GET /api/purchases/status

### 目的

Stripeから戻った購入完了画面で、Webhook反映状況を確認する。

### Query

```text
productCode
```

### Response

```json
{
  "result": "OK",
  "data": {
    "productCode": "SUN_AND_MOON",
    "granted": true
  }
}
```

画面側は数回だけ再確認し、反映待ちなら以下を表示する。

```text
購入情報を確認しています。しばらくしてから再読み込みしてください。
```

権限付与を完了画面のAPIから代行しない。

## 5. 二重購入

有効権限がある商品はCheckout作成前に警告し、原則購入開始を拒否する。

Stripe側で二重決済が成立した場合:

- T_PURCHASEは購入事実として保持
- T_USER_PRODUCTは1件のまま
- 管理者が返金判断する

## 6. 返金

初期はStripe管理画面で返金し、対応Webhookまたは管理者操作で以下を更新する。

```text
T_PURCHASE.PAYMENT_STATUS = 2
T_PURCHASE.REFUND_DATE = 返金日時
```

商品権限停止は返金処理と分離する。

自動停止の採否は実装時に明示する。初期推奨は管理者確認後の手動停止。

## 7. 秘密情報

Cloudflare Secrets:

```text
STRIPE_SECRET_KEY
STRIPE_WEBHOOK_SECRET
STRIPE_PRICE_SUN_AND_MOON
```

公開可能なPublishable Key以外はフロントへ渡さない。
