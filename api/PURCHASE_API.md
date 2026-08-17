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

複数商品をまとめて購入できる（WORK-011）。`productCodes` を配列で受け取る。

```json
{
  "productCodes": ["HANABI", "HANABI_GOOGLE_EARTH", "SUN_AND_MOON"]
}
```

後方互換として単数 `{ "productCode": "SUN_AND_MOON" }` も受理し、`["SUN_AND_MOON"]` として扱う（既存STORE UIが単品呼び出しのため）。

AUTH_USER_ID、金額、Stripe Price ID、PRODUCT_ID、商品順、購入者メールをブラウザから受け取らない/信用しない。購入には Platform 認証が必須。

### 購入者メール（WORK-011）

購入者メールは Platform 認証情報（検証済み JWT の AuthContext.email）から確定する。ブラウザ request body の email は使用しない（body に email を含めても無視される）。

Platform はメールをログイン ID として使う設計のため、AUTH_USER_ID は取得できるが email が取得できない場合は異常状態として Checkout 開始を拒否する（`AUTH_EMAIL_REQUIRED` 403）。Stripe 画面でメールを入力させて不足情報を補完する設計にしない。

確定した email は Checkout Session の `customer_email` に渡す。Stripe 公式仕様により、`customer_email` を渡すと Checkout 上で購入者メールが事前入力され編集不可（readonly）になる。これにより「別メール自由入力 → そのメールで購入」を防ぎ、Platform ログインメール = Stripe Checkout 購入者メール を保証する。

権限付与先は Stripe の email ではなく AUTH_USER_ID を正本とする（後述 metadata / Webhook 参照）。Stripe email は権限判定キーではない。

### サーバー処理

1. JWT検証
2. 認証済み email（AuthContext.email）を取得。取得できなければ `AUTH_EMAIL_REQUIRED` 403 で拒否
3. `productCodes` が配列・非空・重複なしを検証（重複は黙って除去せずエラー。body の email 等は参照しない）
4. M_USER有効確認
5. 各PRODUCT_CODEのM_PRODUCT存在確認
6. 各商品が販売対象（Stripe Price Secret設定済み）を確認
7. 既存の有効権限確認（二重購入防止）
8. 商品依存条件検証（HANABI_GOOGLE_EARTHは、HANABI保有済み または 同一注文にHANABIを含む場合のみ購入可）
9. M_PRODUCT.SORT_NO ASC, PRODUCT_ID ASC で決定的に正規化（配列順は正本にしない）
10. サーバー側設定から各Stripe Price ID取得
11. 複数line_itemsでCheckout Session作成（mode=payment、各quantity=1、customer_email=認証済みemail）
12. metadataを設定
13. Checkout URL返却

決済手段（カード / Apple Pay / Link 等）は payment_method_types をコードで固定せず、Stripe Dashboard の Payment methods 設定（Dynamic Payment Methods）を正とする。Apple Pay は Hosted Checkout の標準機能で追加コード不要。Link は Dashboard の payment method settings で無効化する（コード変更不要）。

### metadata

```text
auth_user_id
product_codes   （監査用途。カンマ連結。権限付与対象の正本にはしない）
```

権限付与対象商品の正本は、Webhookで再取得したStripe実line_itemsのPrice ID逆引きとする（metadataを正本にしない）。

購入者メールは Stripe 側（customer_email / customer）が保持するため、metadata へ email を重複保持しない。entitlement 付与先は metadata.auth_user_id（= AUTH_USER_ID）を正本とし、Stripe の email で付与先ユーザーを特定しない。

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

### 支払い試行ライフサイクル（WORK-011 / migration 0006）

Checkout は「支払い試行」（T_CHECKOUT_ATTEMPT）として追跡し、二重課金の事前防止・HTTP 再送収束・paid 未反映の救済を行う。checkout Request には安定キー `operationId`（ブラウザ生成）を含め、同一 operationId の再送は同一試行へ収束させる（`UX_ATTEMPT_OPERATION`）。

並行 Checkout 排他は `T_PRODUCT_CHECKOUT_LOCK` の `PRIMARY KEY (AUTH_USER_ID, PRODUCT_ID)` を正本とする。試行作成は attempt・item・lock を 1 回の `db.batch()` で原子的に確定する（adr/ADR-008）。lock 競合で作成が rollback した場合は `ALREADY_IN_PROGRESS`（409）を返す（新 attempt 未作成が確定）。

