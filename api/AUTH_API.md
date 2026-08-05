# Account and Authentication API

Status: Approved

## 1. 前提

新規登録、ログイン、ログアウト、パスワード再設定そのものはSupabase Authクライアントを使用する。

Cloudflare APIは以下を担当する。

- JWT検証
- M_USER同期
- ユーザー状態確認
- 商品権限返却
- ログ記録
- 業務日時更新

## 2. GET /api/account/me

### 目的

ログイン中ユーザーの業務情報を返す。

### 認証

必須。

### 処理

1. JWT検証
2. AUTH_USER_ID取得
3. M_USER検索
4. M_USERがない場合は同期処理を行う
5. STATUS確認
6. 商品権限を取得
7. レスポンス返却

### Response

```json
{
  "result": "OK",
  "data": {
    "email": "user@example.com",
    "status": 1,
    "mailAuthenticatedAt": "2026-08-05T10:00:00+09:00",
    "passwordChangedAt": null,
    "lastLoginAt": "2026-08-05T17:00:00+09:00",
    "products": [
      {
        "code": "SUN_AND_MOON",
        "name": "SUN AND MOON PLANNER",
        "status": 1,
        "available": true,
        "startAt": "2026-08-05T12:00:00+09:00",
        "endAt": "9999-12-31T23:59:59+09:00"
      }
    ]
  }
}
```

## 3. POST /api/account/sync

### 目的

Supabase Authの認証ユーザーをM_USERへ同期する。

### Request

本文なし。

### 正本

- AUTH_USER_ID: 検証済みJWT
- LOGIN_MAIL: 検証済みJWTまたはSupabase Authのユーザー情報

ブラウザから送られたメールアドレスを正本にしない。

### 処理

M_USERがない場合:

```text
INSERT
STATUS = メール認証済みなら1、未認証なら0
```

存在する場合:

```text
LOGIN_MAIL
MAIL_AUTH_DATE
LAST_LOGIN_DATE
UPDATE_DATE
```

を同期する。

### 冪等性

AUTH_USER_IDを主キーとしてUPSERTする。

## 4. GET /api/account/products

### 目的

ログイン後ホームで保有商品を表示する。

### Response

```json
{
  "result": "OK",
  "data": {
    "products": [
      {
        "code": "SUN_AND_MOON",
        "name": "SUN AND MOON PLANNER",
        "granted": true,
        "available": true,
        "status": 1
      },
      {
        "code": "HANABI",
        "name": "HANABI PLANNER",
        "granted": false,
        "available": false,
        "status": null
      }
    ]
  }
}
```

M_PRODUCTの有効商品を基準に全商品を返し、T_USER_PRODUCTをLEFT JOINする。

## 5. POST /api/account/password-changed

### 目的

Supabase Authでのパスワード変更成功後、M_USER.PASSWORD_CHANGE_DATEを更新する。

### Request

本文なし。

### 注意

Supabase Authで実際に変更が成功した後に呼ぶ。

このAPIだけでパスワード変更済みとは判断しない。

## 6. ログインログ

Supabase Authのログイン結果に応じ、クライアントまたは共通APIを経由してT_LOGIN_LOGへ記録する。

失敗ログにパスワードを記録しない。

ユーザーが特定できない失敗時はAUTH_USER_IDをNULLとする。

## 7. 停止ユーザー

M_USER.STATUSが2または9の場合:

```http
403 Forbidden
```

```json
{
  "result": "NG",
  "error": {
    "code": "USER_SUSPENDED",
    "message": "このアカウントは現在利用できません。"
  }
}
```

停止理由の詳細は利用者へ返さない。
