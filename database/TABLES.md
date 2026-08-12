# Table Specifications

Status: Approved

## M_USER

プラットフォーム側のユーザーマスタ。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| AUTH_USER_ID | TEXT | NO | Supabase Auth UUID、PK |
| LOGIN_MAIL | TEXT | NO | ログインID、UNIQUE |
| STATUS | INTEGER | NO | ユーザー状態 |
| MAIL_AUTH_DATE | TEXT | YES | メール認証日時 |
| PASSWORD_CHANGE_DATE | TEXT | YES | 最終パスワード変更日時 |
| LAST_LOGIN_DATE | TEXT | YES | 最終ログイン日時 |
| DEL_FLG | INTEGER | NO | 論理削除 |
| CREATE_DATE | TEXT | NO | 作成日時 |
| UPDATE_DATE | TEXT | NO | 更新日時 |

## M_PRODUCT

商品マスタ。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| PRODUCT_ID | INTEGER | NO | PK |
| PRODUCT_CODE | TEXT | NO | API・権限判定用コード、UNIQUE |
| PRODUCT_NAME | TEXT | NO | 表示名称 |
| STATUS | INTEGER | NO | 有効・停止 |
| SORT_NO | INTEGER | NO | 表示順 |
| DEL_FLG | INTEGER | NO | 論理削除 |
| CREATE_DATE | TEXT | NO | 作成日時 |
| UPDATE_DATE | TEXT | NO | 更新日時 |

## M_SYSTEM_SETTING

運用時に変更可能な設定値。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| SETTING_KEY | TEXT | NO | PK |
| SETTING_VALUE | TEXT | NO | 設定値 |
| DESCRIPTION | TEXT | YES | 人向け説明 |
| UPDATE_DATE | TEXT | NO | 更新日時 |

初期候補:

```text
WARNING_SCORE
LOGIN_FAIL_LIMIT
ACCESS_LOG_KEEP_DAYS
WARNING_MAIL_INTERVAL_MIN
DEVICE_CHANGE_SCORE
REGION_CHANGE_SCORE
COUNTRY_CHANGE_SCORE
ACCESS_LOG_INTERVAL_MIN
MANY_DEVICES_LIMIT
MANY_REGIONS_LIMIT
```

MANY_DEVICES_LIMIT（初期値 4）と MANY_REGIONS_LIMIT（初期値 3）は WORK-009 で追加する Warning 判定閾値。実際の登録は実装工程の migration で行う。COUNTRY_CHANGE は「24時間以内に有効な2か国以上」の固定条件のため国数用キーは設けない。LOGIN_FAILURE は既存 LOGIN_FAIL_LIMIT を再利用する。

設定値は文字列で保持し、利用側で型変換する。

### ACCESS_LOG_INTERVAL_MIN

権限確認アクセスログ（ACCESS_TYPE=1）の抑制間隔。

- 意味: 同一条件の権限確認アクセスログを再記録するまでの最小間隔（分）
- 初期値: 60
- 0: 抑制なし（毎回記録する）
- 負数: 無効な設定値として内部設定エラーとして扱う
- 不存在・非整数も内部設定エラーとして扱い、利用者へ内部詳細を返さない

migration 0003 で追加（0001 / 0002 は変更しない）。

## T_ORDER

1回の決済（注文ヘッダ）を保持する（WORK-011）。1回のStripe Checkoutで複数商品をまとめ買いできるようにするため、注文（T_ORDER）と購入明細（T_PURCHASE）を分離する。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| ORDER_ID | INTEGER | NO | PK、自動採番 |
| AUTH_USER_ID | TEXT | NO | 購入者 |
| PURCHASE_SOURCE | INTEGER | NO | 0=Stripe、1=note、2=予備（T_PURCHASEと同義） |
| EXTERNAL_ORDER_ID | TEXT | YES | Stripe Checkout Session ID |
| ORDER_DATE | TEXT | NO | 注文日時 |
| TOTAL_AMOUNT | INTEGER | NO | 注文時点の合計金額（購入時価格を履歴保持） |
| PAYMENT_STATUS | INTEGER | NO | 支払状態（T_PURCHASEと同義） |
| REFUND_DATE | TEXT | YES | 返金日時 |
| DEL_FLG | INTEGER | NO | 論理削除 |
| CREATE_DATE | TEXT | NO | 作成日時 |
| UPDATE_DATE | TEXT | NO | 更新日時 |

`PURCHASE_SOURCE + EXTERNAL_ORDER_ID` は、外部IDがある場合に一意とする（`UX_T_ORDER_EXTERNAL`）。同一Stripe Checkout SessionのWebhook再送による注文二重作成を防ぐ冪等キーとする。