checkout 呼び出し時、同一商品を含む active な別試行が残っている場合の分岐:

- 対象の旧試行が Stripe 上 open（未払い・未失効）→ `CHECKOUT_RESTART_CONFIRM`（409）。ユーザーが明示同意した場合のみ旧試行を expire（Stripe Session を失効）して新試行へ置換する。無断で旧 Session を無効化しない。
- 旧試行が既に paid → 二重購入を防ぐため新規作成しない（成立済みとして扱う）。
- 旧試行の状態が不明で、かつ新 attempt が未作成と確定できる → `CHECKOUT_RESTART_PENDING`（503）。クライアントは pending を復元し、時間をおいて再試行する。
- Stripe create 実行後に結果が確定できない（`CREATE_ATTEMPTED=1` かつ `STRIPE_SESSION_ID=NULL`）→ 推測で終端化せず、`CHECKOUT_CREATE_FAILED` 等で状態確認へ誘導する（安全側を維持）。

Stripe 状態同期（settle）は、対象試行の Stripe Session を再取得して次のいずれかに分類する: open（再開可能）/ complete（支払い済み。fulfillment へ収束）/ expired（失効）/ cancelled / not_created / indeterminate（確定不能）。open を expire するのは「明示的な置換・整理」時のみで、単なる状態確認では expire しない。expired/cancelled と確定できた試行のみ終端化し lock を解放する。confirm できない（indeterminate 等）は lock を維持し、推測で終端化しない。

## 3. GET /api/purchases/active-checkout

### 目的

ログインユーザーの「現在再開可能な購入手続き」を返す（STORE 表示の正本）。別端末で開始した試行も AUTH_USER_ID 基準で発見でき、localStorage を正本にしない。

### 認証

必須（未ログインは 401）。AUTH_USER_ID はサーバーの検証済み JWT から取得し、クライアント指定を受け付けない。

### サーバー処理

1. AUTH_USER_ID の active（CREATING/OPEN）試行を全取得（operationId 不要）。
2. 各候補を settle（Stripe 状態確認、open は expire しない）で評価:
   - open → 再開可能候補。
   - complete → 既存 fulfillment（fulfillCheckoutSession, "recovery"）へ収束させ、成功時は attempt を PAID・lock 解放して非表示。not_paid / invalid / inconsistent 等の確定不能は終端化せず安全側維持（非表示）。
   - expired / not_created → settle が終端化・lock 解放済み（非表示）。
   - indeterminate → 状態不明。lock 維持・非表示。
3. 再開可能候補が複数残る場合は最新（ATTEMPT_ID 最大）を残し、それ以前は明示的に expire（open のみ・expire 成功時のみ終端化）して整理する。
4. 古い open を安全に終了できない（expire 失敗・indeterminate 等）場合は「最大 1 件」の前提を満たせないため、resumable を確定せず `ACTIVE_CHECKOUT_PENDING`（503）で状態確認中を返す（古い Stripe Session を生かしたまま UI だけ隠さない）。

### Response

```json
{
  "result": "OK",
  "data": { "resumable": { "operationId": "...", "productCodes": ["..."] } }
}
```

再開可能な試行がなければ `resumable` は `null`。クライアントは 200 かつ正常 JSON を確認できたときのみ localStorage の pending を整合させ、通信失敗・5xx・JSON 失敗時は補助情報を保持する（誤削除で再開導線を失わない）。

## 4. POST /api/purchases/recover

### 目的

購入手続きの再開（既存 operationId の再送）。対象試行を settle し、open なら同一パラメータで Stripe Session を再現して checkoutUrl を返す。

### 挙動

- 対象試行が paid → 成立済みとして扱う。
- expired → 終端化・lock 解放のうえ `CHECKOUT_EXPIRED`（409）。クライアントは pending を消し「有効期限切れで終了」を通知する。
- operationId 不一致・対象外 → `OPERATION_MISMATCH` / `OPERATION_CLOSED`（409）。
- 状態不明 → 推測で終端化せず状態確認へ誘導。

recover は client_reference_id に operationId を保持し、success 側の復元に用いる。

## 5. POST /api/purchases/cancel

### 目的

