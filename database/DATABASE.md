# Database Design

Status: Approved

## 1. 目的

Cloudflare D1を使用し、shingo-camera Platformの業務データを管理する。

認証情報の正本はSupabase Authとし、D1には商品、購入、商品権限、note移行、利用ログ、Warning、運用設定を保持する。

## 2. 採用DB

- Cloudflare D1
- SQLite互換
- Cloudflare Workers / Pages Functionsから利用
- 既存のCloudflare環境内で完結
- 想定利用者100〜300人、最大1,000人程度

本規模では、別途PostgreSQLや大規模ログ基盤を導入しない。

## 3. テーブル一覧

| テーブル | 役割 |
|---|---|
| M_USER | プラットフォーム利用者 |
| M_PRODUCT | 商品マスタ |
| M_SYSTEM_SETTING | 運用設定・Warning閾値 |
| T_PURCHASE | 購入履歴 |
| T_USER_PRODUCT | ユーザーの商品利用権限 |
| T_NOTE_PURCHASE | note既存購入履歴と移行状態 |
| T_LOGIN_LOG | ログイン操作履歴 |
| T_ACCESS_LOG | アプリ利用履歴 |
| T_WARNING | 要注意利用の検知・通知・対応状態 |

## 4. 設計原則

### 4.1 AUTH_USER_IDを共通キーにする

Supabase Authが発行するUUIDを使用する。

自前の連番USER_IDは持たない。

### 4.2 商品はレコードで増やす

M_USERへ商品別フラグ列を追加しない。

商品追加はM_PRODUCTへの1レコード追加で対応する。

### 4.3 1ユーザー×1商品＝1権限レコード

T_USER_PRODUCTの主キーは以下とする。

```text
AUTH_USER_ID + PRODUCT_ID
```

商品を再付与する場合は、新規INSERTではなく既存レコードを更新・復活させる。

### 4.4 購入と権限を分離する

T_PURCHASEは購入事実を保持する。

T_USER_PRODUCTは利用可能状態を保持する。

二重購入が発生した場合、T_PURCHASEは複数件、T_USER_PRODUCTは1件となる。

### 4.5 パスワードを保存しない

パスワード本体およびハッシュはSupabase Authのみで管理する。

### 4.6 日時形式

D1にはISO 8601形式のTEXTで保存する。

例:

```text
2026-08-05T17:30:00+09:00
```

アプリ内では日本時間を基本表示とする。

Cloudflare側の実装では、保存時のタイムゾーン方針を共通関数に統一する。

### 4.7 論理削除

主要マスタ・トランザクションはDEL_FLGを持つ。

通常運用では物理削除しない。

ログは運用上不要になった場合、CSV出力後に物理削除可能とする。

## 5. コード値

### M_USER.STATUS

| 値 | 内容 |
|---:|---|
| 0 | 仮登録 |
| 1 | 有効 |
| 2 | 一時停止 |
| 9 | 退会 |

### M_PRODUCT.STATUS

| 値 | 内容 |
|---:|---|
| 0 | 停止 |
| 1 | 有効 |

### T_USER_PRODUCT.STATUS

| 値 | 内容 |
|---:|---|
| 0 | 利用開始前 |
| 1 | 有効 |
| 2 | 一時停止 |
| 9 | 失効 |

### T_USER_PRODUCT.GRANT_TYPE

| 値 | 内容 |
|---:|---|
| 0 | 購入 |
| 1 | note移行 |
| 2 | テスター |
| 3 | 管理者付与 |
| 4 | 補償 |

### T_PURCHASE.PURCHASE_SOURCE

| 値 | 内容 |
|---:|---|
| 0 | Stripe |
| 1 | note |
| 2 | 管理者登録 |

### T_PURCHASE.PAYMENT_STATUS

| 値 | 内容 |
|---:|---|
| 0 | 処理中 |
| 1 | 支払済 |
| 2 | 返金済 |
| 3 | 取消 |
| 9 | エラー |

### T_NOTE_PURCHASE.MATCH_STATUS

| 値 | 内容 |
|---:|---|
| 0 | 未移行 |
| 1 | 移行済 |
| 2 | 要確認 |
| 9 | 無効 |

### T_LOGIN_LOG.LOGIN_RESULT

| 値 | 内容 |
|---:|---|
| 0 | 失敗 |
| 1 | 成功 |
| 2 | ログアウト |

### T_ACCESS_LOG.ACCESS_TYPE

| 値 | 内容 |
|---:|---|
| 0 | アプリ起動 |
| 1 | 権限確認 |
| 2 | 定期利用確認 |

### T_WARNING.STATUS

| 値 | 内容 |
|---:|---|
| 0 | 未対応 |
| 1 | 確認済 |
| 2 | ユーザー確認中 |
| 9 | 除外 |

## 6. 初期商品

| PRODUCT_ID | PRODUCT_CODE | PRODUCT_NAME |
|---:|---|---|
| 1 | SUN_AND_MOON | SUN AND MOON PLANNER |
| 2 | HANABI | HANABI PLANNER |
| 3 | HANABI_GOOGLE_EARTH | HANABI PLANNER Google Earth追加機能 |

## 7. 買い切り商品の期間

通常値:

```text
START_DATE = 購入日時または権限付与日時
END_DATE   = 9999-12-31T23:59:59+09:00
```

note移行時は、移行日ではなくnoteでの購入日時をSTART_DATEへ設定する。

MATCH_DATEはプラットフォームへ移行した日時を保持する。

## 8. note CSV取込

取込対象:

- 有料記事の販売レコード
- HANABI
- HANABI Google Earth追加機能

除外:

- チップ

利用するCSV項目:

- 決済/返金日時
- 購入者名
- コンテンツ名
- 販売額
- 取引ID

取引IDをNOTE_TRANSACTION_IDとして保持する。

購入者名はNOTE_IDとして保持し、問い合わせやSNSアカウントとの照合補助に使用する。

NOTE_IDは一意キーや認証キーには使用しない。

## 9. Warning

Warningの元データはT_LOGIN_LOGおよびT_ACCESS_LOGとする。

定期処理で条件を判定し、通知対象となった場合のみT_WARNINGを登録する。

T_WARNINGを保持する理由:

- 同一事象の重複メール防止
- 未対応・確認済みの管理
- 管理画面から対象ログへ追跡
- 牽制のための一時停止判断

自動停止は行わない。

## 10. ログ保持

初期はログローテーションを自動化しない。

件数が増えた場合:

1. CSV出力
2. 必要ならバックアップテーブルへ退避
3. 古いログをDELETE

保存期間はM_SYSTEM_SETTINGで調整可能とする。
