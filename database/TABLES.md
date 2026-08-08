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
```

設定値は文字列で保持し、利用側で型変換する。

### ACCESS_LOG_INTERVAL_MIN

権限確認アクセスログ（ACCESS_TYPE=1）の抑制間隔。

- 意味: 同一条件の権限確認アクセスログを再記録するまでの最小間隔（分）
- 初期値: 60
- 0: 抑制なし（毎回記録する）
- 負数: 無効な設定値として内部設定エラーとして扱う
- 不存在・非整数も内部設定エラーとして扱い、利用者へ内部詳細を返さない

migration 0003 で追加（0001 / 0002 は変更しない）。

## T_PURCHASE

購入事実を保持する。

| カラム | 型 | NULL | 説明 |
|---|---|---|---|
| PURCHASE_ID | INTEGER | NO | PK、自動採番 |
| AUTH_USER_ID | TEXT | NO | 購入者 |
| PRODUCT_ID | INTEGER | NO | 商品 |
| PURCHASE_SOURCE | INTEGER | NO | Stripe、note、管理者 |
| EXTERNAL_PURCHASE_ID | TEXT | YES | 外部決済・取引ID |
| PURCHASE_DATE | TEXT | NO | 実際の購入日時 |
| AMOUNT | INTEGER | NO | 円単位の実支払額 |
| PAYMENT_STATUS | INTEGER | NO | 支払状態 |
| REFUND_DATE | TEXT | YES | 返金日時 |
| DEL_FLG | INTEGER | NO | 論理削除 |
| CREATE_DATE | TEXT | NO | 作成日時 |
| UPDATE_DATE | TEXT | NO | 更新日時 |

`PURCHASE_SOURCE + EXTERNAL_PURCHASE_ID` は、外部IDがある場合に一意とする。

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