購入手続きの取り消し。対象試行を settle し、終端化できたもののみ lock を解放する。

### 挙動

- cancelled / expired / not_created → 終端化し `result` を返す。
- paid / complete → 支払い済みは cancel せず already_paid 相当（反映は success/Webhook/active-checkout の fulfillment 経路に委ねる）。
- indeterminate → SID=NULL の create 結果不明は `CANCEL_INDETERMINATE`（409）、それ以外（通信/expire 失敗）は `CANCEL_RETRY`（503）。推測で終端化しない。

## 6. POST /api/stripe/webhook

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

金額・Price照合のため、署名検証済みSessionをStripeから再取得し `line_items`（`line_items.data.price` 含む）を展開する。クライアント値・metadataは信用しない。

使用値:

- Checkout Session ID
- metadata.auth_user_id
- 再取得したline_items（各price.id / unit_amount / currency / quantity）
- amount_total
- payment_status
- currency
- created

### 商品確定（複数商品）

権限付与対象商品は、再取得したStripe実line_itemsを正本とする（metadata.product_codesは正本にしない）。

各line_itemを検証する:

- quantity === 1（厳密。1以外は不整合）
- price.id が存在
- price.currency === jpy
- price.unit_amount が数値
- price.id がサーバー既知Price（Price ID → PRODUCT_CODE 逆引きに含まれる）
- 同一PRODUCT_CODEが複数line_itemに現れない

さらに session.currency === jpy、全line_itemのunit_amount合計 === session.amount_total を照合する。

Price ID → PRODUCT_CODE の逆引きはサーバー側Secretから構築する。想定外Priceが1件でもあれば購入・権限付与しない。同一Price IDが複数商品へ割り当てられている設定重複も、安全側で設定エラーとして拒否する。

### DB更新

以下を1回のD1 batchで一括実行し、原子的に反映する（adr/ADR-008）。全件成立または全件ロールバックとする。

```text
T_ORDER INSERT               （注文ヘッダ 1行）
T_PURCHASE INSERT × 商品数     （購入明細）
T_USER_PRODUCT INSERT/UPDATE × 商品数（権限）
```

T_ORDER:

```text
PURCHASE_SOURCE = 0
EXTERNAL_ORDER_ID = Checkout Session ID
TOTAL_AMOUNT = amount_total
PAYMENT_STATUS = 1
```

T_PURCHASE（明細ごと）:

```text
PURCHASE_SOURCE = 0
EXTERNAL_PURCHASE_ID = NULL   （Session IDはT_ORDERが保持）
ORDER_ID = 作成したT_ORDERのORDER_ID
AMOUNT = 当該商品のunit_amount（購入時価格を履歴保持）
PAYMENT_STATUS = 1
```

T_USER_PRODUCT（明細ごと）:

```text
STATUS = 1
START_DATE = 決済日時
END_DATE = 9999-12-31T23:59:59+09:00
GRANT_TYPE = 0
PURCHASE_ID = 対応するT_PURCHASEのPURCHASE_ID
DEL_FLG = 0
```

ORDER_IDおよびPURCHASE_IDの参照には `last_insert_rowid()` を使わない。ORDER_IDは `(PURCHASE_SOURCE, EXTERNAL_ORDER_ID)` から、PURCHASE_IDは `(ORDER_ID, PRODUCT_ID)` から後続SQLで参照する（同一batch内で先行INSERT行を参照できる。`UX_T_PURCHASE_ORDER_PRODUCT` で1件に確定）。

### 冪等性

注文単位の冪等キーは T_ORDER の `PURCHASE_SOURCE + EXTERNAL_ORDER_ID` 一意制約とする。

同じWebhookが再送された場合、同一Session IDのT_ORDERが既に存在すれば処理済みとして200を返す。batch成功時は全件成立する設計のため、T_ORDER存在＝その注文の全DB反映済みと扱える。

### 不整合

以下の場合は権限付与せず内部エラーとして記録する。

- AUTH_USER_IDが存在しない
- PRODUCT_CODEが解決できない（Price逆引き不能・商品不在）
- metadata不足（auth_user_id）
- 金額・Price設定の不一致（想定外Price・quantity≠1・通貨不一致・合計不一致・Price設定重複）
- 支払完了条件を満たさない

## 7. GET /api/purchases/status

### 目的