Stripe新方式ではT_ORDER 1行に対しT_PURCHASE N行が対応する。既存note購入・既存Stripe単品はT_ORDERを持たず、T_PURCHASE.ORDER_ID=NULLとする（現状維持）。

## T_PURCHASE

購入事実（購入明細）を保持する。複数商品Checkoutでは1注文に複数明細（商品数分）が対応する。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| PURCHASE_ID | INTEGER | NO | PK、自動採番 |
| AUTH_USER_ID | TEXT | NO | 購入者 |
| PRODUCT_ID | INTEGER | NO | 商品 |
| PURCHASE_SOURCE | INTEGER | NO | Stripe、note、管理者 |
| EXTERNAL_PURCHASE_ID | TEXT | YES | 外部決済・取引ID（Stripe新方式ではNULL。noteはNOTE_TRANSACTION_ID） |
| PURCHASE_DATE | TEXT | NO | 実際の購入日時 |
| AMOUNT | INTEGER | NO | 円単位の実支払額（購入時点の商品金額を履歴保持） |
| PAYMENT_STATUS | INTEGER | NO | 支払状態 |
| REFUND_DATE | TEXT | YES | 返金日時 |
| ORDER_ID | INTEGER | YES | 注文ヘッダ（Stripe新方式で設定。note・既存単品はNULL）（WORK-011） |
| DEL_FLG | INTEGER | NO | 論理削除 |
| CREATE_DATE | TEXT | NO | 作成日時 |
| UPDATE_DATE | TEXT | NO | 更新日時 |

`PURCHASE_SOURCE + EXTERNAL_PURCHASE_ID` は、外部IDがある場合に一意とする（note移行の冪等の正本。変更しない）。

`ORDER_ID + PRODUCT_ID` は、ORDER_IDがある場合に一意とする（`UX_T_PURCHASE_ORDER_PRODUCT`）。同一注文内で同一商品を二重登録しない。LINE_NO等の行番号は持たない。

Stripe新方式のCheckout Session IDはT_PURCHASEではなくT_ORDER.EXTERNAL_ORDER_IDに保持する（1 Session→複数明細のため、Session IDを各明細の一意キーにしない）。

## T_USER_PRODUCT

商品利用権限。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| AUTH_USER_ID | TEXT | NO | PKの一部 |
| PRODUCT_ID | INTEGER | NO | PKの一部 |
| STATUS | INTEGER | NO | 利用状態 |
| START_DATE | TEXT | NO | 利用開始日時 |
| END_DATE | TEXT | NO | 利用終了日時 |
| GRANT_TYPE | INTEGER | NO | 付与区分 |
| PURCHASE_ID | INTEGER | YES | 購入由来の場合に設定 |
| MEMO | TEXT | YES | 人が経緯を見るためのメモ |
| DEL_FLG | INTEGER | NO | 論理削除 |
| CREATE_DATE | TEXT | NO | 作成日時 |
| UPDATE_DATE | TEXT | NO | 更新日時 |

MEMOは判定条件・集計条件に使用しない。

## T_NOTE_PURCHASE

note CSVの取込データと移行状態。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| NOTE_PURCHASE_ID | INTEGER | NO | PK |
| PRODUCT_ID | INTEGER | NO | 対応商品 |
| NOTE_ID | TEXT | YES | note購入者名、ゲストはNULL可 |
| NOTE_TRANSACTION_ID | TEXT | NO | note取引ID、UNIQUE |
| PURCHASE_DATE | TEXT | NO | note購入日時 |
| PURCHASE_AMOUNT | INTEGER | NO | note購入金額 |
| MATCH_STATUS | INTEGER | NO | 移行状態 |
| MATCH_AUTH_USER_ID | TEXT | YES | 移行先ユーザー |
| MATCH_DATE | TEXT | YES | 移行日時 |
| PURCHASE_ID | INTEGER | YES | 作成した正式購入ID |
| DEL_FLG | INTEGER | NO | 論理削除 |
| CREATE_DATE | TEXT | NO | 作成日時 |
| UPDATE_DATE | TEXT | NO | 更新日時 |

## T_LOGIN_LOG

ログイン操作履歴。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| LOGIN_LOG_ID | INTEGER | NO | PK |
| AUTH_USER_ID | TEXT | YES | 失敗時に不明な場合あり |
| LOGIN_DATE | TEXT | NO | 操作日時 |
| LOGIN_RESULT | INTEGER | NO | 失敗、成功、ログアウト |
| IP_ADDRESS | TEXT | YES | 接続元IP |
| COUNTRY_CODE | TEXT | YES | 国 |
| REGION | TEXT | YES | 地域 |
| CITY | TEXT | YES | 市区相当 |
| DEVICE_ID | TEXT | YES | ブラウザ発行UUID |
| USER_AGENT | TEXT | YES | User-Agent |
| OS_NAME | TEXT | YES | OS |
| BROWSER_NAME | TEXT | YES | ブラウザ |
| FAILURE_REASON | TEXT | YES | 失敗理由 |
| CREATE_DATE | TEXT | NO | 作成日時 |

