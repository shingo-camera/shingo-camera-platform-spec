# Product and Entitlement API

Status: Approved

## 1. GET /api/products

### 目的

公開商品一覧またはログイン後の商品一覧を返す。

### 認証

任意。

### 対象

```text
M_PRODUCT.STATUS = 1
M_PRODUCT.DEL_FLG = 0
```

### Response

```json
{
  "result": "OK",
  "data": {
    "products": [
      {
        "code": "SUN_AND_MOON",
        "name": "SUN AND MOON PLANNER",
        "sortNo": 10
      }
    ]
  }
}
```

販売価格やStripe Price IDは、決済設計の実装時にCloudflare Secretsまたは商品設定として管理する。

秘密キーをM_PRODUCTへ保存しない。

## 2. GET /api/products/{code}

### 目的

商品詳細画面用。

### Path

```text
code = PRODUCT_CODE
```

### Error

存在しない、停止、論理削除の場合:

```text
PRODUCT_NOT_FOUND
```

## 3. GET /api/entitlements/{code}

### 目的

アプリ起動直前に商品利用可否を判定する。

### 認証

必須。

### 処理

1. JWT検証
2. M_USER状態確認
3. M_PRODUCT確認
4. T_USER_PRODUCT確認
5. 日付範囲確認
6. T_ACCESS_LOGへ権限確認を記録
7. 結果返却

### Response: 利用可

```json
{
  "result": "OK",
  "data": {
    "productCode": "SUN_AND_MOON",
    "available": true,
    "startAt": "2026-08-05T12:00:00+09:00",
    "endAt": "9999-12-31T23:59:59+09:00"
  }
}
```

### Response: 権限なし

HTTP 403

```json
{
  "result": "NG",
  "error": {
    "code": "PRODUCT_NOT_GRANTED",
    "message": "この商品は利用できません。"
  }
}
```

## 4. requireProduct

各アプリ固有APIから共通関数として呼び出す。

例:

```text
requireProduct(request, env, "SUN_AND_MOON")
```

アプリ画面で判定済みでも、計算API側でも必ず権限を確認する。

画面の表示制御だけをセキュリティ境界にしない。

## 5. アクセスログ抑制

同一ユーザー、同一商品、同一DEVICE_ID、同一ACCESS_TYPEで短時間に連続する場合は、毎回保存しない。

初期目安:

```text
1時間に最大1件
```

この間隔はM_SYSTEM_SETTINGで変更可能とする。