Stripeから戻った購入完了画面で、Webhook反映状況を確認する。複数商品購入に対応し、対象全商品の権限反映状態を確認できる。

### Query

```text
productCodes   （カンマ区切り。例: HANABI,HANABI_GOOGLE_EARTH,SUN_AND_MOON）
productCode    （後方互換。単数。productCodesが無い場合に使用）
```

両方ある場合は productCodes を優先する。サーバー側で trim・空除外・重複を安全に一意化する（順序は最初の出現を維持）。空・64文字超は 400。

### 判定

権限反映は既存のentitlement判定（T_USER_PRODUCTのSTATUS/START_DATE/END_DATE/DEL_FLG とユーザー・商品状態）を正本とする。T_ORDER/T_PURCHASE の存在では判定しない。必ずログイン中ユーザーの権限のみを見る（URLの商品コードは「確認対象」に過ぎず、購入済み判定の正本にしない）。他ユーザーの権限は参照できない。

### Response

```json
{
  "result": "OK",
  "data": {
    "allGranted": false,
    "products": [
      { "productCode": "HANABI", "granted": true },
      { "productCode": "HANABI_GOOGLE_EARTH", "granted": true },
      { "productCode": "SUN_AND_MOON", "granted": false }
    ]
  }
}
```

`allGranted` は全商品が利用可能なときのみ true（商品が0件のときは false）。

後方互換: 単数 `productCode` で問い合わせた場合のみ、上記に加えて従来形 `productCode` / `granted` も併せて返す。

### 成功画面の動作

成功画面はStripeから戻った後、URLの productCodes（単数 productCode も受理し配列へ正規化）に含まれる全商品について本APIで反映を確認する。

- 全商品 granted → 購入完了表示。
- 一部未反映 → 一定回数polling（既存単品画面の間隔・回数を踏襲）。
- 上限到達しても未反映 → 決済失敗と断定せず、「決済は完了していますが、利用権限の反映を確認中です。時間をおいて再度ご確認ください。」等の安全な表示。

成功画面は権限付与やDB更新を行わない（表示・確認のみ）。URLの productCodes は改ざん可能なため、購入済み判定の正本にしない。権限の正本はサーバー側entitlement。

商品名は `/api/products` の name を再利用して表示してよい（PRODUCT_CODE は内部識別子）。

権限付与を完了画面のAPIから代行しない。

## 8. 二重購入

有効権限がある商品はCheckout作成前に警告し、原則購入開始を拒否する。

複数商品購入では、依存条件も購入開始前にサーバー側で検証する。HANABI_GOOGLE_EARTHは、HANABIを保有済み、または同一注文にHANABIを含む場合のみ購入できる。

Stripe側で二重決済が成立した場合:

- T_PURCHASEは購入事実として保持
- T_USER_PRODUCTは1件のまま
- 管理者が返金判断する

## 9. 返金

初期はStripe管理画面で返金し、対応Webhookまたは管理者操作で以下を更新する。

```text
T_PURCHASE.PAYMENT_STATUS = 2
T_PURCHASE.REFUND_DATE = 返金日時
```

商品権限停止は返金処理と分離する。

自動停止の採否は実装時に明示する。初期推奨は管理者確認後の手動停止。

## 10. 秘密情報

Cloudflare Secrets:

```text
STRIPE_SECRET_KEY
STRIPE_WEBHOOK_SECRET
STRIPE_PRICE_SUN_AND_MOON
STRIPE_PRICE_HANABI
STRIPE_PRICE_HANABI_GOOGLE_EARTH
```

公開可能なPublishable Key以外はフロントへ渡さない。

## 10. 環境別 Checkout return URL（2026-08-18追補）

Checkout Sessionの`success_url` / `cancel_url`は `env.APP_BASE_URL` を唯一の正とする。URLを正規化する際に `.origin` のみを返してpathnameを捨ててはならない。

- Production `APP_BASE_URL=https://shingo-camera.com` → Production `/purchase/...`
- DEV `APP_BASE_URL=https://shingo-camera.com/dev` → DEV `/dev/purchase/...`

`request.url.origin`は同一operationのStripe createパラメータを揺らす可能性があるため正本にしない。2026-08-18、DEV Stripe Test Checkoutのreturnが`/dev/purchase/success/`へ戻ることを実機確認済み。