## T_ACCESS_LOG

アプリ利用履歴。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| ACCESS_LOG_ID | INTEGER | NO | PK |
| AUTH_USER_ID | TEXT | NO | 利用者 |
| PRODUCT_ID | INTEGER | NO | 対象商品 |
| ACCESS_DATE | TEXT | NO | 利用日時 |
| ACCESS_TYPE | INTEGER | NO | 起動、権限確認、定期確認 |
| IP_ADDRESS | TEXT | YES | 接続元IP |
| COUNTRY_CODE | TEXT | YES | 国 |
| REGION | TEXT | YES | 地域 |
| CITY | TEXT | YES | 市区相当 |
| DEVICE_ID | TEXT | YES | ブラウザ発行UUID |
| USER_AGENT | TEXT | YES | User-Agent |
| OS_NAME | TEXT | YES | OS |
| BROWSER_NAME | TEXT | YES | ブラウザ |
| SESSION_ID_HASH | TEXT | YES | セッション識別用ハッシュ |
| CREATE_DATE | TEXT | NO | 作成日時 |

全API呼出しは保存しない。

アプリ起動、権限確認、一定時間ごとの利用確認に絞る。

## T_WARNING

要注意事象と対応状態。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| WARNING_ID | INTEGER | NO | PK |
| AUTH_USER_ID | TEXT | NO | 対象ユーザー |
| PRODUCT_ID | INTEGER | YES | 商品単位でない場合はNULL |
| WARNING_TYPE | TEXT | NO | 検知種類 |
| WARNING_SCORE | INTEGER | NO | 検知時スコア |
| DETECT_DATE | TEXT | NO | 検知日時 |
| PERIOD_START | TEXT | YES | 判定対象期間開始 |
| PERIOD_END | TEXT | YES | 判定対象期間終了 |
| NOTIFIED_DATE | TEXT | YES | 管理者メール送信日時 |
| STATUS | INTEGER | NO | 未対応、確認済等 |
| LAST_ACTION_DATE | TEXT | YES | 最終対応日時 |
| MEMO | TEXT | YES | 管理者確認メモ |
| CREATE_DATE | TEXT | NO | 作成日時 |
| UPDATE_DATE | TEXT | NO | 更新日時 |

同じ対象・種類・期間の重複登録を避けるため、実装側で検知キーを作る。

## T_CHECKOUT_ATTEMPT

支払い試行ヘッダ（WORK-011 / migration 0006）。1 行 = 1 回の Checkout 試行 ≒ 1 Stripe Session。Stripe Checkout を Platform 側で「支払い試行」として追跡し、二重課金の事前防止（DB 制約による並行 Checkout 排他）・HTTP 再送収束・paid 未反映の救済・refund/dispute 追跡の土台とする。Stripe 専用（note 移行は通らない）。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| ATTEMPT_ID | INTEGER | NO | PK、自動採番 |
| OPERATION_ID | TEXT | NO | ブラウザ生成の安定キー。HTTP 再送収束・idempotency の基（`UX_ATTEMPT_OPERATION` で一意） |
| AUTH_USER_ID | TEXT | NO | 検証済み sub（購入者） |
| CART_KEY | TEXT | NO | サーバー正規化した商品構成の安定表現（同一試行判定） |
| BUYER_EMAIL | TEXT | NO | 試行開始時の認証 email スナップショット（retry でも不変） |
| STATUS | INTEGER | NO | 0=CREATING、1=OPEN、2=PAID、3=EXPIRED、4=CANCELLED |
| CREATE_ATTEMPTED | INTEGER | NO | Stripe create 呼び出し直前に 1 を確定（0+SID=NULL=未試行 / 1+SID=NULL=結果不明） |
| STRIPE_SESSION_ID | TEXT | YES | Stripe 作成後にセット（作成前 NULL）。`UX_ATTEMPT_SESSION` で一意（NULL 対象外） |
| TOTAL_AMOUNT | INTEGER | NO | 期待合計（表示・照合補助。金額正本は Webhook 再取得値） |
| EXPIRES_AT | TEXT | YES | Stripe Session の expires_at（JST ISO） |
| DEL_FLG | INTEGER | NO | 論理削除 |
| CREATE_DATE | TEXT | NO | 作成日時 |
| UPDATE_DATE | TEXT | NO | 更新日時 |

