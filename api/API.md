# API Design

Status: Approved

## 1. 目的

ブラウザ、管理画面、各アプリからD1を直接操作せず、Cloudflare Workers / Pages FunctionsのAPIを通じて利用する。

画面はAPIだけを参照し、DB構造の変更を画面へ直接波及させない。

## 2. 基本URL

```text
/api
```

## 3. 認証

利用者APIは以下のヘッダーを必須とする。

```http
Authorization: Bearer <SUPABASE_ACCESS_TOKEN>
X-Device-Id: <UUID>
```

Cloudflare APIは受信したJWTをサーバー側で検証し、検証済みの`sub`をAUTH_USER_IDとして使用する。

リクエスト本文・クエリパラメータからAUTH_USER_IDを受け取って権限判定しない。

管理者APIは、認証済みユーザーが管理者として許可されていることを共通処理で確認する。

## 4. 共通レスポンス

### 成功

```json
{
  "result": "OK",
  "data": {}
}
```

### 業務エラー

```json
{
  "result": "NG",
  "error": {
    "code": "PRODUCT_NOT_GRANTED",
    "message": "この商品は利用できません。"
  }
}
```

### 入力エラー

```json
{
  "result": "NG",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容を確認してください。",
    "fields": {
      "email": "メールアドレスの形式が正しくありません。"
    }
  }
}
```

内部例外、SQL、外部サービスの詳細、ユーザー存在有無などを利用者向けmessageへ含めない。

## 5. HTTPステータス

| Status | 用途 |
|---:|---|
| 200 | 正常 |
| 201 | 登録成功 |
| 400 | 入力不正 |
| 401 | 未認証・トークン不正 |
| 403 | 権限なし・停止中 |
| 404 | 対象なし |
| 409 | 重複・処理済み |
| 429 | 試行回数超過 |
| 500 | 内部エラー |
| 502 | 外部サービス連携エラー |

## 6. 日時と金額

- 日時はISO 8601文字列
- 金額は円単位の整数
- API内部の現在日時生成は共通関数を使用
- 表示形式への変換はフロント側共通関数を使用

## 7. 入力検証

全APIで以下を行う。

- Content-Type確認
- JSON構文確認
- 必須項目確認
- 型確認
- 文字数確認
- コード値確認
- UUID形式確認
- 日時形式確認
- 想定外項目は原則無視せずエラーまたは明示的に除外

SQLは必ずPrepared Statementとbindを使用する。

## 8. 更新処理

購入、note移行、権限付与など複数テーブルを同時更新する処理はD1のbatchを使用する。

1ステートメントでも失敗した場合は一連の更新を成立させない。

## 9. 冪等性

以下は重複実行されても二重登録しない。

- Stripe Webhook
- note移行
- M_USER同期
- 商品権限付与
- Warning定期処理

外部ID、複合主キー、処理済み状態を用いて判定する。

## 10. ログ

利用者APIの全呼出しはT_ACCESS_LOGへ保存しない。

以下に限定する。

- アプリ起動
- 商品権限確認
- 一定時間ごとの利用確認
- ログイン成功・失敗・ログアウト

内部エラーはCloudflareの標準ログへ出力する。

秘密情報、パスワード、完全なJWTはログへ出力しない。

## 11. API一覧

### Account

| Method | Path | 用途 |
|---|---|---|
| GET | `/api/account/me` | 自分のアカウント情報 |
| GET | `/api/account/products` | 自分の商品権限 |
| POST | `/api/account/sync` | M_USER同期 |
| POST | `/api/account/password-changed` | パスワード変更日時更新 |

### Products

| Method | Path | 用途 |
|---|---|---|
| GET | `/api/products` | 販売・表示可能商品一覧 |
| GET | `/api/products/{code}` | 商品詳細 |
| GET | `/api/entitlements/{code}` | 商品利用可否確認 |

### Purchase

| Method | Path | 用途 |
|---|---|---|
| POST | `/api/purchases/checkout` | Stripe Checkout開始 |
| POST | `/api/stripe/webhook` | Stripe Webhook受信 |
| GET | `/api/purchases/status` | 購入反映状況確認 |

### note移行

| Method | Path | 用途 |
|---|---|---|
| POST | `/api/migrations/note/apply` | 購入者による移行 |
| GET | `/api/migrations/note/status` | 自分の移行状態 |

### 管理者

| Method | Path | 用途 |
|---|---|---|
| GET | `/api/admin/dashboard` | 管理ダッシュボード |
| GET | `/api/admin/users` | ユーザー一覧 |
| GET | `/api/admin/users/{authUserId}` | ユーザー詳細 |
| PUT | `/api/admin/users/{authUserId}/status` | ユーザー停止・再開 |
| PUT | `/api/admin/users/{authUserId}/products/{productCode}` | 商品権限更新 |
| GET | `/api/admin/warnings` | Warning一覧 |
| PUT | `/api/admin/warnings/{warningId}` | Warning対応状態更新 |
| POST | `/api/admin/note/import` | note CSV取込 |
| GET | `/api/admin/note/purchases` | note購入一覧 |
| PUT | `/api/admin/note/purchases/{id}` | note移行の手動修正 |

## 12. 共通サーバー関数

実装側で以下を共通化する。

```text
requireUser()
requireAdmin()
requireProduct()
getDeviceId()
jsonOk()
jsonError()
validateJson()
nowIso()
writeLoginLog()
writeAccessLog()
getSystemSetting()
```
