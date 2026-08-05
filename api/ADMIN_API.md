# Admin API

Status: Approved

## 1. 方針

管理画面はPC専用。

通常運用を管理画面だけで完結させ、SQL直接更新を避ける。

管理者は初期1人。

複雑なロール・承認フローは作らない。

## 2. 管理者認証

共通`requireAdmin()`を使用する。

初期の管理者判定方式は実装時に以下のいずれかで固定する。

推奨:

```text
Cloudflare Secretまたは環境設定に管理者AUTH_USER_IDを登録
```

管理者メールアドレスだけを判定キーにしない。

## 3. GET /api/admin/dashboard

### Response項目

- 総ユーザー数
- 商品別有効ユーザー数
- 当日新規登録数
- 当日購入数
- note未移行件数
- 未対応Warning件数
- 直近Warning一覧

グラフは初期実装しない。

## 4. GET /api/admin/users

### Query

```text
email
status
productCode
limit
offset
```

### 初期値

```text
limit = 100
offset = 0
```

### 一覧項目

- メールアドレス
- ユーザー状態
- 保有商品
- 登録日時
- 最終ログイン日時
- 最終アクセス日時

## 5. GET /api/admin/users/{authUserId}

### 返却内容

- M_USER
- T_USER_PRODUCT
- T_PURCHASE
- T_NOTE_PURCHASE
- T_LOGIN_LOG
- T_ACCESS_LOG
- T_WARNING

ログは新しい順。

初期取得件数を制限し、追加読込を可能にする。

## 6. PUT /api/admin/users/{authUserId}/status

### Request

```json
{
  "status": 2
}
```

許可:

```text
1: 有効
2: 一時停止
9: 退会
```

退会は初期画面に出さず、必要時のみ使用してもよい。

### 操作

- 停止
- 再開

停止・再開は確認ダイアログ後、ボタン1回で完了する。

## 7. PUT /api/admin/users/{authUserId}/products/{productCode}

### Request例: 付与・再開

```json
{
  "status": 1,
  "grantType": 3,
  "startAt": "2026-08-05T18:00:00+09:00",
  "endAt": "9999-12-31T23:59:59+09:00",
  "memo": "管理者手動付与"
}
```

### Request例: 停止

```json
{
  "status": 2
}
```

同一ユーザー・同一商品の既存レコードを更新する。

新規権限の場合のみINSERTする。

## 8. GET /api/admin/warnings

### Query

```text
status
warningType
authUserId
from
to
limit
offset
```

### 一覧項目

- 対象メール
- 商品
- Warning種別
- スコア
- 検知日時
- 通知日時
- 対応状態

## 9. PUT /api/admin/warnings/{warningId}

### Request

```json
{
  "status": 1,
  "memo": "本人利用を確認"
}
```

ユーザー停止はこのAPIで自動実行しない。

必要なら別途ユーザー状態APIを実行する。

## 10. POST /api/admin/note/import

### 形式

```text
multipart/form-data
```

CSVファイルと取込モードを受け取る。

### 処理結果

```json
{
  "result": "OK",
  "data": {
    "read": 151,
    "imported": 141,
    "ignoredTips": 10,
    "duplicates": 0,
    "errors": []
  }
}
```

### 冪等性

NOTE_TRANSACTION_IDの一意制約により再取込を防ぐ。

同一行は重複として数え、エラー終了しない。

## 11. GET /api/admin/note/purchases

### Query

```text
noteId
transactionId
productCode
matchStatus
limit
offset
```

## 12. PUT /api/admin/note/purchases/{id}

用途:

- 手動紐付け
- 紐付け解除
- 要確認化
- 無効化

誤紐付け解除時は、T_PURCHASEとT_USER_PRODUCTへの影響を画面に表示し、確認後に実行する。

初期は複雑な自動巻戻しを作らず、対象データを明示した管理APIを用意する。