同一 `OPERATION_ID` の HTTP 再送は同一 attempt 行へ収束する（idempotency の DB 保証）。`STATUS` は CREATING→OPEN→PAID/EXPIRED/CANCELLED の一方向遷移。active（CREATING/OPEN）を `IX_ATTEMPT_USER_STATUS` で探索する。

## T_CHECKOUT_ATTEMPT_ITEM

試行で確定した immutable な商品/Price スナップショット（WORK-011 / migration 0006）。Stripe create を同一パラメータで再実行するために必要な値を固定保持する。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| ATTEMPT_ITEM_ID | INTEGER | NO | PK、自動採番 |
| ATTEMPT_ID | INTEGER | NO | T_CHECKOUT_ATTEMPT への参照 |
| PRODUCT_ID | INTEGER | NO | 付与対象（内部 ID） |
| PRODUCT_CODE | TEXT | NO | 監査・CART_KEY・metadata 用 |
| STRIPE_PRICE_ID | TEXT | NO | create 再実行で使う price（試行時点で固定） |
| EXPECTED_AMOUNT | INTEGER | NO | 将来の監査用予約列（試行時点の期待額）。現行フローでは未使用・0 を許容。金額正本は Stripe 側 |
| SORT_NO | INTEGER | NO | line_items の安定順序（create 再現の決定性） |
| CREATE_DATE | TEXT | NO | 作成日時 |

同一試行内での同一商品の重複は `UX_ATTEMPT_ITEM_PRODUCT (ATTEMPT_ID, PRODUCT_ID)` で防止する。

## T_PRODUCT_CHECKOUT_LOCK

active な同一商品購入の競合防止（WORK-011 / migration 0006）。行の存在＝active 予約。`PRIMARY KEY (AUTH_USER_ID, PRODUCT_ID)` が並行 Checkout 排他の正本。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| AUTH_USER_ID | TEXT | NO | 購入者。PK 構成列 |
| PRODUCT_ID | INTEGER | NO | 商品。PK 構成列 |
| ATTEMPT_ID | INTEGER | NO | どの試行が押さえているか（解放・stale 掃除用） |
| CREATE_DATE | TEXT | NO | 作成日時 |

同一ユーザー・同一商品の active 予約は最大 1（PK 制約）。attempt の完了/expire/cancel 時に `IX_LOCK_ATTEMPT` で試行単位に解放する。

## T_PAYMENT_EVENT

決済運用イベント記録（WORK-011 / migration 0006）。duplicate paid / refund / dispute / fulfill failure / reconcile / server indeterminate を記録する。既存 4 種の不正利用 warning（T_WARNING）とは意味が異なるため分離する。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| PAYMENT_EVENT_ID | INTEGER | NO | PK、自動採番 |
| EVENT_TYPE | INTEGER | NO | 1=DUPLICATE_PAID、2=REFUND、3=DISPUTE、4=FULFILL_FAILURE、5=RECONCILE、6=SERVER_INDETERMINATE |
| AUTH_USER_ID | TEXT | YES | 対象ユーザー（あれば） |
| ORDER_ID | INTEGER | YES | 該当注文（あれば） |
| STRIPE_SESSION_ID | TEXT | YES | Checkout Session ID |
| PAYMENT_INTENT_ID | TEXT | YES | PaymentIntent ID |
| STRIPE_OBJECT_ID | TEXT | YES | refund id / dispute id 等 |
| STRIPE_EVENT_ID | TEXT | YES | Webhook event.id（冪等キー。`UX_PAYEVENT_STRIPE_EVENT` で一意、NULL 対象外） |
| STRIPE_REQUEST_ID | TEXT | YES | 調査用（可能なら保存） |
| STATUS | TEXT | YES | refund/dispute の status 文字列 |
| AMOUNT | INTEGER | YES | 金額 |
| DETAIL | TEXT | YES | reason 等（機微情報・カード情報は入れない） |
| NOTIFIED_DATE | TEXT | YES | 管理者通知済み時刻 |
| DEL_FLG | INTEGER | NO | 論理削除 |
| CREATE_DATE | TEXT | NO | 作成日時 |
| UPDATE_DATE | TEXT | NO | 更新日時 |

refund/dispute の Webhook 再送は `STRIPE_EVENT_ID` の一意制約で二重記録しない。注文・種別で `IX_PAYEVENT_ORDER` / `IX_PAYEVENT_TYPE_DATE` により追跡する。

## T_ORDER への追加列（migration 0006）

`PAYMENT_INTENT_ID`（TEXT / NULL 可）を追加した。Refund/Charge の payment_intent から `T_ORDER.PAYMENT_INTENT_ID`（`IX_T_ORDER_PI`）で注文を逆引きする。
