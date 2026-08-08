# WORK-007 Stripe Purchase

Status: Completed

Local E2E・Production E2E ともに完了（Stripe は Sandbox/Test Mode）。

## 目的
SUN AND MOON PLANNER の買い切り販売を実装する。

## 実装
- Checkout Session 作成
- metadata 設定
- Webhook 署名検証
- raw body 保持
- 金額・Price 照合
- D1 batch
- 冪等性
- 購入反映待ち画面

## 決済フロー（確定）
- `mode=payment`（買い切り）。
- 権限付与の正本は署名検証済み Webhook のみ。完了画面から権限付与しない。
- `payment_method_types` はコードで固定せず、Stripe Dashboard の Payment methods 設定を正とする
  （card で開始、PayPay 等は審査承認後に Dashboard で有効化。コード変更不要）。
- Checkout 作成の Request は原則 `{ "productCode": "SUN_AND_MOON" }` のみ。
  - AUTH_USER_ID は requireUser の JWT から取得。
  - Price ID はサーバー側 env（Cloudflare Secret）から取得。
  - metadata に auth_user_id / product_code を設定。
  - success_url = /purchase/success/?productCode=...（URL encode）
  - cancel_url = /purchase/cancel/?productCode=...（URL encode）
- 二重購入防止: Checkout 作成前に available を確認し、既に available なら作成せず拒否（ALREADY_PURCHASED 409）。
  ただし Webhook 側は「作成 API で拒否したから二重購入は来ない」と仮定しない。

## Webhook（確定）
- エンドポイント: POST /api/stripe/webhook（利用者 JWT は要求しない。Stripe 署名検証必須）。
- raw request body を変更せず使用する（body は 1 回だけ取得し署名検証に使う）。
- Stripe 公式 SDK の constructEventAsync + createSubtleCryptoProvider（Workers 用 Web Crypto）で署名検証。
  timestamp tolerance 等は SDK 標準に従い、独自に無効化しない。
- 署名なし・不正署名は INVALID_SIGNATURE（400）。DB は変更しない。
- 対象イベント: checkout.session.completed。
  - checkout.session.completed かつ payment_status = paid の場合のみ購入成立として権限付与。
  - async_payment_succeeded / async_payment_failed / expired は将来の遅延決済のための拡張ポイントとして
    受理のみ（現状は即時決済前提）。返金系（charge.refunded 等）は WORK-007 では扱わない。
- metadata 不足は内部不整合として権限付与しない（400）。

## 金額・Price 照合（確定）
クライアント値（payload の amount_total 等）を信用せず、署名検証後に Stripe 側の確定値で照合する。
- 署名検証後に Stripe API で Checkout Session を retrieve し、line_items を expand する
  （`expand: ["line_items"]`。price は Line Item 内に含まれるため追加 expand しない）。
- Stripe API 呼び出しは 1 Session につき 1 回。
- 照合条件（WORK-007 初期仕様として固定。割引・Stripe Tax・複数量販売は使わない）:
  - line_items.length === 1
  - line_items[0].quantity === 1（null / undefined / 1 以外は不整合。fallback しない）
  - price.id === 期待 Price ID（サーバー側 env）
  - price.currency === "jpy" かつ session.currency === "jpy"
  - price.unit_amount が number（null でない）
  - session.amount_total === price.unit_amount（quantity=1 のため乗算不要）
- 不整合時は T_PURCHASE INSERT なし・T_USER_PRODUCT 変更なし、内部ログへ理由を記録し、
  Secret や内部 Price 情報を利用者・レスポンスへ返さない（内部不整合として扱う）。
- 将来、割引・Tax・複数量販売を導入する場合は別途仕様変更する。

## DB 反映（確定）
- EXTERNAL_PURCHASE_ID = Checkout Session ID。
- D1 batch で T_PURCHASE INSERT + T_USER_PRODUCT INSERT/UPDATE を一括実行（アトミック）。
  T_USER_PRODUCT は last_insert_rowid() で T_PURCHASE の PURCHASE_ID を引き継ぐ。
- T_PURCHASE: PURCHASE_SOURCE=0（Stripe）/ PAYMENT_STATUS=1（支払済）/ AMOUNT=Stripe 確定値。
- T_USER_PRODUCT: STATUS=1 / GRANT_TYPE=0（購入）/ END_DATE=9999-12-31T23:59:59+09:00（買い切り）。
  1 ユーザー × 1 商品 = 1 レコード（既存があれば UPDATE）。
- 冪等性: 既存の一意制約 UX_T_PURCHASE_EXTERNAL（PURCHASE_SOURCE, EXTERNAL_PURCHASE_ID）で
  Webhook 再送の二重登録を防ぐ。再送は処理済みとして 200 を返す。
- 新テーブル・新カラム・新 SETTING_KEY・新公開 API は追加していない。

## 反映待ち画面（確定）
- /purchase/success/: 「Checkout から戻った＝付与完了」とは扱わない。
  GET /api/purchases/status?productCode=... を数回リトライし、granted=true で利用可能表示。
  最終的に未反映でもエラー断定せず「反映を確認中」の表示。DB は変更しない。
- /purchase/cancel/: 「購入は完了していません」表示。DB は変更しない。

## 秘密情報
Cloudflare Secrets（Local は .dev.vars）で管理。実値を Git・wrangler.toml・API レスポンス・フロント JS・ログへ出さない。
- STRIPE_SECRET_KEY
- STRIPE_WEBHOOK_SECRET
- STRIPE_PRICE_SUN_AND_MOON

## 環境
Production アプリ内で Stripe Sandbox/Test Mode を使用し、公開前に Live Mode へ切り替える。

## 実機確認結果

### Local E2E（確認済み）
- Stripe Sandbox/Test Mode 購入成功
- Checkout → Webhook → D1 購入登録
- T_USER_PRODUCT 権限登録
- Webhook 再送で T_PURCHASE 二重登録なし
- 署名なし Webhook を INVALID_SIGNATURE で拒否

### Production E2E（確認済み）
- Production Worker + Production D1 を使用（Stripe は Sandbox/Test Mode）
- 公開 Worker URL を Stripe Webhook endpoint として登録
- Production サイトから Checkout 成功
- Stripe から公開 Worker へ Webhook 直接配送成功
- T_PURCHASE 登録確認
- T_USER_PRODUCT 登録確認
- Webhook 再送後も T_PURCHASE は 1 件
- 署名なし Webhook を INVALID_SIGNATURE で拒否
- keep_vars=true および ADMIN_AUTH_USER_ID 維持を確認

## 完了条件（達成）
Test Mode 購入成功、購入・権限登録、再送で二重登録なし、不正署名拒否、金額・Price 照合による不整合拒否。
Local E2E・Production E2E ともに達成し、WORK-007 を完了とする。
